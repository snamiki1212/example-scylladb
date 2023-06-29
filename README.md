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

## Shard

```zsh
docker exec -it scylla-node3 bash
./usr/lib/scylla/seastar-cpu-map.sh -n scylla # check shard
```

## DC

```zsh
git clone https://github.com/scylladb/scylla-code-samples.git
cd scylla-code-samples/mms
docker-compose up -d # wake up DC1
sleep 60; say ok; # wait 1m
docker-compose -f docker-compose-dc2.yml up -d # wake up DC2
sleep 60; say ok; # wait 1m
docker exec -it scylla-node1 nodetool status # check all UN status or wait/retry
```

```zsh
docker exec -it scylla-node2 nodetool status
docker exec -it scylla-node2 cqlsh

# CQL
CREATE KEYSPACE scyllaU WITH REPLICATION = {'class' : 'NetworkTopologyStrategy', 'DC1' : 3, 'DC2' : 2};
Use scyllaU;
DESCRIBE KEYSPACE
```

```zsh
# CQL
consistency EACH_QUORUM;

CREATE TABLE users ( user_id int, fname text, lname text, PRIMARY KEY((user_id)));
insert into users(user_id, fname, lname) values (1, 'rick', 'sanchez');
insert into users(user_id, fname, lname) values (4, 'rust', 'cohle');
# => Success
```

```zsh
docker-compose -f docker-compose-dc2.yml pause
docker exec -it scylla-node2 nodetool status # 3 DN in DC2

# CQL
insert into users(user_id, fname, lname) values (8, 'lorne', 'malvo');
# => NoHostAvailable:
```

```zsh
docker-compose -f docker-compose-dc2.yml unpause
sleep 60; say ok; # wait 1m
docker exec -it scylla-node2 nodetool status # check if all UN
# CQL
insert into users(user_id, fname, lname) values (8, 'lorne', 'malvo'); # Success
```

```zsh
# CQL
select * from users; # InvalidRequest:...
consistency LOCAL_QUORUM;
select * from users;
```

## Token

```zsh
# Show Token info
docker exec -it scylla-node1 nodetool ring
# or
docker exec -it scylla-node1 nodetool describering scyllau
```

## Gossip

```zsh
# check
docker exec -it scylla-node1 nodetool statusgossip
# => running

# more info
docker exec -it scylla-node1 nodetool gossipinfo
```

## Snitch

```zsh
docker exec -it scylla-node1 nodetool describecluster
docker exec -it  scylla-node1 cat /etc/scylla/cassandra-rackdc.properties
```

## Tools

- Nodetool
  - https://opensource.docs.scylladb.com/stable/operating-scylla/nodetool-commands/status

## REF

- https://university.scylladb.com/courses/scylla-essentials-overview/lessons/quick-wins-install-and-run-scylla/topic/install-and-start-scylladb/
