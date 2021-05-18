# Selective Encryption for CockroachDB Stores
=======

## This tutorial walks through setting up two disks per node, with one disk encrypted and containing sensitive information and the other disk having plaintext data.

A few days ago I was working with a customer that had a requirement to encrypt only certain tables in a database. The customer acknowledged that there is a slight overhead for using data at rest encryption and they didn't want to incur the ever so slight penalty on less sensitive data. Creating a CockroachDB cluster that can encrypt sensitive data but leave non-sensitive data in plaintext, requires creating multiple stores on each of the Cockroach nodes with the proper locality attribute flags. Once setup, zone configurations for database, tables and rows can be used to pin data to either an encrypted store or a plain text store. The setup for this is rather simple which this blog will walk you through.

The steps to set up encryption are documented [here](https://www.cockroachlabs.com/docs/v21.1/encryption.html).

### Setup a working directory and an environment variable

```bash
export keypath="${PWD}/workdir/key"
export storepath="${PWD}/workdir/data"
```

### Create an encryption key to encrypt a CockroachDB store

```bash
cockroach gen encryption-key -s 128 $keypath/aes-128.key
```

```bash
successfully created AES-128 key: /Users/artem/Documents/cockroach-work/cockroach-demo/workdir/aes-128.key
```

### Create a CockroachDB cluster with 3 nodes containing two stores each, one encrypted and one plaintext

The syntax for this is `--store=path=${dir}/1e/data,attrs=encrypt` and `--store=path=${dir}/1o/data,attrs=open` respectively. The attributes at the end of the store creation are used to create a reference for pinning a database, a table or a row of data to an encrypted store. Additionally, the encryption key we created in the prior step is referenced in the `--enterprise-encryption` flag as well.

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

Now the fun part, let’s create a `pii` table, put some data in and pin it to the encrypted stores

```bash

cockroach sql --insecure \
-e "create table pii (k int primary key, v string);" \
-e "alter table pii configure zone using constraints='[+encrypt]';" \
-e "insert into pii (k,v) values (1,'bob');"
```

Let's create a new table containing non-sensitive data called `non-pii`, pin it to the plaintext store

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

-- show node attributes per node
SELECT node_id, attrs, locality FROM crdb_internal.kv_node_status;

-- show store attributes per node
SELECT node_id, store_id, attrs FROM crdb_internal.kv_store_status;

SELECT
	node_id, repls.store_id, count(repls.store_id)
FROM
	(
		SELECT
			unnest(replicas) AS store_id
		FROM
			crdb_internal.ranges_no_leases
		WHERE
			table_name = 'customers'
	)
		AS repls
	JOIN crdb_internal.kv_store_status AS ss ON (ss.store_id = repls.store_id)
GROUP BY
	node_id, repls.store_id;


SELECT
	range_id, node_id, repls.store_id
FROM
	(
		SELECT
			range_id, unnest(replicas) AS store_id
		FROM
			crdb_internal.ranges_no_leases
		WHERE
			table_name = 'customers'
	)
		AS repls
	JOIN crdb_internal.kv_store_status AS ss ON (ss.store_id = repls.store_id)
ORDER BY
	range_id;
```

### Force the plaintext data off of encrypted disk

```sql
ALTER TABLE customers2 CONFIGURE ZONE USING CONSTRAINTS='[-encrypt]';
```
