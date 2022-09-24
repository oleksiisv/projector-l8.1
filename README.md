# Transactions, Isolations, Locks

## Installation

### Percona

Run ```docker composer up -d``` to launch percona container. 

Enable innodb_status_output

```SET GLOBAL innodb_status_output=ON;```\
```SET GLOBAL innodb_status_output_locks=ON;```

Import sample data from ./percona/l81_users.sql 

Run ``` mysql -uroot -proot123 -h127.0.0.1 -P3308 l81``` to connect to the database. 

### Postgres

I wasn't lucky to build postgres container from docker-compose, there were multile issues with connections and permissions along with database creation. So used the manual approach. 

Start postgres container using ``` docker run --rm -P -p 127.0.0.1:5432:5432 -e POSTGRES_PASSWORD="1234" --name pg postgres:alpine ```

Connect to postgres from the host 
```psql postgresql://postgres:1234@localhost:5432/postgres```

or from the container 

```
docker exec -it pg bash
psql postgresql://postgres:1234@localhost:5432/postgres
```
Import sample table and import users from ./postgres/l81_users.sql

## Phantom read problem 

*Postgres and percona behaves the same*

We have a table of 10 users (user_id from 1 to 10). 
Insert a user with user_id = 13 to create a gap between user ids. 

```mysql
INSERT INTO Users (user_id, first_name, last_name, email, address, dob)
VALUES (13, 'Destinee', 'Roob', 'ikerluke@yahoo.com', '35750 Gerry Pass\nPort Gabriellatown, OH 77731-3343', '2006-04-10')
```

Set session transaction isolation level to Read Commited and exeute a select query to get users with user_id > 8 
```mysql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT * FROM Users WHERE user_id > 8;
```
![Phantom rows](./doc/phantom_rows_1.png "Phantom rows")

In another transaction insert a user with user_id = 11 to insert it tino the gap. 

```mysql
INSERT INTO Users (user_id, first_name, last_name, email, address, dob)
VALUES (11, 'Dawn', 'Boehm', 'eunice.lang@wilderman.com', '891 Sheridan Highway\nWest Vedamouth, IL 73154', '2012-01-29')
```

Execute selection inside a transaction once again ```SELECT * FROM Users WHERE user_id > 8;```. You'll see different result. 

![Phantom rows](./doc/phantom_rows_2.png "Phantom rows")

So the isolation principle is violated. 

## Non-repeatable read problem 

*Percona and postgres behaviour are the same. *

Start transaction with Read Commited isolation level, select user with user_id = 2

```mysql
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT * FROM Users WHERE user_id = 2;
```
![nr read](./doc/non-repeatable_read_1.png "nr read")

In other transaction update the user's first_name. 

```mysql
UPDATE users SET first_name = 'Ashley' WHERE user_id = 2;
```
Execute the select query from the first transaction once again

```mysql
SELECT * FROM Users WHERE user_id = 2;
```
![nr read](./doc/non-repeatable_read_2.png "nr read")

As you can see the first name changed. So we have a non-repeatable read problem. This doesn't happenwhen using locking read ```SELECT * ... FOR UPDATE``` or REPEATABLE READ isolation level. 

## Dirty read problem 

*Postgres doesn't implement READ UNCOMMITTED isolation level so the uncommitted changes didn't affect transaction results.* 

Start transaction with READ UNCOMMITTED isolation level. Get user with user_id = 3

```mysql
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT * FROM Users WHERE user_id = 3;
```

![dirty read](./doc/dirty_read_1.png "dirty read")

Update user first_name in other transaction but do not commit

```mysql
START TRANSACTION;
UPDATE users SET first_name = 'Arlo' WHERE user_id = 3;
```

Repeat select query from the first transaction

```mysql
SELECT * FROM Users WHERE user_id = 3;
```
![dirty read](./doc/dirty_read_2.png "dirty read")

We can see an updated first_name but the second transaction wasn't commited yet.
This is a dirty read problem. 

## Lost update problem

*Problem can be seen in MySQL but doesn't appear in same conditions in Postgres*

Start transacton with REPEATABLE READ isolation level. Select users with dob > 1990-01-01. We have 8 users in result. 

```mysql
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT * FROM Users WHERE dob > DATE('1990-01-01');
```

![lost update](./doc/lost_update_1.png "lost update")

In another transaction update dob for some users. So we moved 4 users out from the "younger than 1990" group. 

```mysql
UPDATE users SET dob = '1982-10-07' WHERE user_id = 1;
UPDATE users SET dob = '1970-01-11' WHERE user_id = 2;
UPDATE users SET dob = '1967-10-10' WHERE user_id = 6;
UPDATE users SET dob = '1989-08-11' WHERE user_id = 8;
```

If we repeat the select query from the first trasaction we'll see the same result (8 users). So we have a reads-isolated snapshot. 

However if we execute UPDATE query

```mysql
UPDATE Users SET last_name = '' WHERE dob > DATE('1990-01-01');
```

![lost update](./doc/lost_update_2.png "lost update")

We have 4 processsed rows, as UPADATE operation is executed outside the snapshot. 
This is a lost update problem that can cause issues if update operations are affected by the SELECT query results. 