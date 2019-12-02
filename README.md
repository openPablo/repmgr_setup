 ===Repmgr Nota's (CentOS 7 PG 11)===

Note: these are raw notes

```
curl https://dl.2ndquadrant.com/default/release/get/11/rpm | sudo bash
yum repolist
yum install repmgr11
```


vi ~/.bashrc:

```
PATH="$PATH:/usr/pgsql-11/bin/"
```



```
firewall-cmd --permanent --zone=public --add-rich-rule=' rule family="ipv4"source address="192.168.56.0/24" port protocol="tcp" port="5432" accept'
firewall-cmd --reload
```

in postgresql.conf:

```
  max_wal_senders = 10
  wal_level = 'hot_standby'
  hot_standby = on
  archive_mode = on
  archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
  listen_adndress = '*'
  shared_preload_libraries = 'repmgr'
  wal_keep_segmentes = 64
```

mkdir voor archive command

in pg_hba.conf:

```
local   replication   repmgr                              trust
host    replication   repmgr      127.0.0.1/32            trust
host    replication   repmgr      192.168.56.0/24          trust
local   repmgr        repmgr                              trust
host    repmgr        repmgr      127.0.0.1/32            trust
host    repmgr        repmgr      192.168.56.0/24          trust
```

connect to database:

```
create user repmgr with login superuser createdb replication,;
\c postgres repmgr
create database repmgr;
ALTER USER repmgr SET search_path TO repmgr_test, "$user", public;
```

/etc/repmgr/11/repmgr.conf:

```
node_id=1
node_name='srv1'
reconnect_attempts=5
reconnect_interval=1
conninfo='host=srv1 user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/pgsql/11/data'
promote_command='repmgr standby promote -f /etc/repmgr/11/repmgr.conf'
follow_command='repmgr standby follow -f /etc/repmgr/11/repmgr.conf'
pg_bindir='/usr/pgsql-11/bin/
failover=automatic
use_replication_slots = 1
```

Configure SSH in all nodes
login as postgres:

```
ssh-keygen
ssh-copy-id postgres@srv
firewall-cmd --zone=public --add-port=22/tcp --permanent
firewall-cmd --reload
restorecon -R -v /var/lib/pgsql/.ssh
```

Register primary:

```
repmgr -f /etc/repmgr.conf primary register
```

Copy to standby:

```
repmgr -h MASTER_IP -U repmgr -d repmgr -f /etc/repmgr/11/repmgr.conf standby clone --dry-run
repmgr -f /etc/repmgr/11/repmgr.conf standby register
```

On all servers:

```
systemctl enable repmgr11
systemctl start repmgr11
systemctl enable postgresql-11
systemctl start postgresql-11
```

===RECOVER===

```
sudo su postgres
rm -rf /var/lib/pgsql/11/data/*
repmgr -h srv2 -U repmgr -d repmgr  -f /etc/repmgr/11/repmgr.conf standby clone
exit
systemctl start postgresql-11
sudo su postgres
repmgr -f /etc/repmgr/11/repmgr.conf standby register -F
```


