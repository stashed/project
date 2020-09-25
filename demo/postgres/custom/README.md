# PostgreSQL Demo

## Deploy PostgreSQL

- Create PostgreSQL

```bash
kubectl apply -f ./postgres.yaml
```

**Insert sample data:**

- Exec into the database pod

```bash
$ kubectl exec -it -n demo sample-postgres-7fc6c6559b-5xbgp /bin/sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.

```

- Connect to database

```bash
$ psql -U postgres
psql (12.4 (Debian 12.4-1.pgdg100+1))
Type "help" for help.
```

- List available databases

```bash
postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(3 rows)
```

- Create a new database

```bash
postgres=# CREATE DATABASE records;
CREATE DATABASE
```

- List available databases

```bash
postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 records   | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(4 rows)
```

- Connected to the newly created database

```bash
postgres=# \c records
You are now connected to database "records" as user "postgres".
```

- Create table

```bash
records=# CREATE TABLE company( NAME TEXT NOT NULL, EMPLOYEE INT NOT NULL);
CREATE TABLE
```

- List tables

```bash
records=# \d
          List of relations
 Schema |  Name   | Type  |  Owner   
--------+---------+-------+----------
 public | company | table | postgres
(1 row)
```

- Insert data into the table

```bash
records=# INSERT INTO company VALUES ('appscode' , 100);
INSERT 0 1
```

- Show data from the table

```bash
records=# SELECT * FROM company;
   name   | employee 
----------+----------
 appscode |      100

```

## Backup

- Create AppBinding

```bash
$ kubectl apply -f ./appbinding.yaml
```

- Create Secret

```bash
$ kubectl create secret generic -n demo s3-secret \
    --from-file=./RESTIC_PASSWORD \
    --from-file=./AWS_ACCESS_KEY_ID \
    --from-file=./AWS_SECRET_ACCESS_KEY
```

- Create Repository

```bash
kubectl apply -f ./repository.yaml
```

- Create BackupConfiguration

```bash
kubectl apply -f ./backupconfiguration.yaml
```

- Verify CronJob

```bash
$ kubectl get cronjob -n demo
NAME                                  SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
stash-backup-sample-postgres-backup   */5 * * * *   False     0        3m44s           5m26s
```

- Trigger instant backup

```bash
kubectl apply -f ./backupsession.yaml
```

- Verify backup has succeeded

```bash
$ kubectl get backupsession -n demo
NAME                                INVOKER-TYPE          INVOKER-NAME             PHASE       AGE
sample-postgres-backup-1600412177   BackupConfiguration   sample-postgres-backup   Succeeded   4m24s
```

## Restore

- Pause Backup

```bash
$ kubectl patch backupconfiguration -n demo sample-postgres-backup --type="merge" --patch='{"spec": {"paused": true}}'
```

- Verify CronJob paused

```bash
$ kubectl get cronjob -n demo
NAME                                  SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
stash-backup-sample-postgres-backup   */5 * * * *   True      0        4m33s           11m
```

- Simulate disaster (delete some data)

```bash
kubectl exec -it -n demo sample-postgres-7fc6c6559b-5xbgp -- /bin/sh
# psql -U postgres
psql (12.4 (Debian 12.4-1.pgdg100+1))
Type "help" for help.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 records   | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(4 rows)

postgres=# \c records
You are now connected to database "records" as user "postgres".
records=# \d
          List of relations
 Schema |  Name   | Type  |  Owner   
--------+---------+-------+----------
 public | company | table | postgres
(1 row)

records=# DROP TABLE company;
DROP TABLE
records=# \d
Did not find any relations.
records=# \q
# exit
```

- Create RestoreSession

```bash
$ kubectl apply -f ./restoresession.yaml
restoresession.stash.appscode.com/sample-postgres-restore created
```

- Verify Restore succeeded

```bash
$ kubectl get restoresession -n demo
NAME                      REPOSITORY   PHASE       AGE
sample-postgres-restore   s3-repo      Succeeded   65s
```

- Verify restored data

```bash
kubectl exec -it -n demo sample-postgres-7fc6c6559b-5xbgp -- /bin/sh
# psql -U postgres
psql (12.4 (Debian 12.4-1.pgdg100+1))
Type "help" for help.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 records   | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(4 rows)

postgres=# \c records
You are now connected to database "records" as user "postgres".

records=# \d
          List of relations
 Schema |  Name   | Type  |  Owner   
--------+---------+-------+----------
 public | company | table | postgres
(1 row)

records=# SELECT * FROM company;
   name   | employee 
----------+----------
 appscode |      100
(1 rows)

records=# \q
# exit
```
