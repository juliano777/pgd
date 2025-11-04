# The lab environment


| **Hostname** | **IP address** |
|--------------|----------------|
| pgd0         | 192.168.56.70  |
| pgd1         | 192.168.56.71  |
| pgd2         | 192.168.56.72  |


## bla bla bla

```bash
cat << EOF > /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

#=============================================================================
192.168.56.70  pgd0  pgd0.my.domain
192.168.56.71  pgd1  pgd1.my.domain
192.168.56.72  pgd2  pgd2.my.domain

EOF
```

# PostgreSQL installation

```bash
dnf install -y lsb_release

read -p 'Enter the PostgreSQL major version: ' PGMAJOR

DISTRO_VERSION=`lsb_release -r | awk '{print $2}' | cut -f1 -d.`

URL="https://download.postgresql.org/pub/repos/yum/reporpms/\
EL-${DISTRO_VERSION}-`arch`/pgdg-redhat-repo-latest.noarch.rpm"

PGDATA="/var/lib/pgsql/${PGMAJOR}/data"

PGLOG="/var/log/pgsql/${PGMAJOR}"

mkdir -p ${PGLOG} && chown -R postgres /var/log/pgsql

dnf install -y ${URL}

dnf -qy module disable postgresql

dnf install -y postgresql${PGMAJOR}-server

/usr/pgsql-${PGMAJOR}/bin/postgresql-${PGMAJOR}-setup initdb


# listen_addresses = '*'
sed "s:\(^#listen_addresses.*\):\1\nlisten_addresses = '*':g" \
-i ${PGDATA}/postgresql.conf

# log_destination = 'stderr'
sed "s:\(^#log_destination.*\):\1\nlog_destination = 'stderr':g" \
-i ${PGDATA}/postgresql.conf

# logging_collector = on
sed "s:\(^#logging_collector.*\):\1\nlogging_collector = on:g" \
-i ${PGDATA}/postgresql.conf

# log_filename (nova linha descomentada)
sed "s:\(^#\)\(log_filename.*\):\1\2\n\2:g" \
-i ${PGDATA}/postgresql.conf

# log_directory = '${PGLOG}'
sed "s:\(^#log_directory.*\):\1\nlog_directory = '${PGLOG}':g" \
-i ${PGDATA}/postgresql.conf


cat << EOF > ${PGDATA}/pg_hba.conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
# IPv6 local connections:
host    all             all             ::1/128                 trust

host    all             all             192.168.56.0/24         trust

# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
host    replication     all             ::1/128                 scram-sha-256
EOF


cat << EOF > ~postgres/.pgvars
# Environment Variables ======================================================

# PostgreSQL major version
PGMAJOR="${PGMAJOR}"

# Binary directory
PGBIN="/usr/pgsql-\${PGMAJOR}/bin"

# Path to binaries
export PATH="\${PGBIN}:\${PATH}"

# PostgreSQL data directory
export PGDATA="/var/lib/pgsql/\${PGMAJOR}/data"vim 
# Unset variables
unset PGMAJOR PGBIN

EOF


echo -e "\nsource ~/.pgvars" >> ~postgres/.bash_profile



systemctl enable --now postgresql-${PGMAJOR}
```


# PGD installation


Run the following statement and paste your token

```bash
read -sp 'Insert your EDB token: ' TOKEN

# Variable for URL
URL="https://downloads.enterprisedb.com/${TOKEN}/enterprise/setup.rpm.sh"

curl -1sSLf ${URL} | bash
```

```bash
dnf install -y edb-pgd6-essential-pg${PGMAJOR} 

dnf clean all
```


## pagila database

```bash
su - postgres

cd /tmp/

wget https://ftp.postgresql.org/pub/projects/pgFoundry/dbsamples/pagila/\
pagila/pagila-0.10.1.zip

unzip pagila-0.10.1.zip

createdb db_pagila

psql -d db_pagila -f pagila-0.10.1/pagila-schema.sql

psql -d db_pagila -f pagila-0.10.1/pagila-data.sql
```


# PGD configuration

track_commit_timestamp = on
shared_preload_libraries = 'bdr'


pgd node pgd0 setup --dsn "host=pgd0 user=postgres port=5432 dbname=db_pagila" --group-name mygroup00


