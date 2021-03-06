---
layout: post
title: Programming Oracle
tags: [Oracle, Database]
color: rgb(250, 154, 133)
feature-img: "assets/img/post-cover/36-cover.png"
thumbnail: "assets/img/post-cover/36-cover.png"
author: QubitPi
excerpt_separator: <!--more-->
---

<!--more-->

* TOC
{:toc}

## References

* [Oracle Tutorial](https://www.oracletutorial.com/)

## Shell

### Run Oracle SQL script and Exit Afterwards

    exit | sqlplus -S $user/$pwd@//$server:$port/$service_name @script.sql

One example value of `$service_name` could be "ORCL". Note that there is a `@` symble preceding the script
path(`script.sql`)

### Connecting to Oracle DB Instance From the Command Line

#### (First Time) Install Oracle Instant Client and `sqlplus` using Homebrew

1. Download the two files below from
   http://www.oracle.com/technetwork/topics/intel-macsoft-096467.html. This is necessary because of Oracle licenses:
     - instantclient-basic-macos.x64–<version>.zip
     - instantclient-sqlplus-macos.x64–<version>.zip
2. 

        brew tap InstantClientTap/instantclient
        brew install instantclient-basic
        brew install instantclient-sqlplus

#### Connect

    sqlplus ${DBUSER}/${DBUSERPASSWORD}@//${HOST}:${PORT}/${SERVICE_NAME}
    
where `SERVICE_NAME` is the same thing as "database name" as in MySQL terminology
    
### Shell Commands

#### Show Tables

If you have worked with MySQL, you may be familiar with the `SHOW TABLES` command that lists all tables in a database:

```sql
SHOW TABLES;
```

Unfortunately, Oracle **does not** directly support the `SHOW TABLES` command. However, you can list all tables in a
database by querying from various data dictionary views.

##### Show Tables Owned By the Current User

To show tables **owned** by the current user, you query from the `user_tables` view.

```sql
SELECT table_name
FROM user_tables
ORDER BY table_name;
```

Note that this view does not show the `OWNER` column. Also, the user_tables table **does not** contain the other tables
that are **accessible** by the current user.

##### Show Tables Accessible By the Current User

To show all tables that are currently **accessible** by the current user, regardless of owners, you query from the
`all_tables` view:

```sql
SELECT table_name
FROM all_tables
ORDER BY table_name;
```

If you want to show all tables of a specific owner, you add the `OWNER` column in the `WHERE` clause as shown in the
following query:

```sql
SELECT *
FROM all_tables
WHERE OWNER = 'OT'
ORDER BY table_name;
```

##### Show All Tables in the Oracle Database

To show **all tables** in the entire Oracle Database, you query from the `dba_tables` view as follows:

```sql
SELECT table_name 
FROM dba_tables;
```

You will get the following error message if you don't have access to the `dba_tables` view:

```
ORA-00942: table or view does not exist
```

In this case, you should request your database administrator to grant your account either privileges on the `dba_tables`
view, or `SELECT ANY DICTIONARY` privilege, or `SELECT_CATALOG_ROLE` privilege.

#### [Comparing Dates](https://stackoverflow.com/a/34061999)

```
Select count(*) From Employee 
Where to_char(employee_date_hired, 'YYYMMMDDD') > 19940620 
```

or

```
Select count(*) From Employee 
employee_date_hired > TO_DATE('20-06-1994', 'DD-MM-YYYY');
```

#### [Show Last Command Run](https://stackoverflow.com/a/51193086)

The 'l' command will show the last run command

```
SQL> l
    1* select owner, count(1) from dba_tables group by owner
SQL>
```

To get more than that, turn on history

```
SQL> set history on
SQL> history
  1  select * from dual;
  2  select sysdate from dual;
  3  show history
```

#### [Get A List of All Tables](https://stackoverflow.com/a/205746)

```
SELECT OWNER, TABLE_NAME
FROM DBA_TABLES;
```

This is assuming that you have access to the `DBA_TABLES` data dictionary view. If you do not have those privileges but
need them, you can request that the DBA explicitly grants you privileges on that table, or, that the DBA grants you the
`SELECT ANY DICTIONARY` privilege or the `SELECT_CATALOG_ROLE` role (either of which would allow you to query any data
dictionary table). Of course, you may want to exclude certain schemas like `SYS` and `SYSTEM` which have large numbers
of Oracle tables that you probably don't care about.

Alternatively, if you do not have access to `DBA_TABLES`, you can see all the tables that your account has access to
through the `ALL_TABLES` view:

```
SELECT OWNER, TABLE_NAME
FROM ALL_TABLES;
```

Although, that may be a subset of the tables available in the database (`ALL_TABLES` shows you the information for all
the tables that your user has been granted access to).

If you are only concerned with the tables that you own, not those that you have access to, you could use `USER_TABLES`:

```
SELECT TABLE_NAME
FROM USER_TABLES;
```

Since `USER_TABLES` only has information about the tables that you own, it does not have an `OWNER` column - the owner,
by definition, is you.

Oracle also has a number of legacy data dictionary views -- `TAB`, `DICT`, `TABS`, and `CAT` for example -- that could
be used. In general, I would not suggest using these legacy views unless you absolutely need to backport your scripts to
Oracle 6. Oracle has not changed these views in a long time so they often have problems with newer types of objects. For
example, the `TAB` and `CAT` views both show information about tables that are in the user's recycle bin while the
`[DBA|ALL|USER]_TABLES` views all filter those out. `CAT` also shows information about materialized view logs with a
`TABLE_TYPE` of "TABLE" which is unlikely to be what you really want. `DICT` combines tables and synonyms and doesn't
tell you who owns the object.

## Scripting

### WHENEVER SQLERROR

When you have a script that has syntax errors, the script won't exit with error while executing the line that has the
syntax error. We would like to, instead, halt the execution at that point for the script tuning purposes. To do that
`WHENEVER SQLERROR` could be used.

The commands in the following script cause SQL*Plus to exit and return the SQL error code if the SQL UPDATE command
fails:

```sql
WHENEVER SQLERROR EXIT SQL.SQLCODE
UPDATE EMP_DETAILS_VIEW SET SALARY = SALARY*1.1;
```


### Drop All User Tables

```
BEGIN
   FOR cur_rec IN (SELECT object_name, object_type
                   FROM user_objects
                   WHERE object_type IN
                             ('TABLE',
                              'VIEW',
                              'MATERIALIZED VIEW',
                              'PACKAGE',
                              'PROCEDURE',
                              'FUNCTION',
                              'SEQUENCE',
                              'SYNONYM',
                              'PACKAGE BODY'
                             ))
   LOOP
      BEGIN
         IF cur_rec.object_type = 'TABLE'
         THEN
            EXECUTE IMMEDIATE 'DROP '
                              || cur_rec.object_type
                              || ' "'
                              || cur_rec.object_name
                              || '" CASCADE CONSTRAINTS';
         ELSE
            EXECUTE IMMEDIATE 'DROP '
                              || cur_rec.object_type
                              || ' "'
                              || cur_rec.object_name
                              || '"';
         END IF;
      EXCEPTION
         WHEN OTHERS
         THEN
            DBMS_OUTPUT.put_line ('FAILED: DROP '
                                  || cur_rec.object_type
                                  || ' "'
                                  || cur_rec.object_name
                                  || '"'
                                 );
      END;
   END LOOP;
   FOR cur_rec IN (SELECT * 
                   FROM all_synonyms 
                   WHERE table_owner IN (SELECT USER FROM dual))
   LOOP
      BEGIN
         EXECUTE IMMEDIATE 'DROP PUBLIC SYNONYM ' || cur_rec.synonym_name;
      END;
   END LOOP;
END;
/
```
