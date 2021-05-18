# Selective Encryption for CockroachDB Stores

*This tutorial walks through setting up two disks per node, with one disk encrypted and containing sensitive information and the other disk having plaintext data.*

## Motivation

A few days ago, both Artem and Chris were working with two different customers that had a requirement to encrypt only certain tables in the database.  So we joined forces to create a demonstration and published this blog post.  Both customers acknowledged the slight overhead for using data at rest encryption.  However, both customers did not want to incur the ever so slight penalty on less sensitive data.  So we embarked on creating a CockroachDB cluster that can encrypt sensitive data but leave non-sensitive data in plaintext.  To do this, it requires utilizing multiple CockroachDB stores on each of the Cockroach nodes with the proper locality attribute flags.  Once setup, zone configurations can be applied for a database, tables and/or rows.  In this blog, we setup the zone configuration at the table level to use encrypted store, or not. The setup for this is rather simple which this blog will walk you through.

#### High Level Steps
- Create an Encryption Key (AES-128)
- Start a CockroachDB Cluster with an Encrypted Non-Encrypted Store
- Create & Assign PII and Non PII tables to Encrypted and Non-Encrypted stores
- Insert Sample Data
- Verify Data

----

## Step By Step Instructions
### Create an Encryption Key

First, let's assign some variables to easily manage this setup in a scriptable fashion.  We'll create a `$storepath` variable which will tell CockroachDB where to keep it's data.  And we'll use a `$keypath` variable which will have the location of our encryption key.

```bash
export keypath="${PWD}/workdir/key"
export storepath="${PWD}/workdir/data"
```

Create an Encryption Key

```bash
cockroach gen encryption-key -s 128 $keypath/aes-128.key
```

Confirm the Encryption Key was created
```
successfully created AES-128 key: /Users/artem/Documents/cockroach-work/cockroach-demo/workdir/aes-128.key
```

### Start a CockroachDB Cluster with an Encrypted and Non-Encrypted Store.

Next, let's create a cluster with an encrypted store and a plaintext store.  The syntax for this is `--store=path=${dir}/1e/data,attrs=encrypt` and `--store=path=${dir}/1o/data,attrs=open` respectively. The attributes at the end of the store creation is used pinning a table to an encrypted store or not. Additionally, the encryption key we created in the prior step is referenced in the `--enterprise-encryption` flag as well.

```bash
cockroach start \
--insecure \
--store=path=$storepath/1e/data,attrs=encrypt \
--store=path=$storepath/1o/data,attrs=open \
--enterprise-encryption=path=$storepath/1e/data,key=$keypath/aes-128.key,old-key=plain \
--listen-addr=127.0.0.1 \
--port=26257 \
--http-port=8080 \
--locality=region=local,zone=local \
--join=127.0.0.1:26257 \
--background

cockroach start \
--insecure \
--store=path=$storepath/2e/data,attrs=encrypt \
--store=path=$storepath/2o/data,attrs=open \
--enterprise-encryption=path=$storepath/2e/data,key=$keypath/aes-128.key,old-key=plain \
--listen-addr=127.0.0.1 \
--port=26259 \
--http-port=8081 \
--locality=region=local,zone=local \
--join=127.0.0.1:26257 \
--background

cockroach start \
--insecure \
--store=path=$storepath/3e/data,attrs=encrypt \
--store=path=$storepath/3o/data,attrs=open \
--enterprise-encryption=path=$storepath/3e/data,key=$keypath/aes-128.key,old-key=plain \
--listen-addr=127.0.0.1 \
--port=26261 \
--http-port=8082 \
--locality=region=local,zone=local \
--join=127.0.0.1:26257 \
--background
```

Now that the cluster is created, let’s initialize it.

```bash
cockroach init --insecure
```

### Create & Assign PII and Non PII tables to Encrypted and Non-Encrypted stores


This is the cool part, let’s create a PII and a Non-PII tables and put each of them in the proper ecrypted or non-encrypted stores.  For the PII table, we'll create the table, set it's zone configuration to use the encrypted store and then insert some sample data to confirm our setup.

Let’s create the PII table, apply it to the encrypted store and add some sample data.

```bash
cockroach sql --insecure \
-e "create table pii (k int primary key, v string);" \
-e "alter table pii configure zone using constraints='[+encrypt]';" \
-e "insert into pii (k,v) values (1,'bob');"
```

Similarly, let’s create a Non-PII table, apply it to the non-encrypted store and add some sample data.

```bash
cockroach sql --insecure \
-e "create table non_pii (k int primary key, v string);" \
-e "alter table non_pii configure zone using constraints='[+open]';" \
-e "insert into non_pii (k,v) values (1,'bob');"
```

Now let’s look at the ranges of the table to see if they have been moved to the encrypted store.

```sql
SHOW ALL ZONE CONFIGURATIONS;
```

we are going to focus only on the last two tables, the trimmed output is below


```sql
TABLE defaultdb.public.pii                       | ALTER TABLE defaultdb.public.pii CONFIGURE ZONE USING
                                                 |     constraints = '[+encrypt]'
TABLE defaultdb.public.non_pii                   | ALTER TABLE defaultdb.public.non_pii CONFIGURE ZONE USING
                                                 |     constraints = '[+open]'
```

```sql
SHOW RANGES FROM TABLE pii;
```

```sql
start_key | end_key | range_id | range_size_mb | lease_holder | lease_holder_locality | replicas |                               replica_localities
------------+---------+----------+---------------+--------------+-----------------------+----------+----------------------------------------------------------------------------------
NULL      | NULL    |       37 |      0.000027 |            5 | NULL                  | {1,3,5}  | {"region=local,zone=local","region=local,zone=local","region=local,zone=local"}
```

We can even inspect the range at the per row level

```sql
SHOW RANGE FROM TABLE pii FOR ROW ('1');
```

Notice that the range in question is number 37

```sql
SELECT
	range_id, node_id, repls.store_id
FROM
	(
		SELECT
			range_id, unnest(replicas) AS store_id
		FROM
			crdb_internal.ranges_no_leases
		WHERE
			table_name = 'pii'
	)
		AS repls
	JOIN crdb_internal.kv_store_status AS ss ON (ss.store_id = repls.store_id)
ORDER BY
	range_id;
```

```sql
range_id | node_id | store_id
-----------+---------+-----------
37 |       1 |        1
37 |       2 |        3
37 |       3 |        5
```

We can see that range 37 associated with the table `pii` is on stores 1, 3, 5 across the three nodes.

Let's list out the stores and their associated attributes per node

```sql
SELECT node_id, store_id, attrs
FROM crdb_internal.kv_store_status;
```

```sql
node_id | store_id |    attrs
----------+----------+--------------
      1 |        1 | ["encrypt"]
      1 |        2 | ["open"]
      2 |        3 | ["encrypt"]
      2 |        4 | ["open"]
      3 |        5 | ["encrypt"]
      3 |        6 | ["open"]
```

and indeed, store_ids 1, 3, 5 are associated with the encrypted disk.

Let's take a look at the range associated with the `non-pii` table.

```sql
SELECT range_id, replicas FROM [SHOW RANGES FROM TABLE non_pii];
```

```sql
SELECT range_id, replicas FROM [SHOW RANGES FROM TABLE non_pii];
```

```sql
root@:26257/defaultdb> SELECT range_id, replicas FROM [SHOW RANGES FROM TABLE non_pii];
  range_id | replicas
-----------+-----------
        38 | {2,4,6}
```

The range in question is 38.

Let's confirm the range_id and store_id correllate

```sql
SELECT        range_id, node_id, repls.store_id
FROM
        (
                SELECT                        range_id, unnest(replicas) AS store_id
                FROM
                        crdb_internal.ranges_no_leases
                WHERE
                        table_name = 'non_pii'
        )
                AS repls
        JOIN crdb_internal.kv_store_status AS ss ON (ss.store_id = repls.store_id)
ORDER BY
        range_id;
```

```sql
range_id | node_id | store_id
-----------+---------+-----------
      38 |       1 |        2
      38 |       2 |        4
      38 |       3 |        6
```

and indeed, the range with range_id 38 is stored on stores 2, 4 and 6.
