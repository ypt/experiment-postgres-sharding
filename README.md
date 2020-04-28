# Experiment with Postgres horizontal scaling

Here are the main questions explored:

1. There are various Postgres compatible solutions built with scalibility in mind ([Citus](https://www.citusdata.com/), [AWS Aurora](https://aws.amazon.com/rds/aurora/), [CockroachDB](https://www.cockroachlabs.com/)), but what about plain old Postgres?
2. Are we able to let a Postgres data layer of new services start small and simple and grow in complexity just-in-time if needed?
3. How much of the data layer complexity can be done transparently to application developers?

Here is an example scenario:

1. A new service starts with a small, simple, easy to work with database - a single schema, no partitions, no sharding, using just a column in each table to specify the tenant for multi-tenancy
2. As the service evolves, various schema changes may happen
3. As the service sees more usage, more and more data is inserted
4. At some point, the database has grown large enough such that we'd want to scale it somehow. (Vertical scling is probably the simplest thing we'd reach for. Though, at some point, if the service continues to grow - we'd likely want to look o our horizontal scaling options)
5. We horizontally scale the database
6. Ideally, this change can be transparent to the app. No code changes. Schema changes can continue to happen as the service continues to evolve
7. Repeat from Step 2

## Hands on example
Bring up two Postgres containers - `db_1` and `db_2`
```shell
docker-compose up
```

Log into `db_1`
```shell
psql -h localhost -p 5432 -U experiment
# when prompted, enter the password: experiment
```

### Humble beginnings
First, create a standard table
```SQL
CREATE TABLE exercises (
  id UUID PRIMARY KEY,
  tenant VARCHAR NOT NULL,
  name VARCHAR NOT NULL
);
```

Insert some data
```SQL
INSERT INTO exercises (id, tenant, name) VALUES ('ac8e6a36-722e-4e53-b9e6-e916aded0b00', 'Acme', 'jumping');
INSERT INTO exercises (id, tenant, name) VALUES ('284d1166-0bdc-4c66-93c1-43ddbd0491a0', 'Acme', 'running');
INSERT INTO exercises (id, tenant, name) VALUES ('aa46f49b-5fca-4fc8-bd63-fdf587710c00', 'Bobmart', 'hopping');
INSERT INTO exercises (id, tenant, name) VALUES ('b9257460-173b-45e8-b4c3-7ec269c8f74d', 'Crazytown', 'skipping');
INSERT INTO exercises (id, tenant, name) VALUES ('4f32eadf-0abe-480a-93bc-8780fdf85b5d', 'Dory', 'swimming');
INSERT INTO exercises (id, tenant, name) VALUES ('216b0fd6-5baf-4432-80c6-069243e8cba4', 'Eggbert', 'sitting');
```

### Partitioning a table on the same machine via Postgres Declarative Table Partitioning
Now we'll convert the table to a partitioned table via [Postgres Declarative Table Partitioning].

We'll start with just a _single_ partition on the same server.

```SQL
BEGIN;

ALTER TABLE exercises RENAME TO exercises_old;

CREATE TABLE exercises (LIKE exercises_old) -- TODO: look into various LIKE INCLUDING options
  PARTITION BY RANGE ( tenant );

ALTER TABLE exercises
  ATTACH PARTITION exercises_old DEFAULT;

COMMIT;
```

Let's see what's in each table now.

In the main partitioned table.
```SQL
SELECT * FROM exercises;
                  id                  |  tenant   |   name
--------------------------------------+-----------+----------
 ac8e6a36-722e-4e53-b9e6-e916aded0b00 | Acme      | jumping
 284d1166-0bdc-4c66-93c1-43ddbd0491a0 | Acme      | running
 aa46f49b-5fca-4fc8-bd63-fdf587710c00 | Bobmart   | hopping
 b9257460-173b-45e8-b4c3-7ec269c8f74d | Crazytown | skipping
 4f32eadf-0abe-480a-93bc-8780fdf85b5d | Dory      | swimming
 216b0fd6-5baf-4432-80c6-069243e8cba4 | Eggbert   | sitting
```

In the single partition.
```SQL
SELECT * FROM exercises_old;
                  id                  |  tenant   |   name
--------------------------------------+-----------+----------
 ac8e6a36-722e-4e53-b9e6-e916aded0b00 | Acme      | jumping
 284d1166-0bdc-4c66-93c1-43ddbd0491a0 | Acme      | running
 aa46f49b-5fca-4fc8-bd63-fdf587710c00 | Bobmart   | hopping
 b9257460-173b-45e8-b4c3-7ec269c8f74d | Crazytown | skipping
 4f32eadf-0abe-480a-93bc-8780fdf85b5d | Dory      | swimming
 216b0fd6-5baf-4432-80c6-069243e8cba4 | Eggbert   | sitting
```

Now let's actually start setting up multiple partitions and moving data over to them

```SQL
BEGIN;

CREATE TABLE exercises_0 (
  LIKE exercises_old INCLUDING ALL
);

-- NOTE: these rows will be locked for the duration of the move,
-- so care will need to be taken if this moves a lot of data in practice
WITH x AS (
  DELETE FROM exercises_old WHERE tenant < 'B' returning *
)
INSERT INTO exercises_0
  SELECT * FROM x;

ALTER TABLE exercises
  ATTACH PARTITION exercises_0 FOR VALUES FROM (MINVALUE) TO ('B');

COMMIT;
```

Now our `exercises` table is comprised of two partions - `exercises_old` and `exercises_0`.
```SQL
SELECT * FROM exercises;
                  id                  |  tenant   |   name
--------------------------------------+-----------+----------
 ac8e6a36-722e-4e53-b9e6-e916aded0b00 | Acme      | jumping
 284d1166-0bdc-4c66-93c1-43ddbd0491a0 | Acme      | running
 aa46f49b-5fca-4fc8-bd63-fdf587710c00 | Bobmart   | hopping
 b9257460-173b-45e8-b4c3-7ec269c8f74d | Crazytown | skipping
 4f32eadf-0abe-480a-93bc-8780fdf85b5d | Dory      | swimming
 216b0fd6-5baf-4432-80c6-069243e8cba4 | Eggbert   | sitting

SELECT * FROM exercises_old;
                  id                  |  tenant   |   name
--------------------------------------+-----------+----------
 aa46f49b-5fca-4fc8-bd63-fdf587710c00 | Bobmart   | hopping
 b9257460-173b-45e8-b4c3-7ec269c8f74d | Crazytown | skipping
 4f32eadf-0abe-480a-93bc-8780fdf85b5d | Dory      | swimming
 216b0fd6-5baf-4432-80c6-069243e8cba4 | Eggbert   | sitting

SELECT * FROM exercises_0;
                  id                  | tenant |  name
--------------------------------------+--------+---------
 ac8e6a36-722e-4e53-b9e6-e916aded0b00 | Acme   | jumping
 284d1166-0bdc-4c66-93c1-43ddbd0491a0 | Acme   | running
```

Table modifications still work on the partitioned table, with modifications propagating to the partitions

```SQL
ALTER TABLE exercises ADD description VARCHAR;
```

```SQL
SELECT * FROM exercises;
                  id                  |  tenant   |   name   | description
--------------------------------------+-----------+----------+-------------
 ac8e6a36-722e-4e53-b9e6-e916aded0b00 | Acme      | jumping  |
 284d1166-0bdc-4c66-93c1-43ddbd0491a0 | Acme      | running  |
 aa46f49b-5fca-4fc8-bd63-fdf587710c00 | Bobmart   | hopping  |
 b9257460-173b-45e8-b4c3-7ec269c8f74d | Crazytown | skipping |
 4f32eadf-0abe-480a-93bc-8780fdf85b5d | Dory      | swimming |
 216b0fd6-5baf-4432-80c6-069243e8cba4 | Eggbert   | sitting  |

SELECT * FROM exercises_old;
                  id                  |  tenant   |   name   | description
--------------------------------------+-----------+----------+-------------
 aa46f49b-5fca-4fc8-bd63-fdf587710c00 | Bobmart   | hopping  |
 b9257460-173b-45e8-b4c3-7ec269c8f74d | Crazytown | skipping |
 4f32eadf-0abe-480a-93bc-8780fdf85b5d | Dory      | swimming |
 216b0fd6-5baf-4432-80c6-069243e8cba4 | Eggbert   | sitting  |

SELECT * FROM exercises_0;
                  id                  | tenant |  name   | description
--------------------------------------+--------+---------+-------------
 ac8e6a36-722e-4e53-b9e6-e916aded0b00 | Acme   | jumping |
 284d1166-0bdc-4c66-93c1-43ddbd0491a0 | Acme   | running |
```

If we want to break up the partitioning operation into chunks that deal with smaller amounts of data,
we can write them as separate operations that operate on a subset of the data each time.

Move some of the data
```SQL
BEGIN;

CREATE TABLE exercises_1 (
    LIKE exercises_old INCLUDING ALL
);

WITH x AS (
  DELETE FROM exercises_old WHERE tenant < 'C' returning *
)
INSERT INTO exercises_1
  SELECT * FROM x;

ALTER TABLE exercises
  ATTACH PARTITION exercises_1 FOR VALUES FROM ('B') TO ('C');

COMMIT;
```

Table `exercises_1` should have some of the data now.

```SQL
SELECT * FROM exercises;
                  id                  |  tenant   |   name   | description
--------------------------------------+-----------+----------+-------------
 ac8e6a36-722e-4e53-b9e6-e916aded0b00 | Acme      | jumping  |
 284d1166-0bdc-4c66-93c1-43ddbd0491a0 | Acme      | running  |
 aa46f49b-5fca-4fc8-bd63-fdf587710c00 | Bobmart   | hopping  |
 b9257460-173b-45e8-b4c3-7ec269c8f74d | Crazytown | skipping |
 4f32eadf-0abe-480a-93bc-8780fdf85b5d | Dory      | swimming |
 216b0fd6-5baf-4432-80c6-069243e8cba4 | Eggbert   | sitting  |

SELECT * FROM exercises_old;
                  id                  |  tenant   |   name   | description
--------------------------------------+-----------+----------+-------------
 b9257460-173b-45e8-b4c3-7ec269c8f74d | Crazytown | skipping |
 4f32eadf-0abe-480a-93bc-8780fdf85b5d | Dory      | swimming |
 216b0fd6-5baf-4432-80c6-069243e8cba4 | Eggbert   | sitting  |

SELECT * FROM exercises_0;
                  id                  | tenant |  name   | description
--------------------------------------+--------+---------+-------------
 ac8e6a36-722e-4e53-b9e6-e916aded0b00 | Acme   | jumping |
 284d1166-0bdc-4c66-93c1-43ddbd0491a0 | Acme   | running |

SELECT * FROM exercises_1;
                  id                  | tenant  |  name   | description
--------------------------------------+---------+---------+-------------
 aa46f49b-5fca-4fc8-bd63-fdf587710c00 | Bobmart | hopping |
```

Move the rest
```SQL
BEGIN;

ALTER TABLE exercises DETACH PARTITION exercises_1;

WITH x AS (
  DELETE FROM exercises_old WHERE tenant < 'D' returning *
)
INSERT INTO exercises_1
  SELECT * FROM x;

ALTER TABLE exercises
  ATTACH PARTITION exercises_1 FOR VALUES FROM ('B') TO ('D');

COMMIT;
```

Table `exercises_1` should have more of the data now.
```SQL
SELECT * FROM exercises;
                  id                  |  tenant   |   name   | description
--------------------------------------+-----------+----------+-------------
 ac8e6a36-722e-4e53-b9e6-e916aded0b00 | Acme      | jumping  |
 284d1166-0bdc-4c66-93c1-43ddbd0491a0 | Acme      | running  |
 aa46f49b-5fca-4fc8-bd63-fdf587710c00 | Bobmart   | hopping  |
 b9257460-173b-45e8-b4c3-7ec269c8f74d | Crazytown | skipping |
 4f32eadf-0abe-480a-93bc-8780fdf85b5d | Dory      | swimming |
 216b0fd6-5baf-4432-80c6-069243e8cba4 | Eggbert   | sitting  |

SELECT * FROM exercises_old;
                  id                  | tenant  |   name   | description
--------------------------------------+---------+----------+-------------
 4f32eadf-0abe-480a-93bc-8780fdf85b5d | Dory    | swimming |
 216b0fd6-5baf-4432-80c6-069243e8cba4 | Eggbert | sitting  |

SELECT * FROM exercises_0;
                  id                  | tenant |  name   | description
--------------------------------------+--------+---------+-------------
 ac8e6a36-722e-4e53-b9e6-e916aded0b00 | Acme   | jumping |
 284d1166-0bdc-4c66-93c1-43ddbd0491a0 | Acme   | running |

SELECT * FROM exercises_1;
                  id                  |  tenant   |   name   | description
--------------------------------------+-----------+----------+-------------
 aa46f49b-5fca-4fc8-bd63-fdf587710c00 | Bobmart   | hopping  |
 b9257460-173b-45e8-b4c3-7ec269c8f74d | Crazytown | skipping |
```

### Query planning
Let's see how Postgres plans different types of queries.

This scans the tables in all paritions.
```SQL
EXPLAIN SELECT * FROM exercises;
                               QUERY PLAN
------------------------------------------------------------------------
 Append  (cost=0.00..56.10 rows=1740 width=112)
   ->  Seq Scan on exercises_0  (cost=0.00..15.80 rows=580 width=112)
   ->  Seq Scan on exercises_1  (cost=0.00..15.80 rows=580 width=112)
   ->  Seq Scan on exercises_old  (cost=0.00..15.80 rows=580 width=112)
```

This searches all of the tables in all partitions.
```SQL
EXPLAIN SELECT * FROM exercises WHERE id = '284d1166-0bdc-4c66-93c1-43ddbd0491a0';
                                         QUERY PLAN
--------------------------------------------------------------------------------------------
 Append  (cost=0.15..24.52 rows=3 width=112)
   ->  Index Scan using exercises_0_pkey on exercises_0  (cost=0.15..8.17 rows=1 width=112)
         Index Cond: (id = '284d1166-0bdc-4c66-93c1-43ddbd0491a0'::uuid)
   ->  Index Scan using exercises_1_pkey on exercises_1  (cost=0.15..8.17 rows=1 width=112)
         Index Cond: (id = '284d1166-0bdc-4c66-93c1-43ddbd0491a0'::uuid)
   ->  Index Scan using exercises_pkey on exercises_old  (cost=0.15..8.17 rows=1 width=112)
         Index Cond: (id = '284d1166-0bdc-4c66-93c1-43ddbd0491a0'::uuid)
```

When given enough information in the query, Postgres is able to be smart and only execute the query
in the relevant partition.
```SQL
EXPLAIN SELECT * FROM exercises WHERE id = '284d1166-0bdc-4c66-93c1-43ddbd0491a0' AND tenant = 'Acme';
                                         QUERY PLAN
--------------------------------------------------------------------------------------------
 Append  (cost=0.15..8.18 rows=1 width=112)
   ->  Index Scan using exercises_0_pkey on exercises_0  (cost=0.15..8.17 rows=1 width=112)
         Index Cond: (id = '284d1166-0bdc-4c66-93c1-43ddbd0491a0'::uuid)
         Filter: ((tenant)::text = 'Acme'::text)
```

### Adding tables on remote servers to the partitioned table via Foreign Data Wrapper
Now let's set up a partition that references a table on remote server using [Postgres Foreign Data Wrapper] (postgres_fdw).

On the primary server, `db_1`.

Install the `postgres_fdw` extension.
```SQL
CREATE EXTENSION postgres_fdw;
```

Create a foreign server object to represent the remote database you want to connect to.
```SQL
CREATE SERVER db_2
  FOREIGN DATA WRAPPER postgres_fdw
  OPTIONS (host 'db_2', port '5432', dbname 'experiment');
```

Create a user mapping for each database user you want to allow to access the foreign server.
```SQL
CREATE USER MAPPING FOR experiment
  SERVER db_2
  OPTIONS (user 'experiment', password 'experiment');
```

Connect to `db_2`.
```shell
psql -h localhost -p 5433 -U experiment
```

On `db_2`, create a table that matches the schema of the partition table
```SQL
-- on db_2

CREATE TABLE exercises_2 (
  id UUID PRIMARY KEY,
  tenant VARCHAR NOT NULL,
  name VARCHAR NOT NULL,
  description VARCHAR
);

-- TODO: look into a better way to get the schema from the primary table
```

On `db_1`.

Create a foreign table for the remote table, move data over to it, and attach it as a partition

```SQL
-- on db_1

BEGIN;

-- create a foreign table
IMPORT FOREIGN SCHEMA public LIMIT TO (exercises_2)
  FROM SERVER db_2 INTO public;

-- move data
WITH x AS (
  DELETE FROM exercises_old WHERE tenant < 'E' returning *
)
INSERT INTO exercises_2
  SELECT * FROM x;

-- attach as partition
ALTER TABLE exercises
  ATTACH PARTITION exercises_2 FOR VALUES FROM ('D') TO ('E');

COMMIT;
```

The data is now in the table on the remote server, queries to the main table will include the data on the remote server

```SQL
SELECT * FROM exercises;
                  id                  |  tenant   |   name   | description
--------------------------------------+-----------+----------+-------------
 ac8e6a36-722e-4e53-b9e6-e916aded0b00 | Acme      | jumping  |
 284d1166-0bdc-4c66-93c1-43ddbd0491a0 | Acme      | running  |
 aa46f49b-5fca-4fc8-bd63-fdf587710c00 | Bobmart   | hopping  |
 b9257460-173b-45e8-b4c3-7ec269c8f74d | Crazytown | skipping |
 4f32eadf-0abe-480a-93bc-8780fdf85b5d | Dory      | swimming |
 216b0fd6-5baf-4432-80c6-069243e8cba4 | Eggbert   | sitting  |

SELECT * FROM exercises_old;
                  id                  | tenant  |  name   | description
--------------------------------------+---------+---------+-------------
 216b0fd6-5baf-4432-80c6-069243e8cba4 | Eggbert | sitting |

SELECT * FROM exercises_0;
                  id                  | tenant |  name   | description
--------------------------------------+--------+---------+-------------
 ac8e6a36-722e-4e53-b9e6-e916aded0b00 | Acme   | jumping |
 284d1166-0bdc-4c66-93c1-43ddbd0491a0 | Acme   | running |

SELECT * FROM exercises_1;
                  id                  |  tenant   |   name   | description
--------------------------------------+-----------+----------+-------------
 aa46f49b-5fca-4fc8-bd63-fdf587710c00 | Bobmart   | hopping  |
 b9257460-173b-45e8-b4c3-7ec269c8f74d | Crazytown | skipping |

SELECT * FROM exercises_2;
                  id                  | tenant |   name   | description
--------------------------------------+--------+----------+-------------
 4f32eadf-0abe-480a-93bc-8780fdf85b5d | Dory   | swimming |
```

### Limitations

Let's see if table alterations continue to propagate out from the primary partitioned table

```SQL
ALTER TABLE exercises ADD difficulty SMALLINT;
```

Unfortunately, the foreign table is not altered

```SQL
SELECT * FROM exercises;

-- ERROR:  column "difficulty" does not exist
-- CONTEXT:  remote SQL command: SELECT id, tenant, name, description, difficulty FROM public.exercises_2
```

To fix this, we'll need to alter the table on the remote server, too.

On `db_2`.
```SQL
-- on db_2

ALTER TABLE exercises_2 ADD difficulty SMALLINT;
```

Now the query should work as expected.
```SQL
-- on db_1
SELECT * FROM exercises;
                  id                  |  tenant   |   name   | description | difficulty
--------------------------------------+-----------+----------+-------------+------------
 ac8e6a36-722e-4e53-b9e6-e916aded0b00 | Acme      | jumping  |             |
 284d1166-0bdc-4c66-93c1-43ddbd0491a0 | Acme      | running  |             |
 aa46f49b-5fca-4fc8-bd63-fdf587710c00 | Bobmart   | hopping  |             |
 b9257460-173b-45e8-b4c3-7ec269c8f74d | Crazytown | skipping |             |
 4f32eadf-0abe-480a-93bc-8780fdf85b5d | Dory      | swimming |             |
 216b0fd6-5baf-4432-80c6-069243e8cba4 | Eggbert   | sitting  |             |
```

## TODO
- test out [transactions across foreign server with postgres_fdw](https://www.postgresql.org/docs/11/postgres-fdw.html#id-1.11.7.42.12)
- try out different partitioning schemes - LIST, HASH
- examine performance for partitioning, FDW
- look into improving dev experience around `ALTER TABLE` limitation with foreign tables. Maybe leverage a trigger as a workaround?
- Is this [foreign key limitation](https://www.postgresql.org/docs/11/ddl-partitioning.html#DDL-PARTITIONING-DECLARATIVE-LIMITATIONS) a dealbreaker? "While primary keys are supported on partitioned tables, foreign keys referencing partitioned tables are not supported. (Foreign key references from a par"titioned table to some other table are supported.)"

## Resources
- https://pgdash.io/blog/postgres-11-sharding.html
- https://www.depesz.com/2019/03/19/migrating-simple-table-to-partitioned-how/
- https://www.postgresql.org/docs/11/postgres-fdw.html
- https://www.postgresql.org/docs/11/ddl-partitioning.html


[Postgres Foreign Data Wrapper]: https://www.postgresql.org/docs/11/postgres-fdw.html
[Postgres Declarative Table Partitioning]: https://www.postgresql.org/docs/11/ddl-partitioning.html