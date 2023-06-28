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

## REF

- https://university.scylladb.com/courses/scylla-essentials-overview/lessons/quick-wins-install-and-run-scylla/topic/install-and-start-scylladb/
