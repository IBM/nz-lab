# Data Distribution

Netezza Performance Server is a family of data-warehousing appliances
that combine high performance with low administrative effort. Due to the
unique data warehousing centric architecture of Netezza Performance
Server, most performance tuning tasks are either not necessary or
automated. Unlike normal data warehousing solutions, no tablespaces need
to be created or tuned, there are also no indexes, buffer pools or
partitions.

Since Netezza Performance Server is built on a massively parallel
architecture that distributes data and workloads over many processing
and data nodes, the single most important tuning factor is picking the
right distribution key. The distribution key governs which data rows of
a table are distributed to which data slice and it is very important to
pick an optimal distribution key to avoid data skew, processing skew and
to make joins co-located whenever possible.

## 1 Objectives

In this lab we will cover a typical scenario in a POC or customer
engagement which involves an existing data warehouse for customer
transactions.

![A close up of a map Description automatically
generated](./nz-images/nz-03-Data-Distribution/media/image5.png)

***Figure 1. LABDB database***

Figure 1 shows a visualization of the tables in the data warehouse and
the relationships between the tables. The warehouse contains the
customers of the company, their orders, and the line items that are part
of the order. The warehouse also has a list of suppliers, providing the
parts that are part of the shipped line items.

For this lab we already have the DDLs for creation of the tables and
load files containing the warehouse data. Both are already in a format
usable by the Netezza Performance Server System. In this lab we will
define the distribution keys for these tables.

In addition to the data and the DDLs we also have received a couple of
queries from the customer that are usually run against the warehouse.
Those are important input as well for picking optimal distribution keys.

## 2 Check the Number of Data Slices

By default, the NPS VirtualBox virtual machine only comes with 1 data
slice. To perform the steps in this lab and the other performance
related labs you will need 4 data slices. It is not recommended to run
more than 4 data slices.

If you did not configure your virtual machine for 4 data slices, go back
to the lab called:
**01-Setup-NPS-Virtual-Machine-Setup-Lab-Guide-Win10-OSX-Final** and
follow the section called: "**Increase the Number of Data Slices**".

1.  To check your data slices, login to NPS Command Line using one of
    these two methods.

    a.  Login to the VM directly and use the terminal application
        available inside the VM.

    b.  Connect to your Netezza Performance Server image using a
        terminal application (Windows PowerShell, PuTTY, Mac OSX
        Terminal)

2.  If you are continuing from the previous lab and are already
    connected to `nzsql`, quit the `nzsql` console with the `\q`
    command.
    
	=== "Input"
	
		``` Input
		nzstate # if NPS is offline, run nzstart
		nzhw
		nzds
		```
		
	=== "Output"
	
		``` Output
		[nz@localhost labs]$ nzstate
		System state is 'Online'.
		[nz@localhost labs]$ nzhw
		Description HW ID Location Role State Security
		----------- ----- ---------- ------ ------ --------
		Rack 1001 rack1 Active Ok N/A
		SPA 1002 spa1 Active Ok N/A
		SPU 1003 spa1.spu1 Active Online N/A
		Disk 1004 spa1.disk1 Active Ok N/A
		Disk 1005 spa1.disk2 Active Ok N/A
		Disk 1006 spa1.disk3 Active Ok N/A
		Disk 1007 spa1.disk4 Active Ok N/A
		
		[nz@localhost labs]$ nzds
		Data Slice Status  SPU  Partition Size (GiB) % Used Supporting Disks
		---------- ------- ---- --------- ---------- ------ ----------------
		1          Unknown 1003 0         16         0.84   1004
		2          Unknown 1003 1         16         0.84   1005
		3          Unknown 1003 2         16         0.84   1006
		4          Unknown 1003 3         16         0.84   1007
		```    
	
	With the proper data slices continue to the next section.

## 3 Lab Setup

This lab uses an initial setup script to make sure the correct user and
database exist for the remainder of the lab. Follow the instructions
below to run the setup script.

1.  Login to NPS Command Line using one of these two methods.

    a.  Login to the VM directly and use the terminal application
        available inside the VM.

    b.  Connect to your Netezza Performance Server image using a
        terminal application (Windows PowerShell, PuTTY, Mac OSX
        Terminal)

2.  If you are continuing from the previous lab and are already
    connected to `nzsql` quit the `nzsql` console with the `\q`
    command.

3.  Prepare for this lab by running the setup script. To do this use the
    following two commands:

	=== "Input"
	
	    ```
	    cd ~/labs/dataDistribution/setupLab
	    ./setupLab.sh
	    ```
	
	=== "Output"
		
		```
		ERROR: DROP DATABASE: object LABDB does not exist.
		CREATE DATABASE
		ERROR: CREATE USER: object LABADMIN already exists as a USER.
		ALTER USER
		ALTER DATABASE
		```
	    
	The error message at the beginning is expected since the script tries to clean up existing `LINEITEM` tables.
	
	This lab is now setup to use for the remainder of the sections.
	
## 4 Skew

Tables in Netezza Performance Server are distributed across data slices
based on the distribution method and key. If a bad data distribution
method has been picked, it will result in skewed tables or processing
skew. Data skew occurs when the distribution method puts significantly
more records of a table on one data slice than on other data slices.
Apart from bad performance this also results in a situation where the
Netezza Performance Server can hold significantly less data than
expected.

Processing skew occurs if processing of queries is mainly taking place
on some data slices for example because queries only apply to data on
those data slices. Both types of skew result in suboptimal performance
since in a parallel system the slowest node defines the total execution
time.

### 4.1 Data Skew

The first table we will create is `LINEITEM`, the main fact table of the
schema. It contains roughly 6 million rows.

1.  To create the `LINEITEM` table, switch to the lab directory
	    `~/labs/dataDistribution`. To do this use the following command:
	    (Notice that you can use bash auto complete by using the Tab key to
	    complete folder and files names)
	
	=== "Input"
	
	    ```
	    cd ~/labs/dataDistribution
	    ls -l
	    ```
	
	=== "Output"
	
		``` 
		[nz@localhost dataDistribution]$ ls -l
		total 24
		-rwxr-xr-x. 1 nz nz 177 Apr 20 2020 create_lineitem_1.sh
		-rwxr-xr-x. 1 nz nz 170 Dec 9 2015 create_orders_1.sh
		-rwxr-xr-x. 1 nz nz 612 Dec 9 2015 create_remaining.sh
		-rwxr-xr-x. 1 nz nz 656 Mar 29 13:44 lineitem.sql
		-rwxr-xr-x. 1 nz nz 374 Mar 29 13:44 orders.sql
		-rwxr-xr-x. 1 nz nz 1660 Mar 29 13:44 remaining_tables.sql
		drwxr-xr-x. 2 nz nz 91 Mar 29 2020 setupLab
		```
	
	The command line prompt changes to reflect the directory you are in
	(`dataDistribution`).
	
2.  Create the `LINEITEM` table by using the following script. Since the
	    fact table is quite large this can take several minutes.
	
	=== "Input"
	
	    ```
	    ./create_lineitem_1.sh
	    ```
	
	=== "Output"
	
		```
		ERROR: relation does not exist LABDB.ADMIN.LINEITEM
		CREATE TABLE
		Load session of table 'LINEITEM' completed successfully
		```
	
	The error message at the beginning is expected since the script tries
	to clean up existing `LINEITEM` tables. Also note that a load log file
	has been created in the `~/labs/dataDistribution` directory named
	`LINEITEM.ADMIN.LABDB.nzlog`. (`ls \*.nzlog`) Review the load
	logfile as time and interest permits.
	
	This load will take some time, if you want to see the status of the
	load run the following SQL from another terminal session.
	
	=== "Input"
	
	    ```
	    nzsql -c "select * from _v_load_status;"
	    ```
	
	=== "Output"
	
		```
		PLANID | DATABASENAME | TABLENAME | SCHEMANAME | USERNAME | BYTESPROCESSED | ROWSINSERTED | ROWSREJECTED | BYTESDOWNLOADED
		-------+--------------+-----------+------------+----------+----------------+--------------+--------------+-----------------
		592    | LABDB        | LINEITEM  | ADMIN      | LABADMIN | 704643063      | 5522182      | 0            | 718896512
		(1 row)
		```
	
3.  Now let's have a look at the created table, open the `nzsql` console
	    by entering the command:
	
	=== "Input"
	
	    ```
	    nzsql
	    ```
	
	=== "Output"
	
		```
		Welcome to nzsql, the IBM Netezza SQL interactive terminal.
		
		Type:  \h for help with SQL commands
		       \? for help on internal slash commands
		       \g or terminate with semicolon to execute query
		       \q to quit
		
		SYSTEM.ADMIN(ADMIN)=>
		
		```
	
4.  Connect to the database `LABDB` as user `LABADMIN` by typing the
	    following command:
	
	=== "Input"
	
	    ```
	    \c LABDB LABADMIN
	    ```
	
	=== "Output"
	
		```
		You are now connected to database LABDB as user LABADMIM
		LABDB.ADMIN(LABADMIN)=>
		```
	
	You should now be connected to the `LABDB` database as the `LABADMIN` user.
	
5.  Let's have a look at the table we just created. First, we want to
	    see a description of its columns and distribution key. Use the `NZSQL`
	    describe command `\d LINEITEM` to get a description of the table.
	
	=== "Input"
	
	    ```
	    \d LINEITEM
	    ```
	
	=== "Output"
	
		```
		                        Table "LINEITEM"
		    Attribute    |         Type          | Modifier | Default Value
		-----------------+-----------------------+----------+---------------
		 L_ORDERKEY      | INTEGER               | NOT NULL |
		 L_PARTKEY       | INTEGER               | NOT NULL |
		 L_SUPPKEY       | INTEGER               | NOT NULL |
		 L_LINENUMBER    | INTEGER               | NOT NULL |
		 L_QUANTITY      | NUMERIC(15,2)         | NOT NULL |
		 L_EXTENDEDPRICE | NUMERIC(15,2)         | NOT NULL |
		 L_DISCOUNT      | NUMERIC(15,2)         | NOT NULL |
		 L_TAX           | NUMERIC(15,2)         | NOT NULL |
		 L_RETURNFLAG    | CHARACTER(1)          | NOT NULL |
		 L_LINESTATUS    | CHARACTER(1)          | NOT NULL |
		 L_SHIPDATE      | DATE                  | NOT NULL |
		 L_COMMITDATE    | DATE                  | NOT NULL |
		 L_RECEIPTDATE   | DATE                  | NOT NULL |
		 L_SHIPINSTRUCT  | CHARACTER(25)         | NOT NULL |
		 L_SHIPMODE      | CHARACTER(10)         | NOT NULL |
		 L_COMMENT       | CHARACTER VARYING(44) | NOT NULL |
		Distributed on hash: "L_LINESTATUS"
		```
	
	We can see that the `LINEITEM` table has 16 columns with different data
	types. Some of the columns have a "key" suffix and substrings
	containing the names of other tables and are most likely foreign keys
	of dimension tables. The distribution key is `L_LINESTATUS`, which is of
	a `CHAR(1)` data type.
	
6.  Now let's have a look at the data in the table. To return a limited
	    number of rows you can use the limit keyword in your select queries.
	    Execute the following select command to return 10 rows of the
	    `LINEITEM` table. For readability we only select a couple of columns
	    including the order key (`L_ORDERKEY`), the ship date (`L_SHIPDATE`)
	    and the line status (`L_LINESTATUS`) distribution key:
	
	=== "Input"
	
	    ```
	    SELECT L_ORDERKEY, L_QUANTITY, L_SHIPDATE, L_LINESTATUS FROM LINEITEM LIMIT 10;
	    ```
	
	=== "Output"
	
		```
		 L_ORDERKEY | L_QUANTITY | L_SHIPDATE | L_LINESTATUS
		------------+------------+------------+--------------
		          3 |      45.00 | 1994-02-02 | F
		          3 |      49.00 | 1993-11-09 | F
		          3 |      27.00 | 1994-01-16 | F
		          3 |       2.00 | 1993-12-04 | F
		          3 |      28.00 | 1993-12-14 | F
		          3 |      26.00 | 1993-10-29 | F
		          5 |      15.00 | 1994-10-31 | F
		          5 |      26.00 | 1994-10-16 | F
		          5 |      50.00 | 1994-08-08 | F
		          6 |      37.00 | 1992-04-27 | F
		(10 rows)
		```
		
	From this limited sample we cannot make any definite judgement, but we
	can make a couple of assumptions. While the `L_ORDERKEY` column is not
	unique, it seems to have a lot of distinct values. The `L_SHIPDATE`
	column also appears to have a lot of distinct shipping date values. On
	the other hand, our current distribution key `L_LINESTATUS` has only two
	values, which may make it a bad distribution key. It is possible that
	you get different results. Since a database table is an unordered set,
	it is probable that you get different results. For example, only O or
	F values may be in the `L_LINESTATUS` column.
	
7.  We will now verify the number of distinct values in the `L_LINESTATUS`
	    column with a `SELECT DISTINCT` statement. To return a list of all
	    distinct values that are in the `L_LINESTATUS` column execute the
	    following SQL command:
	
	=== "Input"
	
	    ```
	    SELECT DISTINCT L_LINESTATUS FROM LINEITEM;
	    ```
	
	=== "Output"
	
		```
		L_LINESTATUS
		--------------
		 O
		 F
		(2 rows)
		```
	
	We can see that the `L_LINESTATUS` column only contains two distinct
	values. As a distribution key, this will result in a table that is
	only distributed to two of the available data slices.
	
8.  We verify this by executing the following SQL call, which will
	    return a list of all data slices which contain rows of the `LINEITEM`
	    table, and the corresponding number of rows stored in them:
	
	=== "Input"
	
	    ```
	    SELECT DATASLICEID, COUNT(*) FROM LINEITEM GROUP BY DATASLICEID;
	    ```
	
	=== "Output"
	
		```
		 DATASLICEID |  COUNT
		-------------+---------
		           4 | 2996217
		           1 | 3004998
		(2 rows)
		```
	
	Every Netezza Performance Server table has a hidden column
	`DATASLICEID`, which contains the id of the data slice the selected row
	is being stored in. By executing a SQL query that does a GROUP BY on
	this column and counts the number of rows for each `DATASLICEID`, data
	skew can be detected.
	
	In this case the table has been distributed to only two of the available
	four data slices. This means that only half of the available space and
	compute is used, which will result in low performance during most query
	executions. In general, a good distribution key should have many unique
	values, with an even distribution of rows in the data slices. Columns
	with few unique values, especially boolean columns or date columns,
	should not be considered as distribution keys.
	
### 4.2 Processing Skew

Even in tables that are distributed evenly across data slices, data
processing for queries can be concentrated or skewed to a limited number
of data slices. One way for processing skew to occur for a query is if
there is a predicate (`WHERE` condition) on the distribution key, and the
distribution key has few unique values. Using a DATE columns for a
distribution key is notorious for causing processing skew, and usually
should be avoided. The next section of the lab will show this effect.
	
1.  First, we will pick a new distribution key. As we have seen it
	    should have many unique (distinct) values. One of the columns that
	    did fit this description was the `L_SHIPDATE` column. Check the number
	    of distinct values in the `L_SHIPDATE` column with the `COUNT(DISTINCT
	    )` statement:
	
	=== "Input"
	
	    ```
	    SELECT COUNT(DISTINCT L_SHIPDATE) FROM LINEITEM;
	    ```
	
	=== "Output"
	
		```
		COUNT
		-------
		  2526
		(1 row)
		```
		
	The column has over 2500 distinct values and therefore has more than
	enough unique values to guarantee a good data distribution on 4 data
	slices.
	
2.  Now let's reload the `LINEITEM` table with the new distribution key.
	    For this we need to change the SQL of the load script we executed at
	    the beginning of the lab. Exit the `nzsql` console by entering:
	    `\q`.
	
3.  You should now be in the lab directory `~/labs/dataDistribution`. The
	    table creation statement is in the lineitem.sql file. We will need
	    to make changes to the file with a text editor. Open the file with
	    the default Linux text editor vi. To do this enter the following
	    command:
	
	=== "Input"
	    ```
	    vi lineitem.sql
	    ```
	
4.  The vi editor has two modes, a command mode used to save files, quit
	    the editor etc. and an insert mode. Initially you will be in the
	    command mode. To change the lineitem.sql file you need to switch to
	    the insert mode by pressing "i". The editor will show an `-- INSERT
	    --` at the bottom of the screen. Use the delete key to remove
	    `l_linestatus` and type `l_shipdate`.
	
	!!! note
		If you are not comfortable with vi you can used `gedit` from the virtual machine desktop.
	
	=== "Input"
	    ```
	    gedit lineitem.sql
	    ```
	
5.  You can now use the cursor keys to navigate to the `DISTRIBUTE ON`
	    clause at the bottom of the create command. Change the distribution
	    key to `L_SHIPDATE`. The editor should now look like the following:
	
	=== "Output"
	    ```
	    create table lineitem
	      (
	        l_orderkey integer not null ,
	        l_partkey integer not null ,
	        l_suppkey integer not null ,
	        l_linenumber integer not null ,
	        l_quantity decimal(15,2) not null ,
	        l_extendedprice decimal(15,2) not null ,
	        l_discount decimal(15,2) not null ,
	        l_tax decimal(15,2) not null ,
	        l_returnflag char(1) not null ,
	        l_linestatus char(1) not null ,
	        l_shipdate date not null ,
	        l_commitdate date not null ,
	        l_receiptdate date not null ,
	        l_shipinstruct char(25) not null ,
	        l_shipmode char(10) not null ,
	        l_comment varchar(44) not null
	      )
	    DISTRIBUTE ON (l_shipdate);
	    ~
	    ~
	    -- INSERT --   
	    ```
	
6.  We will now save our changes. Press `Esc` to switch back
	    into command mode. You should see that the `-- INSERT --` string at
	    the bottom of the screen vanishes. Enter `:wq!` and press
	    enter to write the file and quit the editor without any questions.
	    If you made a mistake editing and would like to undo it press `Esc`
	    then enter `:q!` and go back to step 3.
	
7.  Now repeat steps 3-5 of section 2.1 Data Skew:
	
    a.  Recreate and load the `LINEITEM` table with your new distribution
        key by executing the `./create_lineitem_1.sh` command

    b.  Use the `nzsql` command to enter the command console

    c.  Switch to the `LABDB` database by using the `\c LABDB
        LABADMIN` command.

    d.  Validate the distribution key is `L_SHIPDATE` using the `\d
        LINEITEM` command.

8.  Now we verify that the new distribution key results in a good data
	    distribution. For this we will repeat the query, which returns the
	    number of rows for each datasliceid of the `LINEITEM` table. Execute
	    the following command:
	
	=== "Input"
	
	    ```
	    SELECT DATASLICEID, COUNT(*) FROM LINEITEM GROUP BY DATASLICEID ORDER BY 1;
	    ```
	
	=== "Output"
	
		```
		DATASLICEID |  COUNT
		-------------+---------
		           1 | 1499990
		           2 | 1497649
		           3 | 1501760
		           4 | 1501816
		(4 rows)
		```
		
	We can see that the data distribution is much better now. All four
	data slices have a roughly equal number of rows.
	
9.  Now that we have a database table with a good data distribution
	    let's look at a couple of queries. The following query is executed
	    regularly by the end user. It returns the average quantity shipped
	    on a given day grouped by the shipping mode. Execute the following
	    query:
	
	=== "Input"
	
	    ```
	    SELECT AVG(L_QUANTITY) AS AVG_Q, L_SHIPMODE 
	    FROM LINEITEM 
	    WHERE L_SHIPDATE = '1996-03-29' 
	    GROUP BY L_SHIPMODE;
	    ```
	
	=== "Output"
	
		```
		   AVG_Q   | L_SHIPMODE
		-----------+------------
		 26.045455 | MAIL
		 24.494186 | REG AIR
		 25.562500 | SHIP
		 24.780282 | RAIL
		 27.147826 | TRUCK
		 25.708556 | AIR
		 26.038567 | FOB
		(7 rows)
		```
		
	This query will take all rows from the 29th March of 1996 and compute
	the average value of the `L_QUANTITY` column for each `L_SHIPMODE` value. It
	is a typical warehousing query insofar as a date column is used to
	restrict the row set that is taken as input for computation.
	
	In this example most rows of the `LINEITEM` table will be filtered away,
	only rows that have the specified date will be used as input for
	computation of the AVG aggregation.
	
	Execute the following SQL statement to see on which data slice we can
	find the rows from the 29th March of 1996:
	
	=== "Input"
	
	    ```
	    SELECT COUNT(\*), DATASLICEID 
	    FROM LINEITEM
	    WHERE L_SHIPDATE = '1996-03-29' 
	    GROUP BY DATASLICEID;
	    ```
	
	=== "Output"
	
		```
		 COUNT | DATASLICEID
		-------+-------------
		  2501 |           2
		(1 row)
		```
	
	Since we used the shipping date column as a distribution key, all rows
	from a specific date can be found on one data slice and therefore also
	one SPU. This means that for our previous query all rows on other data
	slices are unused and the computation takes place only on one data
	slice and SPU. This is known as processing skew. While this one SPU is
	working the other SPUs will be idle.
	
	Columns that are often used in `WHERE` conditions should not be used as
	distribution keys, since this can easily result in processing skew. In
	warehousing environments this is especially true for DATE columns.
	
	Good distribution keys are key columns (join columns); they have lots of
	unique values and rarely result in processing skew. In our example we
	have a couple of distribution keys to choose from: `L_SUPPKEY`,
	`L_ORDERKEY`, `L_PARTKEY`. All columns have many distinct values.

## 5 Co-Location

The most basic warehouse schema consists of a fact table containing a
list of all business transactions and a set of dimension tables that
contain the different actors, objects, locations and time points that
have taken part in these transactions. This means that most queries will
not only access one database table but will require joins between many
tables.

In Netezza Performance Server tables are distributed over a large
numbers of data slices on different SPUs. This means that during a join
of two tables there are two possibilities.

-   Rows of the two tables that are joined together are on the same data
    slice, which means the rows are co-located and can be joined
    locally.

-   Rows of the two tables that are joined together are on different
    data slices, which means that rows need to be redistributed (moved)
    so that rows being joined are on the same data slice.

### 5.1 Investigation

Co-location has big performance advantages. In the following section we
will demonstrate this by introducing a second table called `ORDERS`.
	
1.  Switch to the Linux command line, if you are in the NZSQL console.
	    Do this with the `\q` command.
	
2.  Switch to the data distribution lab directory with the `cd
	    ~/labs/dataDistribution`. command, create and load the
	    ORDERS table by executing the `./create_orders_1.sh` script
	
	=== "Input"
	
	    ```
	    cd ~/labs/dataDistribution/
	    ./create_orders_1.sh
	    ```
	
	=== "Output"
	
		```
		ERROR:  relation does not exist LABDB.ADMIN.ORDERS
		CREATE TABLE
		Load session of table 'ORDERS' completed successfully
		```
	
3.  Enter the NZSQL console with the `nzsql labdb labadmin`
	    command and take a look at the `ORDERS` table with the `\d
	    orders` command.
	
	=== "Input"
	
	    ```
	    nzsql labdb
	    LABDB.ADMIN(LABADMIN)=> \d orders
	    ```
	
	=== "Output"
	
		```
		                           Table "ORDERS"
		    Attribute    |         Type          | Modifier | Default Value
		-----------------+-----------------------+----------+---------------
		 O_ORDERKEY      | INTEGER               | NOT NULL |
		 O_CUSTKEY       | INTEGER               | NOT NULL |
		 O_ORDERSTATUS   | CHARACTER(1)          | NOT NULL |
		 O_TOTALPRICE    | NUMERIC(15,2)         | NOT NULL |
		 O_ORDERDATE     | DATE                  | NOT NULL |
		 O_ORDERPRIORITY | CHARACTER(15)         | NOT NULL |
		 O_CLERK         | CHARACTER(15)         | NOT NULL |
		 O_SHIPPRIORITY  | INTEGER               | NOT NULL |
		 O_COMMENT       | CHARACTER VARYING(79) | NOT NULL |
		Distributed on random: (round-robin)
		```
	
	The `ORDERS` table has a key column `O_ORDERKEY` that is the primary key of
	the table. The `ORDERS` table contains information on the order value,
	priority and date, and has been distributed on random. This means that
	Netezza Performance Server doesn't use a hash-based algorithm to
	distribute the data. Instead, rows are distributed randomly on the
	available data slices in a round-robin fashion.
	
4.  You can check the data distribution of the table, using the methods
	    we have used before for the LINEITEM table. The data distribution
	    will be perfect. For example, try the following:
	
	=== "Input"
	
	    ```
	    SELECT COUNT(*), DATASLICEID 
	    FROM ORDERS
	    GROUP BY DATASLICEID 
	    ORDER BY 1;
	    ```
	
	=== "Output"
	
		```
		COUNT  | DATASLICEID
		--------+-------------
		 374393 |           3
		 374518 |           1
		 375517 |           2
		 375572 |           4
		(4 rows)
		```
	
	There will also not be any processing skew for queries on the single
	table, since in a random distribution there can be no correlation
	between any WHERE condition and the distribution key.
	
5.  We have received another typical query from our customer. It returns
	    the average total price and item quantity of all orders grouped by
	    the shipping priority. This query has to join together the `LINEITEM`
	    and ORDERS tables to get the total order cost from the orders table
	    and the quantity for each shipped item from the `LINEITEM` table. The
	    tables are joined with an inner join on the `L_ORDERKEY` column.
	    Execute the following query and note the approximate execution time:
	
	=== "Input"
	
	    ```
	    \time
	    SELECT AVG(O.O_TOTALPRICE) AS PRICE,
	           AVG(L.L_QUANTITY) AS QUANTITY, 
	           O.O_ORDERPRIORITY 
	    FROM ORDERS AS O, LINEITEM AS L 
	    WHERE L.L_ORDERKEY=O.O_ORDERKEY 
	    GROUP BY O_ORDERPRIORITY;
	    ```
	
	=== "Output"
	
		```
		     PRICE     | QUANTITY  | O_ORDERPRIORITY
		---------------+-----------+-----------------
		 189219.594349 | 25.532474 | 5-LOW
		 189285.029553 | 25.526186 | 2-HIGH
		 188546.457203 | 25.472923 | 4-NOT SPECIFIED
		 189026.093657 | 25.494518 | 3-MEDIUM
		 189093.608965 | 25.513563 | 1-URGENT
		(5 rows)
		Elapsed time: 0m4.293s
		```
	
	Notice that the query takes about 4 seconds to complete on our
	machine. The actual execution times on your machine will be different.
	
6.  Remember that the `ORDERS` table was distributed randomly and the
	    `LINEITEM` table is still distributed by the `L_SHIPDATE` column. The
	    join on the other hand is taking place on the `L_ORDERKEY` and
	    `O_ORDERKEY` columns. We will now have a quick look at what is
	    happening inside the Netezza Performance Server system in this
	    scenario. To do this we use the Netezza Performance Server `EXPLAIN`
	    function. This will be more thoroughly covered in the Optimization
	    lab.
	
	Execute the following command:
	 
	=== "Input"
	
	    ```
	    EXPLAIN VERBOSE 
	    SELECT AVG(O.O_TOTALPRICE) AS PRICE,
	           AVG(L.L_QUANTITY) AS QUANTITY, 
	           O.O_ORDERPRIORITY 
	    FROM ORDERS AS O, LINEITEM AS L 
	    WHERE L.L_ORDERKEY=O.O_ORDERKEY 
	    GROUP BY O_ORDERPRIORITY;    
	    ```
	
	You will get a long output. Scroll up until you see your command in the
	text window. The start of the `EXPLAIN` output should look like the
	following:
	 
	=== "Output"
	
		```
		EXPLAIN VERBOSE SELECT AVG(O.O_TOTALPRICE) AS PRICE, AVG(L.L_QUANTITY) AS QUANTITY, O_ORDERPRIORITY FROM ORDERS AS O, LINEITEM AS L WHERE L.L_ORDERKEY=O.O_ORDERKEY GROUP BY O_ORDERPRIORITY;
		
		QUERY VERBOSE PLAN:
		
		Node 1.
		  [SPU Sequential Scan table "ORDERS" as "O" {}]
		      -- Estimated Rows = 1500000, Width = 27, Cost = 0.0 .. 1653.2, Conf = 100.0
		      Projections:
		        1:O.O_TOTALPRICE  2:O.O_ORDERPRIORITY  3:O.O_ORDERKEY
		  [SPU Distribute on {(O.O_ORDERKEY)}]
		  [HashIt for Join]
		Node 2.
		  [SPU Sequential Scan table "LINEITEM" as "L" {(L.L_SHIPDATE)}]
		      -- Estimated Rows = 6001215, Width = 12, Cost = 0.0 .. 6907.1, Conf = 100.0
		      Projections:
		        1:L.L_QUANTITY  2:L.L_ORDERKEY
		  [SPU Distribute on {(L.L_ORDERKEY)}]
		Node 3.
		  [SPU Hash Join Stream "Node 2" with Temp "Node 1" {(O.O_ORDERKEY,L.L_ORDERKEY)}]
		      -- Estimated Rows = 90018225000, Width = 31, Cost = 1653.2 .. 928769.9, Conf = 80.0
		      Restrictions:
		        (L.L_ORDERKEY = O.O_ORDERKEY)
		      Projections:
		        1:O.O_TOTALPRICE  2:L.L_QUANTITY  3:O.O_ORDERPRIORITY
		  [SPU Fabric Join]
		
		..<Rest of EXPLAIN Plan>..
		
		```
	
	The `EXPLAIN` functionality will be covered in detail in a following
	chapter, but it is easy to see what is happening here. What's happening
	is the system is redistributing both the `ORDERS` and `LINEITEM` tables so
	that the rows can be joined together. This is very bad because both
	tables are of significant size so there is a considerable overhead. This
	inefficient double redistribution occurs because neither of the tables
	are distributed on the join key. In the next section we will fix this.

### 5.2 Co-Located Joins

In the last section we have seen that a query using joins can result in
costly data redistribution during join execution when the joined tables
are not distributed on the join key. In this section we will reload the
tables based on the mutual join key to enhance performance during joins.
	
1.  Exit the `NZSQL` console with the `\q` command.
	
2.  If not already in the `dataDistribution` directory run the command `cd
	    ~/labs/dataDistribution` to enter the directory.
	
3.  Change the distribution key in the lineitem.sql file to `L_ORDERKEY`:
	
    a.  Open the file with the vi editor (or gedit from the VM desktop)
        by executing the command: `vi lineitem.sql` (`gedit
        lineitem.sql`).

    b.  In vi switch to `INSERT` mode by pressing "i"

    c.  Navigate with the cursor keys to the `DISTRIBUTE ON` clause and
        change it to `DISTRIBUTE ON (L_ORDERKEY)`

    d.  Exit the `INSERT` mode by pressing `ESC`

    e.  Enter `:wq!` In the command line of the vi editor and press enter.
        Before pressing enter your screen should look like the
        following:

	=== "Output"
	    ```
	    create table lineitem
	      (
	        l_orderkey integer not null ,
	        l_partkey integer not null ,
	        l_suppkey integer not null ,
	        l_linenumber integer not null ,
	        l_quantity decimal(15,2) not null ,
	        l_extendedprice decimal(15,2) not null ,
	        l_discount decimal(15,2) not null ,
	        l_tax decimal(15,2) not null ,
	        l_returnflag char(1) not null ,
	        l_linestatus char(1) not null ,
	        l_shipdate date not null ,
	        l_commitdate date not null ,
	        l_receiptdate date not null ,
	        l_shipinstruct char(25) not null ,
	        l_shipmode char(10) not null ,
	        l_comment varchar(44) not null
	      )
	    DISTRIBUTE ON (l_orderkey);
	    ```
	
4.  Change the Distribution key in the `orders.sql` file to `O_ORDERKEY`.
	
    a.  Open the file with the vi editor by executing the command: `vi
        orders.sql`

    b.  Switch to `INSERT` mode by pressing "i"

    c.  Navigate with the cursor keys to the `DISTRIBUTE ON` clause and
        change it to `DISTRIBUTE ON (O_ORDERKEY)`

    d.  Exit the `INSERT` mode by pressing `ESC`

    e.  Enter `:wq!` In the command line of the VI editor and
        Press Enter. Before pressing Enter your screen should look like
        the following:

	=== "Output"
	    ```
	    create table orders
	      (
	        o_orderkey integer not null ,
	        o_custkey integer not null ,
	        o_orderstatus char(1) not null ,
	        o_totalprice decimal(15,2) not null ,
	        o_orderdate date not null ,
	        o_orderpriority char(15) not null ,
	        o_clerk char(15) not null ,
	        o_shippriority integer not null ,
	        o_comment varchar(79) not null
	      )
	    DISTRIBUTE ON (O_ORDERKEY);
	    ```
	
5.  Recreate and load the `LINEITEM` table with the distribution key
	    `L_ORDERKEY` by executing the `./create_lineitem_1.sh` script.
	
	=== "Input"
	
	    ```
	    cd ~/labs/dataDistribution/
	    ./create_lineitem_1.sh
	    ```
	
	=== "Output"
	
		```
		DROP TABLE
		CREATE TABLE
		Load session of table 'LINEITEM' completed successfully
		```
	
6.  Recreate and load the `ORDERS` table with the distribution key
	    `O_ORDERKEY` by executing the `./create_orders_1.sh` script.
	
	=== "Input"
	
	    ```
	    ./create_orders_1.sh
	    ```
	
	=== "Output"
	
		```
		DROP TABLE
		CREATE TABLE
		Load session of table 'ORDERS' completed successfully
		```
	
7.  Enter the `NZSQL` console by executing the `nzsql labdb
	    labadmin` command and check the distribution key using the
	    `\d lineitem` and `\d orders` commands.
	    
	=== "Display Lineitem"
	
	    ```
	    nzsql labdb labadmin
	    \d lineitem
	    ```
	
	=== "Lineitem Details"
	
		```
		                          Table "LINEITEM"
		    Attribute    |         Type          | Modifier | Default Value
		-----------------+-----------------------+----------+---------------
		 L_ORDERKEY      | INTEGER               | NOT NULL |
		 L_PARTKEY       | INTEGER               | NOT NULL |
		 L_SUPPKEY       | INTEGER               | NOT NULL |
		 L_LINENUMBER    | INTEGER               | NOT NULL |
		 L_QUANTITY      | NUMERIC(15,2)         | NOT NULL |
		 L_EXTENDEDPRICE | NUMERIC(15,2)         | NOT NULL |
		 L_DISCOUNT      | NUMERIC(15,2)         | NOT NULL |
		 L_TAX           | NUMERIC(15,2)         | NOT NULL |
		 L_RETURNFLAG    | CHARACTER(1)          | NOT NULL |
		 L_LINESTATUS    | CHARACTER(1)          | NOT NULL |
		 L_SHIPDATE      | DATE                  | NOT NULL |
		 L_COMMITDATE    | DATE                  | NOT NULL |
		 L_RECEIPTDATE   | DATE                  | NOT NULL |
		 L_SHIPINSTRUCT  | CHARACTER(25)         | NOT NULL |
		 L_SHIPMODE      | CHARACTER(10)         | NOT NULL |
		 L_COMMENT       | CHARACTER VARYING(44) | NOT NULL |
		Distributed on hash: "L_ORDERKEY"
		```
	
	=== "Display Orders"
	
	    ```
	    nzsql labdb labadmin
	    \d orders
	    ```
	
	=== "Orders Details"
	
		```
		Table "ORDERS"
		    Attribute    |         Type          | Modifier | Default Value
		-----------------+-----------------------+----------+---------------
		 O_ORDERKEY      | INTEGER               | NOT NULL |
		 O_CUSTKEY       | INTEGER               | NOT NULL |
		 O_ORDERSTATUS   | CHARACTER(1)          | NOT NULL |
		 O_TOTALPRICE    | NUMERIC(15,2)         | NOT NULL |
		 O_ORDERDATE     | DATE                  | NOT NULL |
		 O_ORDERPRIORITY | CHARACTER(15)         | NOT NULL |
		 O_CLERK         | CHARACTER(15)         | NOT NULL |
		 O_SHIPPRIORITY  | INTEGER               | NOT NULL |
		 O_COMMENT       | CHARACTER VARYING(79) | NOT NULL |
		Distributed on hash: “O_ORDERKEY”
		```
	
8.  Repeat executing the explain of our join query from the previous
	    section by executing the following command:
	
	=== "Input"
	
	    ```
	    EXPLAIN VERBOSE 
	    SELECT AVG(O.O_TOTALPRICE) AS PRICE, AVG(L.L_QUANTITY) AS QUANTITY,O_ORDERPRIORITY 
	    FROM ORDERS AS O, LINEITEM AS L 
	    WHERE L.L_ORDERKEY=O.O_ORDERKEY 
	    GROUP BY O_ORDERPRIORITY;
	    ```
	
	=== "Output"
	
		```
		EXPLAIN VERBOSE SELECT AVG(O.O_TOTALPRICE) AS PRICE, AVG(L.L_QUANTITY) AS QUANTITY, O_ORDERPRIORITY FROM ORDERS AS O, LINEITEM AS L WHERE L.L_ORDERKEY=O.O_ORDERKEY GROUP BY O_ORDERPRIORITY;
		
		QUERY VERBOSE PLAN:
		
		Node 1.
		  [SPU Sequential Scan table "ORDERS" as "O" {(O.O_ORDERKEY)}]
		      -- Estimated Rows = 1500000, Width = 27, Cost = 0.0 .. 1653.2, Conf = 100.0
		      Projections:
		        1:O.O_TOTALPRICE  2:O.O_ORDERPRIORITY  3:O.O_ORDERKEY
		  [HashIt for Join]
		Node 2.
		  [SPU Sequential Scan table "LINEITEM" as "L" {(L.L_ORDERKEY)}]
		      -- Estimated Rows = 6001215, Width = 12, Cost = 0.0 .. 6907.1, Conf = 100.0
		      Projections:
		        1:L.L_QUANTITY  2:L.L_ORDERKEY
		Node 3.
		  [SPU Hash Join Stream "Node 2" with Temp "Node 1" {(O.O_ORDERKEY,L.L_ORDERKEY)}]
		      -- Estimated Rows = 90018225000, Width = 31, Cost = 1653.2 .. 892653.2, Conf = 80.0
		      Restrictions:
		        (L.L_ORDERKEY = O.O_ORDERKEY)
		      Projections:
		        1:O.O_TOTALPRICE  2:L.L_QUANTITY  3:O.O_ORDERPRIORITY
		
		..<Rest of EXPLAIN Plan>..
		```
	
	The query itself has not been changed. The only changes are in the
	distribution keys of the involved tables. You will again see a long
	output. Scroll up to the start of the output, directly after your
	query.
	 
	Again, we do not want to make a complete analysis of the explain
	output. We will cover that in more detail in later chapters. But if
	you compare the output with the output of the last section you will
	see that the **[SPU Distribute on O.O_ORDERKEY)}]** nodes have
	vanished. The reason is that the join is now co-located because both
	tables are distributed on the join key. All rows to be joined from
	both tables are on the same data slices.
	 
	You may see a distribution node further below during the execution of
	the group by clause, but this is estimated to distribute only one
	hundred rows which has no negative performance influence.
	
9.  Finally execute the joined query again:
	
	=== "Input"
	
	    ```
	    \time
	    SELECT AVG(O.O_TOTALPRICE) AS PRICE, AVG(L.L_QUANTITY) AS QUANTITY,O_ORDERPRIORITY 
	    FROM ORDERS AS O, LINEITEM AS L 
	    WHERE L.L_ORDERKEY=O.O_ORDERKEY 
	    GROUP BY O_ORDERPRIORITY;
	    ```
	
	=== "Output"
	
		```
		     PRICE     | QUANTITY  | O_ORDERPRIORITY
		---------------+-----------+-----------------
		 189219.594349 | 25.532474 | 5-LOW
		 189285.029553 | 25.526186 | 2-HIGH
		 189093.608965 | 25.513563 | 1-URGENT
		 189026.093657 | 25.494518 | 3-MEDIUM
		 188546.457203 | 25.472923 | 4-NOT SPECIFIED
		(5 rows)
		Elapsed time: 0m1.051s
		```
		
	The query should return the same results as in the previous section
	but run faster even in the VM environment. In a real Netezza
	Performance Server with 8, 16 or more SPUs the difference would be
	much more significant.
	 
	You now have loaded the `LINEITEM` and `ORDERS` table into your
	Performance Server appliance using the optimal distribution key for
	these tables for most situations.
	
	a.  Both tables are distributed evenly across data slices, so there is
	    no data skew.
	
	b.  The distribution key is highly unlikely to result in processing
	    skew, since most `WHERE` conditions will restrict a key column evenly
	
	c.  Since `ORDERS` is a parent table of `LINEITEM`, with a foreign key
	    relationship between them, most queries joining them together will
	    utilize the join key. These queries will be co-located.
	
	Now we will pick the distribution keys of the full schema.

## 6 Schema Creation

Now that we have created the `ORDERS` and `LINEITEM` tables we need to pick
the distribution keys for the remaining tables as well.

### 6.1 Investigation

![A close up of a map Description automatically
generated](./nz-images/nz-03-Data-Distribution/media/image5.png)

***Figure 2 LABDB database***

You will notice that it is much harder to find optimal distribution keys
in a more complicated schema like this. In many situations you will be
forced to choose between enabling co-located joins between one set of
tables or another one.

The following provides some details on the remaining tables:


| **Table**           | **Number of Rows**   | **Primary Key**      |
|---------------------|----------------------|----------------------|
| REGION              | 5                    | R_REGIONKEY          |
| NATION              | 25                   | N_NATIONKEY          |
| CUSTOMER            | 150000               | C_CUSTKEY            |
| ORDERS              | 1500000              | O_ORDERKEY           |
| SUPPLIER            | 10000                | S_SUPPKEY            |
| PART                | 200000               | P_PARTKEY            |
| PARTSUPP            | 800000               | -               |
| LINEITEM            | 6000000              | -              |

And on the involved relationships:

 | **Parent Table**      | **Child Table**        | **Parent table Join Column**       | **Child table Join Column**  |
 |--------------|-----------------|----------------|--------------|
 | REGION        | NATION         | R_REGIONKEY    | N_REGIONKEY   |
 | NATION        | CUSTOMER       | N_NATIONKEY    | C_NATIONKEY   |
 | NATION        | SUPPLIER       | N_NATIONKEY    | S_NATIONKEY   |
 | CUSTOMER      | ORDERS         | C_CUSTKEY      | O_CUSTKEY     |
 | ORDERS        | LINEITEM       | O_ORDERKEY     | L_ORDERKEY    |
 | SUPPLIER      | LINEITEM       | S_SUPPKEY      | L_SUPPKEY     |
 | SUPPLIER      | PARTSUPP       | S_SUPPKEY      | PS_SUPPKEY    |
 | PART          | LINEITEM       | P_PARTKEY      | L_PARTKEY     |
 | PART          | PARTSUPP       | P_PARTKEY      | PS_PARTKEY    |

Given all that you heard in the presentation and lab, try to fill in the
distribution keys in the chart below. Let's assume that we will not
change the distribution keys for `LINEITEM` and `ORDERS` anymore.

 | **Table**                        | **Distribution Key (up to 4 columns) or Random** |
 |----------------------------------|------------------------------------|
 | REGION                           |                                    |
 | NATION                           |                                    |
 | CUSTOMER                         |                                    |
 | SUPPLIER                         |                                    |
 | PART                             |                                    |
 | PARTSUPP                         |                                    |
 | ORDERS                           | O_ORDERKEY                       |
 | LINEITEM                         | L_ORDERKEY                       |

### 6.2 Solution

It is important to note that there is no optimal way to pick
distribution keys. It always depends on the queries that run against the
database. Without these queries it is only possible to follow some
general rules:

-   Co-Location between big tables (esp. if a fact table is involved) is
    more important than between small tables.

-   Very small tables can be broadcast by the system with little
    performance penalty. If one table of a join is broadcast the other
    will not need to be redistributed.

-   If you suspect that there will be lots of queries joining two big
    tables but you cannot distribute both of them on the expected join
    key, distributing one table on the join key is better than nothing,
    since it will lead to a single redistribute instead of a double
    redistribute.

If we break down the problem, we can see that `PART` and `PARTSUPP` are the
biggest two of the remaining tables. Recall that we have already
distributed the `LINEITEM` and `ORDERS` tables based on the join key between
the two tables as seen in available customer queries. So, it might make
sense to distribute `PART` and `PARTSUPP` on the join key between these two
tables.

`CUSTOMER` is big as well and has two relationships. The first
relationship is with the very small `NATION` table that is easily
broadcasted by the system. The second relationship is with the `ORDERS`
table which is big as well but already distributed by the order key. But
as mentioned above a single redistribute is better than a double
redistribute. Therefore, it makes sense to distribute the `CUSTOMER` table
on the customer key, which is also the join key of this relationship.

The situation is very similar for the `SUPPLIER` table. It has two very
large child tables `PARTSUPP` and `LINEITEM` which are both related to it
through the supplier key, so it should be distributed on this key.

`NATION` and `REGION` are both very small and will most likely be
broadcasted by the Optimizer. You could distribute those tables
randomly, on their primary keys, on their join keys. In this case we
have decided to distribute both on their primary keys but there is no
definite right or wrong approaches. One possible solution for the
distribution keys could be the following.

 | **Table**                        | **Distribution Key (up to 4 columns) or Random** |
 |----------------------------------|------------------------------------|
 | REGION                           | R_REGIONKEY                      |
 | NATION                           | N_NATIONKEY                      |
 | CUSTOMER                         | C_CUSTKEY                        |
 | SUPPLIER                         | S_SUPPKEY                        |
 | PART                             | P_PARTKEY                        |
 | PARTSUPP                         | PS_PARTKEY                       |
 | ORDERS                           | O_ORDERKEY                       |
 | LINEITEM                         | L_ORDERKEY                       |

Finally, we will load the remaining tables.
	
1.  You should still be connected to the `LABDB` database. We now need to
	    recreate the `NATION` and `REGION` tables with a new distribution key.
	    To drop the old table versions execute the following commands.
	
	=== "Input"
	
	    ```
	    DROP TABLE NATION;
	    DROP TABLE REGION;
	    ```
	
2.  Quit the `NZSQL` console with the `\q` command.
	
3.  Navigate to the lab folder by executing the `cd
	    ~/labs/dataDistribution` command.
	
4.  Verify the SQL script creating the remaining 6 tables with the `more
	    remaining_tables.sql` command.
	
5.  Now create the remaining tables and load the data into it with the
	    following script:
	
	=== "Input"
	
	    ```
	    ./create_remaining.sh
	    ```
	
	=== "Output"
	
		```
		ERROR:  relation does not exist LABDB.ADMIN.NATION
		CREATE TABLE
		CREATE TABLE
		CREATE TABLE
		CREATE TABLE
		CREATE TABLE
		CREATE TABLE
		Load session of table 'NATION' completed successfully
		Load session of table 'REGION' completed successfully
		Load session of table 'CUSTOMER' completed successfully
		Load session of table 'SUPPLIER' completed successfully
		Load session of table 'PART' completed successfully
		Load session of table 'PARTSUPP' completed successfully
		```
		
	The error message at the top is expected since the script tries to
	clean up any old tables of the same name in case a reload is
	necessary. Also note that load logfiles have been created for these
	tables too. (`ls *.nzlog`). Review the load logfiles as time
	and interest permit.
	 
	=== "Input"
	
	    ```
	    ls -l *.nzlog
	    ```
	
	=== "Output"
	
		```
		-rw-rw-r--. 1 nz nz 2315 Mar 30 03:55 CUSTOMER.ADMIN.LABDB.4178.nzlog
		-rw-rw-r--. 1 nz nz 2321 Mar 29 13:51 LINEITEM.ADMIN.LABDB.29339.nzlog
		-rw-rw-r--. 1 nz nz 2320 Mar 29 14:05 LINEITEM.ADMIN.LABDB.32336.nzlog
		-rw-rw-r--. 1 nz nz 2320 Mar 30 03:16 LINEITEM.ADMIN.LABDB.370.nzlog
		-rw-rw-r--. 1 nz nz 2320 Mar 29 15:49 LINEITEM.ADMIN.LABDB.9042.nzlog
		-rw-rw-r--. 1 nz nz 2209 Mar 30 03:55 NATION.ADMIN.LABDB.4134.nzlog
		-rw-rw-r--. 1 nz nz 2317 Mar 30 02:58 ORDERS.ADMIN.LABDB.31185.nzlog
		-rw-rw-r--. 1 nz nz 2317 Mar 30 03:18 ORDERS.ADMIN.LABDB.581.nzlog
		-rw-rw-r--. 1 nz nz 2306 Mar 30 03:55 PART.ADMIN.LABDB.4227.nzlog
		-rw-rw-r--. 1 nz nz 2318 Mar 30 03:56 PARTSUPP.ADMIN.LABDB.4254.nzlog
		-rw-rw-r--. 1 nz nz 2206 Mar 30 03:55 REGION.ADMIN.LABDB.4156.nzlog
		-rw-rw-r--. 1 nz nz 2223 Mar 30 03:55 SUPPLIER.ADMIN.LABDB.4205.nzlog
		```
		

!!! success "Congratulations on finishing the chapter!"
	Congratulations! You just have defined data distribution keys for a
	customer data schema in Netezza Performance Server. You can have a look
	at the created tables and their definitions with the commands you used
	in the previous chapters. We will continue to use the tables we created
	in the following labs.
	