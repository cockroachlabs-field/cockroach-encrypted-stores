# Selective Encryption for CockroachDB Tables

*This tutorial walks through setting up two disks per node, with one disk encrypted and containing sensitive information and the other disk having plaintext data.*

## Motivation

A few days ago, both Artem and Chris were working with two different customers that had a requirement to encrypt only certain tables in the database.  So we joined forces to create a demonstration and published this blog post.  Both customers acknowledged the slight overhead for using data at rest encryption.  However, both customers did not want to incur the ever so slight penalty on less sensitive data.  So we embarked on creating a CockroachDB cluster that can encrypt sensitive data but leave non-sensitive data in plaintext.  To do this, it requires utilizing multiple CockroachDB stores on each of the Cockroach nodes with the proper locality attribute flags.  Once setup, zone configurations can be applied to a database, tables and/or rows.  In this blog, we setup the zone configuration at the table level to use encrypted store, or not. The setup for this is rather simple which this blog will walk you through.

#### High Level Steps
- Create an Encryption Key (AES-128)
- Start a CockroachDB Cluster with an Encrypted and Non-Encrypted Store
- Create & Assign PII and Non PII tables to Encrypted and Non-Encrypted stores
- Verify Setup

----

## Step By Step Instructions
### Create an Encryption Key

First, let's assign some variables to easily manage this setup in a scriptable fashion.  We'll create a `$storepath` variable which will tell CockroachDB where to keep it's data.  And we'll use a `$keypath` variable which will have the location of our encryption key.

```bash
export keypath="${PWD}/workdir/key"
export storepath="${PWD}/workdir/data"
mkdir -p "${keypath}"
mkdir -p "${storepath}"
```

Create an Encryption Key

```bash
cockroach gen encryption-key -s 128 $keypath/aes-128.key
```

Confirm the Encryption Key was created

`successfully created AES-128 key: /Users/artem/Documents/cockroach-work/cockroach-demo/workdir/aes-128.key`

### Start a CockroachDB Cluster with an Encrypted and Non-Encrypted Store.

Next, let's create a cluster with an encrypted store and a plaintext store.  The syntax for this is `--store=path=${dir}/1e/data,attrs=encrypt` and `--store=path=${dir}/1o/data,attrs=open` respectively. The attributes at the end of the store creation are used pinning a table to an encrypted store. Additionally, the encryption key we created in the prior step is referenced in the `--enterprise-encryption` flag as well.

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

```bash
cockroach sql --insecure -e "SHOW ALL ZONE CONFIGURATIONS;"
```

Confirm the output has the right constraints.  PII should have a constraint of "Encrypt" and Non-PII should have a contraint of "Open" )

`...`

`TABLE defaultdb.public.pii     | ALTER TABLE defaultdb.public.pii CONFIGURE ZONE USING     | constraints = '[+encrypt]'`

`TABLE defaultdb.public.non_pii | ALTER TABLE defaultdb.public.non_pii CONFIGURE ZONE USING | constraints = '[+open]'`



## Verify Setup

Verifying that the tables are in the right place requires us to query some of the internal meta data of CockroachDB.  To do this, we need to do the following:
- Find the Table Ranges
- Find the Encrypted and Non-Encrypted Store Ids
- Confirm that the ranges from the tables are mapped to the correct stores.

Let's start a SQL terminal session to run these commands

```bash
cockroach sql --insecure
```

#### Find the Table Ranges

Using the `SHOW RANGES` command, we can easily find the range ids for the pii table.

```sql
SELECT range_id, replicas FROM [SHOW RANGES FROM TABLE pii];
```

The output shows the range ids that we want to verify.  As you can see, we have one range to verfiy (range_id = 37).

```
range_id | replicas
-----------+-----------
      37 | {1,3,5}
```

#### Find the Encrypted and Non-Encrypted Store Ids

Now let's query an internal table called `crdb_internal.kv_store_status` to find out all of the stores being used in the cluster and which attributes they have.

```sql
SELECT node_id, store_id, attrs
FROM crdb_internal.kv_store_status;
```

As you can see below, there are 6 stores.  3 of which have an **["encrypt"]** attribute and 3 that have an **["open"]** attribute.


```
node_id | store_id |    attrs
----------+----------+--------------
      1 |        1 | ["encrypt"]
      1 |        2 | ["open"]
      2 |        3 | ["encrypt"]
      2 |        4 | ["open"]
      3 |        5 | ["encrypt"]
      3 |        6 | ["open"]
```

#### Confirm that the ranges from the tables are mapped to the correct stores.

Last but not least, let's confirm that the ranges from the tables are mapped to the correct stores.  This query is bit sophisticated but it will get us the right answer.  The query looks up the range id, node id and store id for a given table name.  So the "PII" table, we would expect stores 1,3 and 5 to be in the output.


```sql
SELECT range_id, node_id, repls.store_id
FROM
(
  SELECT range_id, unnest(replicas) AS store_id
	FROM crdb_internal.ranges_no_leases
	WHERE table_name = 'pii'
) AS repls
JOIN crdb_internal.kv_store_status AS ss ON (ss.store_id = repls.store_id)
ORDER BY range_id;
```

And as expected, store ids for PII table ranges include store ids 1, 3, and 5.

```
range_id | node_id | store_id
-----------+---------+-----------
37 |       1 |        1
37 |       2 |        3
37 |       3 |        5
(3 rows)
```

The same step above can be used to confirm the store ids for the Non-PII table.


```sql
SELECT range_id, replicas FROM [SHOW RANGES FROM TABLE non_pii];
```

```
  range_id | replicas
-----------+-----------
        38 | {2,4,6}
```


```sql
SELECT range_id, node_id, repls.store_id
FROM
(
  SELECT range_id, unnest(replicas) AS store_id
  FROM crdb_internal.ranges_no_leases
  WHERE table_name = 'non_pii'
) AS repls
JOIN crdb_internal.kv_store_status AS ss ON (ss.store_id = repls.store_id)
ORDER BY range_id;
```

```
range_id | node_id | store_id
-----------+---------+-----------
38 |       1 |        2
38 |       2 |        4
38 |       3 |        6
(3 rows)
```

And indeed, the range with range_id 38 is stored on stores 2, 4 and 6!

## Clean Up

To clean up your work, kill the CockroachDB process and remove the directories you created above.

```bash
pkill -9 cockroach
rm -Rf "${keypath}"
rm -Rf "${storepath}"
```
