## 1. Network Setup
**We'll be working on the 'postgres' network. Let's start by creating it:**

```
docker network create postgres 
docker network ls  # Verify the network was created. 
```

Let's Keep Refining!

## 2. Create Primary Node

```
docker run -it --rm --name primary `
--net postgres `
-e POSTGRES_PASSWORD=root `
-e PGDATA="/data" `
-v ${PWD}/primary:/data `
-p 5432:5432 `
postgres
```

### 2.1 Create Backup Folder on Primary
Explanation: In this step, you'll create a backup of your primary PostgreSQL database. This backup is essential for setting up replication.
```
docker exec -it primary bash 
mkdir backup
pg_basebackup -D /backup/ -R -v -U postgres
```
**_*Expected Output:*_**

```
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 0/2000060 on timeline 1
pg_basebackup: starting background WAL receiver
pg_basebackup: created temporary replication slot "pg_basebackup_78"
pg_basebackup: write-ahead log end point: 0/2000138
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: syncing data to disk ...
pg_basebackup: renaming backup_manifest.tmp to backup_manifest
pg_basebackup: base backup completed
```

Copy Backup to Data Directory

```
cp -r ./backup/ /data 
```

### 2.2 Copy all of items inside backup folder from primary to standby

copy `./primary/backup/*` folder to `./standby/` on Local machine
_For this instance, I utilized `PowerShell` to explain._

```
cp -r .\standby\backup\* .\standby\

cd ..\standby\

ls
```

Expected Output:

```
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         3/23/2024  10:10 PM                base
d-----         3/23/2024  10:10 PM                global
d-----         3/23/2024  10:10 PM                pg_commit_ts
d-----         3/23/2024  10:10 PM                pg_dynshmem
d-----         3/23/2024  10:10 PM                pg_logical
d-----         3/23/2024  10:10 PM                pg_multixact
d-----         3/23/2024  10:10 PM                pg_notify
d-----         3/23/2024  10:10 PM                pg_replslot
d-----         3/23/2024  10:10 PM                pg_serial
d-----         3/23/2024  10:10 PM                pg_snapshots
d-----         3/23/2024  10:10 PM                pg_stat
d-----         3/23/2024  10:10 PM                pg_stat_tmp
d-----         3/23/2024  10:10 PM                pg_subtrans
d-----         3/23/2024  10:10 PM                pg_tblspc
d-----         3/23/2024  10:10 PM                pg_twophase
d-----         3/23/2024  10:10 PM                pg_wal
d-----         3/23/2024  10:10 PM                pg_xact
-a----         3/23/2024   9:49 PM            225 backup_label
-a----         3/23/2024   9:49 PM         137324 backup_manifest
-a----         3/23/2024   9:49 PM           5743 pg_hba.conf
-a----         3/23/2024   9:49 PM           2640 pg_ident.conf
-a----         3/23/2024   9:49 PM              3 PG_VERSION
-a----         3/23/2024   9:49 PM            381 postgresql.auto.conf
-a----         3/23/2024   9:49 PM          29770 postgresql.conf
-a----         3/23/2024   9:49 PM              0 standby.signal
```

#### come back to primary folder

`cd ./primary/`

Edit `postgresql.conf` file

```
archive_mode = on

archive_command = 'test ! -f /data/archiver/%f && cp %p /data/archiver/%f'

wal_keep_size = 512
```

Edit `pg_hba.conf` file

```
host    replication replication_user    (ip address from docker replica)    trust
```

### 1.4 Create Replication users

On primary server

```
docker exec -it primary bash

psql -U postgres

CREATE USER replication_user replication
```

command should return `CREATE ROLE`

Check inserted replication user.

```
# on psql mode.
\dt
```

command should return

```
                                 List of roles
    Role name     |                         Attributes
------------------+------------------------------------------------------------
 postgres         | Superuser, Create role, Create DB, Replication, Bypass RLS
 replication_user | Replication
```

## 1.5 create Achive folder

```
docker exec -it primary bash

cd data

mkdir archiver
```

## 2. Create standby node

```
docker run -it --rm --name standby `
--net postgres `
-e POSTGRES_PASSWORD=root `
-e PGDATA="/data" `
-v ${PWD}/standby:/data `
-p 5433:5432 `
postgres
```

### 2.1 Configuration

`cd ./standby/`

check ip address of primary node
`docker inspect primary`

Edit `postgresql.conf` file
docker should return `ip address`. Give the ip address of the primary node to use on the next.

```
# enter the ip address from above to config.
primary_conninfo = 'host=<ip address from primary> port=<port> user=replication_user'
```

Edit `postgresql.auto.conf` file

```
primary_conninfo = 'user=replication_user port=<port> host=<ip address from primary>'
```

## 3. Test Replication

Go to primary node

```
docker exec -it primary

psql -U postgres

CREATE DATABASE test_db
# returm CREATE DATABASE

# list of database
\l
```

command should return

```
                                                      List of databases
   Name    |  Owner   | Encoding | Locale Provider |  Collate   |   Ctype    | ICU Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+------------+------------+------------+-----------+-----------------------
 postgres  | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           |
 template0 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           | =c/postgres          +
           |          |          |                 |            |            |            |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           | =c/postgres          +
           |          |          |                 |            |            |            |           | postgres=CTc/postgres
 test_db   | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           |
(4 rows)
```

```
CREATE TABLE users (id int, name varchar(100));
# return CREATE TABLE

INSERT INTO users (id, name) VALUES (1, 'a'), (2, 'b');
# return INSERT (0, 2)

SELECT * FROM users;

 id | name
----+------
  1 | a
  2 | B
(2 rows)

```

Go to standby node

```
docker exec -it primary
psql -U postgres
\l
                                                      List of databases
   Name    |  Owner   | Encoding | Locale Provider |  Collate   |   Ctype    | ICU Locale | ICU Rules |   Access privileges
-----------+----------+----------+-----------------+------------+------------+------------+-----------+-----------------------
 postgres  | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           |
 template0 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           | =c/postgres          +
           |          |          |                 |            |            |            |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           | =c/postgres          +
           |          |          |                 |            |            |            |           | postgres=CTc/postgres
 test_db   | postgres | UTF8     | libc            | en_US.utf8 | en_US.utf8 |            |           |
(4 rows)
q

SELECT * FROM users;
 id | name
----+------
  1 | a
  2 | B
(2 rows)


 INSERT INTO users (id, name) VALUES (2, 'c');
# return ERROR:  cannot execute INSERT in a read-only transaction
```

The standby user should be read-only permissions.

If you want to add a new replication.
You just to copy a backup file from primary and setup config same a standby server again.

## 4. How to connect cluster to `HAPROXY`

The first step is a pull haproxy image.

```
docker pull haproxy
```

We are still using `postgres` network.

### 4.1 Configuration `haproxy.cfg`

create a new `haproxy.cfg` file.
In this scenario. I'll generate `./haproxy/haproxy.cfg` in the same directory as primary and standby.

Set up a configuration in the file `haproxy.cfg`.

```
global
    log stdout format raw local0

defaults
    log global
    option redispatch
    retries 3
    timeout connect 5s
    timeout client 30m
    timeout server 30m

listen postgresql-cluster
    bind *:5000
    mode tcp
    balance roundrobin
    server primary <ip_address>:<internal_port 5432> check
    server standby <ip_address>:<internal_port 5432> check
```

### 4.2 Build & Run container

run `HAPROXY` on docker.

```
docker run -d --name haproxy --rm --net postgres -p 5000:5000 -v ${PWD}/haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
```

### 4.3 Test load balance

For testing, we connect to the haproxy on port 5000 via `DBeaver`.

Configure the Connection Setting from this.

```
Hsot: localhost
Port: 5000
Authentication: Database Native
Username: postgres
Password: root
```

If it can be connected. It should display a message box like this.

```
Connected (148 ms)
Server: PostgreSQL 16.2 (Debian 16.2-1.pgdg120+2)
        PostgreSQL 16.2 (Debian 16.2-1.pgdg120+2) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit
Driver: PostgreSQL JDBC Driver 42.7.2
```
