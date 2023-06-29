## Installation

```zsh
# Run docker
docker run --name scyllaU -d scylladb/scylla:4.5.0 --overprovisioned 1 --smp 1

# Execute to verify cluster is up by Nodetool
docker exec -it scyllaU nodetool status
# => UN status. “U” means up, and N means normal.

# Start CQL
docker exec -it scyllaU cqlsh
```

## Basic

```cql
CREATE KEYSPACE mykeyspace WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 1};

use mykeyspace;

CREATE TABLE users ( user_id int, fname text, lname text, PRIMARY KEY((user_id)));


insert into users(user_id, fname, lname) values (1, 'rick', 'sanchez');
insert into users(user_id, fname, lname) values (4, 'rust', 'cohle');

select * from users;
-- => 2 rows
```

## Cluster: Case1

```zsh
docker run --name Node_X -d scylladb/scylla:4.5.0 --overprovisioned 1 --smp 1
docker run --name Node_Y -d scylladb/scylla:4.5.0 --seeds="$(docker inspect --format='{{ .NetworkSettings.IPAddress }}' Node_X)" --overprovisioned 1 --smp 1
docker run --name Node_Z -d scylladb/scylla:4.5.0 --seeds="$(docker inspect --format='{{ .NetworkSettings.IPAddress }}' Node_X)" --overprovisioned 1 --smp 1

# wait about 1 min
docker exec -it Node_Z nodetool status
# => expect to be all UN

docker exec -it Node_Z cqlsh
```

```cql
-- Seet RF=3
CREATE KEYSPACE mykeyspace WITH REPLICATION = { 'class' : 'NetworkTopologyStrategy', 'replication_factor' : 3};
use mykeyspace;
CREATE TABLE users ( user_id int, fname text, lname text, PRIMARY KEY((user_id)));
insert into users(user_id, fname, lname) values (1, 'rick', 'sanchez');
insert into users(user_id, fname, lname) values (4, 'rust', 'cohle');
select * from users;
```

```cql
-- Change CL
CONSISTENCY QUORUM
insert into users (user_id, fname, lname) values (7, 'eric', 'cartman');
select * from users;

-- Change CL
CONSISTENCY ALL
insert into users (user_id, fname, lname) values (8, 'lorne', 'malvo');
select * from users;
```

```zsh
# stop Node-Y
docker stop Node_Y

# Enter Node-Z with CQL
docker exec -it Node_Z cqlsh

# cql
# Valid Case
CONSISTENCY QUORUM
use mykeyspace;
insert into users (user_id, fname, lname) values (9, 'avon', 'barksdale');   # => success insert
select * from users;

# Invalid Case
CONSISTENCY ALL
insert into users (user_id, fname, lname) values (10, 'vm', 'varga');
# => NoHostAvailable:
select * from users;
# => NoHostAvailable:
```

```zsh
docker stop Node_Z
docker exec -it Node_X nodetool status  # expect 2 DN, 1 UN
docker exec -it Node_X cqlsh

# cql
# invalid case
cqlsh> CONSISTENCY QUORUM
Consistency level set to QUORUM.
cqlsh> use mykeyspace;
cqlsh:mykeyspace> insert into users (user_id, fname, lname) values (11, 'morty', 'smith');
NoHostAvailable:
cqlsh:mykeyspace> select * from users;
NoHostAvailable:

# valid case
cqlsh:mykeyspace> CONSISTENCY ONE
Consistency level set to ONE.
cqlsh:mykeyspace> insert into users (user_id, fname, lname) values (12, 'marlo', 'stanfield');
cqlsh:mykeyspace> select * from users;
```

## Cluster: Case2

```zsh
docker run --name scylla-node1 -d scylladb/scylla:5.1.0
docker run --name scylla-node2 -d scylladb/scylla:5.1.0 --seeds="$(docker inspect --format='{{ .NetworkSettings.IPAddress }}' scylla-node1)"
docker run --name scylla-node3 -d scylladb/scylla:5.1.0 --seeds="$(docker inspect --format='{{ .NetworkSettings.IPAddress }}' scylla-node1)"

# check 3 UN
docker exec -it scylla-node3 nodetool status

docker exec -it scylla-node3 cqlsh

# CQL
CREATE KEYSPACE mykeyspace WITH REPLICATION = { 'class' : 'NetworkTopologyStrategy', 'replication_factor' : 3};
use mykeyspace;
DESCRIBE KEYSPACE mykeyspace;
CREATE TABLE users ( user_id int, fname text, lname text, PRIMARY KEY((user_id)));

insert into users(user_id, fname, lname) values (1, 'rick', 'sanchez');
insert into users(user_id, fname, lname) values (4, 'rust', 'cohle');
select * from users;
```

## Tools

- Nodetool
  - https://opensource.docs.scylladb.com/stable/operating-scylla/nodetool-commands/status

## REF

- https://university.scylladb.com/courses/scylla-essentials-overview/lessons/quick-wins-install-and-run-scylla/topic/install-and-start-scylladb/
