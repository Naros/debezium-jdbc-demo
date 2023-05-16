# Debezium JDBC Sink Demo

This example demonstrates the power of the Debezium JDBC sink connector paired with a Debezium source connector, replicating changes from a source database to a sink database via JDBC.

The overall architecture of this demonstration includes the following:

* Zookeeper
* Apache Kafka broker
* Kafka Connect
* Debezium connectors (based on Debezium 2.3)

## Preparation

The project's `docker-compose.yml` and `connect/Dockerfile` artifacts directly reference the specific Debezium version being used, so there are no environment variables to set.
If you wish to use a different Debezium version, these aforementioned artifacts will need to be adjusted accordingly.

## Starting the demo

### Start containers

A `docker-compose.yml` file has been provided to start the containers.
Open a terminal window and run the following command:

    docker-compose up 

### Deploy the Source Connector

We are going to use a MySQL Debezium source connector to gather the changes from a MySQL source database.
The connector's configuration is defined in `register-source-mysql.json`.

In order to create this connector, we will utilize `Httpie` to interact with Kafka Connect's RESTful endpoints.
If you do not have `http` installed, you can install it based on your environment, or you can use `cURL`.
In a terminal, run the following to deploy the source connector:

    http POST http://localhost:8083/connectors @register-source-mysql.json

After the connector has been deployed, there will be several topics created on the Kafka broker.
If you wish to check this, simply use the following command:

    docker exec -it jdbc-kafka ./bin/kafka-topics.sh --list --bootstrap-server=kafka:9092

We can see the following from the output:

```
__consumer_offsets
connect_configs
connect_offsets
connect_statuses
mysql
mysql-database-history
mysql.inventory.addresses
mysql.inventory.customers
```

### Deploy the Sink Connector

The Debezium JDBC sink connector will be used in order to stream changes from Kafka into our target JDBC database.
For this demonstration, we're going to use a PostgreSQL target database.
The JDBC sink connector's configuration is defined in `register-sink-postgres.json`.

In order to create this connector, we will utilize `Httpie` to interact with Kafka Connect's RESTful endpoints.
If you do not have `http` installed, you can install it based on your environment, or you can use `cURL`.
In a terminal, run the following to deploy the sink connector:

    http POST http://localhost:8083/connectors @register-sink-postgres.json

The sink connector is configured to capture changes from the Kafka topic related to the source `inventory.customers` table.
Once the sink connector has been deployed, we can check that the data has been replicated to PostgreSQL.
In a terminal, run the following command using password `dbz`:

    docker exec -it jdbc-postgres psql --username postgres --password postgres 

Once in the PostgreSQL CLI, run this query:

    select * from mysql_inventory_customers;

The expected output should look like:

```
  id  | first_name | last_name |         email         
------+------------+-----------+-----------------------
 1001 | Sally      | Thomas    | sally.thomas@acme.com
 1002 | George     | Bailey    | gbailey@foobar.com
 1003 | Edward     | Walker    | ed@walker.com
 1004 | Anne       | Kretchmar | annek@noanswer.org
(4 rows)
```

Now lets check the table's constructed layout by running `\d mysql_inventory_customers`.
The output of this command is:

```
       Table "public.mysql_inventory_customers"
   Column   |  Type   | Collation | Nullable | Default 
------------+---------+-----------+----------+---------
 id         | integer |           | not null | 
 first_name | text    |           | not null | 
 last_name  | text    |           | not null | 
 email      | text    |           | not null | 
```

You might notice that the destination character-based columns are created as `TEXT` data types.
Unfortunately, the source events do not provide any details about the column's length by default, so the JDBC sink connector must pick the most generalized type.

### Deploy with column/data type propagation

Debezium's has the ability to produce events with more column-based details including type, length, and precision.
In order to see how this works, the demonstration includes one additional JSON file, `register-source-mysql-propagation.json`.
W
hen taking a look at the contents of this file, what's important are the following entries:

```json
{
  "column.propagate.source.type": ".*",
  "datatype.propagate.source.type": ".*"
}
```

These entries instruct Debezium to emit change events with column and data type details.

In order to see the differences, lets deploy this new source connector.
Open a terminal window and run the following:

    http POST http://localhost:8083/connectors @register-source-mysql-propagation.json

Once the connector has been deployed, it will regenerate the original snapshot but mapped to a slightly different topic name.
These changes should be observed by the sink connector and written to PostgreSQL.
Go back to your PostgreSQL cli window and execute this query:

    select * from mysql_propagation_inventory_customers;

As we can see from the following output, the content looks identical to the other table:

```
  id  | first_name | last_name |         email         
------+------------+-----------+-----------------------
 1001 | Sally      | Thomas    | sally.thomas@acme.com
 1002 | George     | Bailey    | gbailey@foobar.com
 1003 | Edward     | Walker    | ed@walker.com
 1004 | Anne       | Kretchmar | annek@noanswer.org
(4 rows)
```

Now lets look at the table's structure by running `\d mysql_propagation_inventory_customers`

```
         Table "public.mysql_propagation_inventory_customers"
   Column   |          Type          | Collation | Nullable | Default 
------------+------------------------+-----------+----------+---------
 id         | integer                |           | not null | 
 first_name | character varying(255) |           | not null | 
 last_name  | character varying(255) |           | not null | 
 email      | character varying(255) |           | not null | 
```

We can now see that each field is no longer `TEXT` but rather `character varying` with a length of 255.
If we login to the MySQL database using password `dbz` and check the structure of `inventory.customers`, we'll see this:

```
mysql> desc customers;
+------------+--------------+------+-----+---------+----------------+
| Field      | Type         | Null | Key | Default | Extra          |
+------------+--------------+------+-----+---------+----------------+
| id         | int          | NO   | PRI | NULL    | auto_increment |
| first_name | varchar(255) | NO   |     | NULL    |                |
| last_name  | varchar(255) | NO   |     | NULL    |                |
| email      | varchar(255) | NO   | UNI | NULL    |                |
+------------+--------------+------+-----+---------+----------------+
4 rows in set (0.00 sec)
```

### Testing data changes

While looking at the `inventory.customers` table in MySQL, execute the following update:

    UPDATE customers set email = 'ed.walker@walker.com' WhERE id = 1003

If we go back and check the contents in PostgreSQL:

```
postgres=# select * from mysql_inventory_customers;
  id  | first_name | last_name |         email         
------+------------+-----------+-----------------------
 1001 | Sally      | Thomas    | sally.thomas@acme.com
 1002 | George     | Bailey    | gbailey@foobar.com
 1004 | Anne       | Kretchmar | annek@noanswer.org
 1003 | Edward     | Walker    | ed.walker@walker.com
(4 rows)

postgres=# select * from mysql_propagation_inventory_customers;
  id  | first_name | last_name |         email         
------+------------+-----------+-----------------------
 1001 | Sally      | Thomas    | sally.thomas@acme.com
 1002 | George     | Bailey    | gbailey@foobar.com
 1004 | Anne       | Kretchmar | annek@noanswer.org
 1003 | Edward     | Walker    | ed.walker@walker.com
(4 rows)
```

We can see in both tables, the change to the row with ID of 1003 has been replicated.

Now let's add a new customer record in MySQL:

    INSERT INTO customers (id,first_name,last_name,email) values (1005, 'Mickey', 'Mouse', 'mickey@disney.com');

If we look back in PostgeSQL we'll see the change have been replicated:

```
postgres=# select * from mysql_inventory_customers;
  id  | first_name | last_name |         email         
------+------------+-----------+-----------------------
 1001 | Sally      | Thomas    | sally.thomas@acme.com
 1002 | George     | Bailey    | gbailey@foobar.com
 1004 | Anne       | Kretchmar | annek@noanswer.org
 1003 | Edward     | Walker    | ed.walker@walker.com
 1005 | Mickey     | Mouse     | mickey@disney.com
(5 rows)

postgres=# select * from mysql_propagation_inventory_customers;
  id  | first_name | last_name |         email         
------+------------+-----------+-----------------------
 1001 | Sally      | Thomas    | sally.thomas@acme.com
 1002 | George     | Bailey    | gbailey@foobar.com
 1004 | Anne       | Kretchmar | annek@noanswer.org
 1003 | Edward     | Walker    | ed.walker@walker.com
 1005 | Mickey     | Mouse     | mickey@disney.com
(5 rows)
```

And finally, lets delete our recently added customer with ID 1005.  So in MySQL execute:

    DELETE FROM customers WHERE id = 1005;

We will see this in PostgreSQL:

```
postgres=# select * from mysql_inventory_customers;
  id  | first_name | last_name |         email         
------+------------+-----------+-----------------------
 1001 | Sally      | Thomas    | sally.thomas@acme.com
 1002 | George     | Bailey    | gbailey@foobar.com
 1004 | Anne       | Kretchmar | annek@noanswer.org
 1003 | Edward     | Walker    | ed.walker@walker.com
(4 rows)

postgres=# select * from mysql_propagation_inventory_customers;
id  | first_name | last_name |         email
------+------------+-----------+-----------------------
1001 | Sally      | Thomas    | sally.thomas@acme.com
1002 | George     | Bailey    | gbailey@foobar.com
1004 | Anne       | Kretchmar | annek@noanswer.org
1003 | Edward     | Walker    | ed.walker@walker.com
(4 rows)
```

