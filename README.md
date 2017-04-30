# Master-Master Replication PostreSQL-9.4 for FreeSwitch Data_Base 



Good afternoon colleague, I want to share my experience in the implementation of the PostgreSQL replication configuration of the master <-> master type, for replicating CDR data, this solution was used to replicate the CDR database of FreeSwitch


# Initial data
Bucardo is a replication system for Postgres that supports any number of sources and targets (aka masters and slaves). It is asynchronous and trigger based.


We are have two servers  

pg_node1  (we are also install FreeSwitch)

pg_node2  


Operating system on servers is 


CentOS 7

PostgreSQL 9.4 Database

PG_Node_1  192.168.66.128

PG_Node_2  192.168.66.130


# Installation

We'll start step by step by first configuring packages on two servers for stable server operation


we are disable SELINUX
```bash
sed -i 's/\(^SELINUX=\).*/\SELINUX=disabled/' /etc/sysconfig/selinux
sed -i 's/\(^SELINUX=\).*/\SELINUX=disabled/' /etc/selinux/config
```
To apply the changes, we need to reboot the server


Next, install the necessary packages

```bash
yum groupinstall 'Development Tools'
yum  update
```



# Install PostreSQL-9.4

These settings are done on two servers Data Base

After that, as we prepared the system and installed all the necessary packages
Proceed to install the PostreSQL-9.4 database


```bash
rpm -Uvh http://yum.postgresql.org/9.4/redhat/rhel-7-x86_64/pgdg-centos94-9.4-2.noarch.rpm
yum install postgresql94 postgresql94-server postgresql94-contrib
su - postgres -c /usr/pgsql-9.4/bin/initdb
service postgresql-9.4 start
chkconfig --levels 235 postgresql-9.4 on
```


After instalation create Data base CDR 

```bash
sudo -u postgres psql
postgres=# alter user postgres with password 'postgres';
postgres=# CREATE DATABASE freeswitchdb; 
postgres=# \c freeswitchdb;
CREATE TABLE cdr(
 ID SERIAL PRIMARY KEY,
 local_ip_v4 CHAR(16), 
 caller_id_name CHAR(20), 
 caller_id_number CHAR(20),
 outbound_caller_id_number CHAR(20),
 destination_number CHAR(20),
 context CHAR(20),
 start_stamp timestamp default NULL,
 answer_stamp timestamp default NULL,
 end_stamp timestamp default NULL,
 duration INT, 
 billsec INT, 
 hangup_cause CHAR(30),
 uuid CHAR(50),
 bleg_uuid CHAR(50),
 accountcode CHAR(20),
 read_codec CHAR(10),
 write_codec CHAR(10)
);
```


After you have created the Databases on two servers, we will start configuring replication

# Install Bucardo

While all changes are made on two database servers

We will install the necessary packages on each server that will participate in the replication:

```bash
yum install  postgresql postgresql-plperl-9.4 bucardo
```
Create a directory in which to store the PID of the running bucardu server:


```bash
mkdir -p /var/run/bucardo
```

```bash
/var/lib/pgsql/9.4/data/postgresql.conf
```

Change to 

```bash
listen_addresses = ‘*’
```
Then you need to specify which hosts and how to work with the database, first the first host:

PG_Node_1


```bash
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 trust
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                trust
#host    replication     postgres        127.0.0.1/32            trust
#host    replication     postgres        ::1/128                 trust

host    all             bucardo         127.0.0.1/32              md5
host    all             bucardo         192.168.66.130/32         md5
```


PG_Node_2
```bash
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 trust
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     postgres                                trust
#host    replication     postgres        127.0.0.1/32            trust
#host    replication     postgres        ::1/128                 trust

host    all             all             192.168.66.128/32           md5
host    all             bucardo         127.0.0.1/32                md5
host    all             bucardo         192.168.66.128/32           md5
```

Then reboot the server   

```bash
service postgresql-9.4 restart
```

# Installing Bucardu in PostgreSQL

On each of the servers:

Here we can select the host on which we want to install this replication system.

```bash
 bucardo install
 
 
 Changed database name to: freeswitchdb
Current connection settings:
1. Host:           127.0.0.1
2. Port:           5432
3. User:           postgres
4. Database:       freeswitchdb
5. PID directory:  /var/run/bucardo
Enter a number to change it, P to proceed, or Q to quit: p

```


Set up a password

```bash
sudo -u postgres psql
ALTER USER bucardo WITH PASSWORD 'postgres';
```


```bash
 nano /root/.pgpass
 
 Add:
 localhost:5432:*:bucardo:postgres
 
 after 
 
 bucardo start
```



# Setting up Bucardo


Then we work only with the first server
I will remind you that we work only with 1M server. Let's add bases to bucardo:

```bash
bucardo add database freeswitchdb dbname=freeswitchdb dbhost=127.0.0.1 dbuser=bucardo dbpass=postgres
bucardo add database freeswitchdb2 dbname=freeswitchdb dbhost=192.168.33.130 dbuser=bucardo dbpass=postgres
```

Add all available tables on the freeswitchd server to the freeswitchd_herb table group:

```bash
[root@freeswitch bin]# bucardo add table all —db=freeswitchdb —herd=freeswitchdb_herd
Creating relgroup: freeswitchdb_herd
Added table public.cdr to relgroup freeswitchdb_herd
New tables added: 1
```

Add all the sequences:

```bash
[root@freeswitch bin]# bucardo add sequence all —db=freeswitchdb —herd=freeswitchdb_herd
Added sequence public.cdr_id_seq to relgroup freeswitchdb_herd
New sequences added: 1
```

Create a server group:


```bash
bucardo add dbgroup freeswitchdb_servers
bucardo add dbgroup freeswitchdb_servers freeswitchdb:source
bucardo add dbgroup freeswitchdb_servers freeswitchdb2:source
```


Now create the synchronization:

```bash
[root@freeswitch bin]# bucardo add sync  freeswitchdb_sync herd=freeswitchdb_herd dbs=freeswitchdb_servers
Added sync "freeswitchdb_sync"
```

And Check

```bash
bucardo list all
```

After changing the settings, you must restart the Bucardo:


```bash
bucardo restart
``` 

and also we are check the status


```bash
[root@freeswitch bin]# bucardo status
PID of Bucardo MCP: 74450
 Name                State    Last good    Time    Last I/D      
===================+========+============+=======+===========+
 freeswitchdb_sync | Good   | 06:04:59   | 38s   | 0/0       | 
``` 

# Testing

We can now check
example queries


PG_Node_1

```bash
psql -U postgres freeswitchdb -c «INSERT INTO cdr VALUES (1, ‘a’);
psql -U postgres freeswitchdb -c «INSERT INTO cdr VALUES (2, ‘b’);
``` 


PG_Node_2

```bash
psql -U postgres freeswitchdb -c «SELECT * FROM cdr;
psql -U postgres freeswitchdb -c «INSERT INTO cdr VALUES (3, ‘c’);
``` 

And again PG_Node_1

```bash
psql -U postgres freeswitchdb -c «SELECT * FROM cdr;
``` 

As you can see, replication works both ways.

Or we can make test calls that are written to the database since the module mod_cdr_pg_cdv whitch we are setup in FreeSwitch writes values to the database
Example of module configuration

```bash
<param name="db-info" value="host=127.0.0.1 dbname=freeswitchdb user=postgres password=postgres connect_timeout=10" />
``` 
After 


```bash
fs_cli -x 'load mod_cdr_pg_csv'
``` 

Now the CDR will be written to the database.



