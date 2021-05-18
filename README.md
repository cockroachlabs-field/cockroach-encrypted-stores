<<<<<<< HEAD
# Selective Encryption for CockroachDB Stores
=======
# Encryption at rest
>>>>>>> 0886063 (wip)

#### This tutorial walks through setting up two disks per node, with one disk encrypted and containing sensitive information and the other disk having plaintext data.

A few days ago I was working with a customer that had a requirement to encrypt only certain tables in the database.  The customer acknowledged that there is a slight overhead for using data at rest encryption and they didn't want to incur the ever so slight penalty on less sensitive data.  Creating a CockroachDB cluster that can encrypt sensitive data but leave non-sensitive data in plaintext,  requires creating multiple stores on each of the Cockroach nodes with the proper locality attribute flags.  Once setup, zone configurations for database, tables and rows can be used to pin data to either an encrypted store or a plain text store.  The setup for this is rather simple which this blog will walk you through.


## Create encryption key to encrypt a CockroachDB store

```bash
mkdir -p ${PWD}/workdir
export dir="${PWD}/workdir"
```

```bash
cockroach gen encryption-key -s 128 ${dir}/aes-128.key
```

```bash
successfully created AES-128 key: /Users/artem/Documents/cockroach-work/cockroach-demo/workdir/aes-128.key
```

## Create a CockroachDB cluster with 3 nodes containing two stores, one encrypted and one plaintext

The syntax for this is --store=path=${dir}/1e/data,attrs=encrypt and --store=path=${dir}/1o/data,attrs=open respectively. The attributes at the end of the store creation is used to create a reference for pinning database, table or row of data to an encrypted store or not using zone configurations. Additionally, the encryption key we create in the prior step is reference in the --enterprise-encryption flag as well.


```bash
cockroach start \
--insecure \
--store=path=${dir}/1e/data,attrs=encrypt \
--store=path=${dir}/1o/data,attrs=open \
--log-dir=${dir}/1/logs \
--enterprise-encryption=path=${dir}/1e/data,key=/Users/artem/Documents/cockroach-work/cockroach-demo/workdir/aes-128.key,old-key=plain \
--listen-addr=127.0.0.1 \
--port=26257 \
--http-port=8080 \
--locality=region=local,zone=local \
--join=127.0.0.1:26257 \
--background

cockroach start \
--insecure \
--store=path=${dir}/2e/data,attrs=encrypt \
--store=path=${dir}/2o/data,attrs=open \
--log-dir=${dir}/2/logs \
--enterprise-encryption=path=${dir}/2e/data,key=/Users/artem/Documents/cockroach-work/cockroach-demo/workdir/aes-128.key,old-key=plain \
--listen-addr=127.0.0.1 \
--port=26259 \
--http-port=8081 \
--locality=region=local,zone=local \
--join=127.0.0.1:26257 \
--background

cockroach start \
--insecure \
--store=path=${dir}/3e/data,attrs=encrypt \
--store=path=${dir}/3o/data,attrs=open \
--log-dir=${dir}/3/logs \
--enterprise-encryption=path=${dir}/3e/data,key=/Users/artem/Documents/cockroach-work/cockroach-demo/workdir/aes-128.key,old-key=plain \
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

Now the fun part, let’s create a customers table, put some data in and pin it to the encrypted stores

```bash
cockroach sql --insecure \
-e "create table customers (k int primary key, v string);" \
-e "insert into customers (k,v) values (1,'bob');" \
-e "alter table customers configure zone using constraints='[+encrypt]';"
```

Now let’s look at the ranges of the table to see if they have been move to the encrypted store.

```sql
SHOW ZONE CONFIGURATION FOR TABLE customers;
```

```sql
      target      |               raw_config_sql
------------------+---------------------------------------------
  TABLE customers | ALTER TABLE customers CONFIGURE ZONE USING
                  |     range_min_bytes = 134217728,
                  |     range_max_bytes = 536870912,
                  |     gc.ttlseconds = 90000,
                  |     num_replicas = 3,
                  |     constraints = '[+encrypt]',
                  |     lease_preferences = '[]'
(1 row)

Time: 5ms total (execution 5ms / network 0ms)
```

```sql
SHOW RANGES FROM TABLE customers;
```

```sql
  start_key | end_key | range_id | range_size_mb | lease_holder |  lease_holder_locality  | replicas |                               replica_localities
------------+---------+----------+---------------+--------------+-------------------------+----------+----------------------------------------------------------------------------------
  NULL      | NULL    |       36 |      0.000027 |            3 | region=local,zone=local | {1,3,5}  | {"region=local,zone=local","region=local,zone=local","region=local,zone=local"}
(1 row)

Time: 8ms total (execution 8ms / network 0ms)
```

```sql
SHOW RANGE FROM TABLE customers FOR ROW ('1');
```

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
