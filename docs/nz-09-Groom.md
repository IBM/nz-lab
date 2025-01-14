# Data Grooming

As part of your routine database maintenance activities, you should plan
to recover disk space occupied by outdated or deleted rows. In normal
Netezza Performance Server operation, an `UPDATE` or `DELETE` of a table row
does not remove the physical row on the hard disc. Instead the old row
is marked as deleted together with a transaction id of the deleting
transaction and in case of update a new row is created. This approach is
called multiversioning. Rows that could potentially be visible to other
transactions with an older transaction id are still accessible. Over
time however, the outdated or deleted rows are of no interest to any
transaction anymore and need to be removed to free up hard disc space
and improve performance. After the rows have been captured in a backup,
you can reclaim the space they occupy using the SQL `GROOM TABLE` command.
The `GROOM TABLE` command does not lock a table while it is running; you
can continue to `SELECT,` `UPDATE,` and `INSERT` into the table while the
table is being groomed.

## 1 Objectives

In this lab we will use the `GROOM` command to prepare our tables for the
customer. During the course of the POC we have deleted and update a
number of rows. At the end of a POC it is sensible to clean up the
system. Use Groom on the created tables, Generate Statistics, and other
cleanup tasks.

## 2 Lab Setup

This lab uses an initial setup script to make sure the correct user and
database exist for the remainder of the lab. Follow the instructions
below to run the setup script.

1.  Login to NPS Command Line using one of these two methods.

    a.  Login to the VM directly and use the terminal application
        available inside the VM.

    b.  Connect to your Netezza Performance Server image using putty

2.  If you are continuing from the previous lab and are already
    connected to NZSQL quit the NZSQL console with the `\q`
    command.

3.  Prepare for this lab by running the setup script. To do this use the
    following two commands:

	=== "Input"
		```
		cd ~/labs/groom/setupLab
		./setupLab.sh
		```
		
	=== "Output"
		```
		DROP DATABASE
		CREATE DATABASE
		ERROR: CREATE USER: object LABADMIN already exists as a USER.
		ALTER USER
		ALTER DATABASE
		CREATE TABLE
		CREATE TABLE
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
		Load session of table 'ORDERS' completed successfully
		Load session of table 'LINEITEM' completed successfully
		```
		
	There may be error message at the beginning of the output since the
	script tries to clean up existing databases and users.

## 3 Transactions

In this section we will show how transactions can leave logically
deleted rows in a table which later as an administrative task need to be
removed with the groom command. We will go through the different
transaction types and show you what happens under the covers in an
Netezza Performance Server Appliance.

### 3.1 Insert Transaction

In this chapter we will add a new row to the regions table and review
the hidden fields that are saved in the database. As you remember from
the Transactions presentation, Netezza Performance Server uses a concept
called multi-versioning for transactions. Each transaction has its own
image of the table and doesn't influence other transactions. This is
done by adding a number of hidden fields to the Netezza Performance
Server table. The most important ones are the `CREATEXID` and the
`DELETEXID`. Each Netezza Performance Server transaction has a unique
transaction id that is increasing with each new transaction.

In this subsection we will add a new row to the `REGION` table.

1.  Connect to your NPS system using a terminal application (i.e.: PuTTY
	    or Terminal). Login to `<ip-provided-by-your-instructor>` as user nz
	    with password nz. (`<ip-provided-by-your-instructor>` is the default
	    IP address for a lab system).
	
2.  Start nzsql from the Linux command line as following:
	
	=== "Input"
		```
		nzsql
		```
	
	=== "Output"
		```
			Welcome to nzsql, the IBM Netezza SQL interactive terminal.
			
		Type: \h for help with SQL commands
		      \? for help on internal slash commands
		      \g or terminate with semicolon to execute query
		      \q to quit
		      
		SYSTEM.ADMIN(ADMIN)=>
		```
	
	You will be entered into the nzsql interactive terminal.
	
3.  Connect to the database `LABDB` as user `LABADMIN` by typing the
	    following command:
	
	=== "Input"
		```
		\c LABDB LABADMIN
		```
	
	=== "Output"
		```
		You are now connected to database LABDB as user LABADMIN.
		LABDB.ADMIN(LABADMIN)=>
		```
	
	Notice the prompt has changed to show the new connection information.
	
4.  Select all rows from the `REGION` table:
	
	=== "Input"
		```
		SELECT * FROM REGION;
		```
	
	=== "Output"
		```
		 R_REGIONKEY |          R_NAME           |          R_COMMENT
		-------------+---------------------------+-----------------------------
		           3 | emea                      | europe, middle east, africa
		           1 | na                        | north america
		           2 | sa                        | south america
		           4 | ap                        | asia pacific
		(4 rows)
		```
	
	You should see the following output with 4 existing regions.
	
5.  Insert a new row into the `REGIONS` table for the region Australia
	    with the following SQL command
	
	=== "Input"
		```
		INSERT INTO REGION VALUES (5, 'as', 'australia');
		```
	
	=== "Output"
		```
		INSERTED 0 1
		```
	
6.  Now we will again do a select on the `REGION` table. But this time we
	    will also query the hidden fields `CREATEXID,` `DELETEXID` and `ROWID`:
	
	=== "Input"
		```
		SELECT CREATEXID, DELETEXID, ROWID,* FROM REGION;
		```
	
	=== "Output"
		```
		 CREATEXID | DELETEXID |  ROWID   | R_REGIONKEY |          R_NAME           |          R_COMMENT
		-----------+-----------+----------+-------------+---------------------------+-----------------------------
		     17498 |         0 | 28765000 |           3 | emea                      | europe, middle east, africa
		     17498 |         0 | 28765001 |           1 | na                        | north america
		     17498 |         0 | 28765002 |           2 | sa                        | south america
		     17498 |         0 | 28765003 |           4 | ap                        | asia pacific
		     17514 |         0 | 37428000 |           5 | as                        | australia
		(5 rows)
		```
		
	As you can see, we now have five rows in the `REGION` table. The new row
	for Australia has the ID of the last transaction as `CREATEXID` and 0 as
	DELETEXID since it has not yet been deleted. Other transactions with a
	lower transaction ID that might still be running will not be able to
	see this new row. Note also that each row has a unique `ROWID`. `ROWID`s
	do not need to be consecutive, but they are unique across all data
	slices for one table.

### 3.2 Update and Delete Transactions

Delete transactions in Netezza Performance Server do not physically
remove rows but update the `DELETEXID` field of a row to mark it as
logically deleted. These logically deleted rows need to be removed
regularly with the administrative `GROOM` command.

Update transactions in Netezza Performance Server consist of a logical
delete of the old row and an insert of a new row with the updated
fields. To show this effectively we will need to change a system
parameter in Netezza Performance Server that allows us to switch off the
invisibility lists in Netezza Performance Server. Note that the
parameter we will be using is dangerous and shouldn't be used in a real
Netezza Performance Server environment. There is also a safer
environment variable, but this has some restrictions.

To see deleted rows without changing the system registry parameters do
the following:

=== "Registry Parameters"
	```
	nzsql labdb labadmin password
	
	set show_deleted_records = true;
	select * from table_with_deleted_rows;
	set show_deleted_records = false;
	```
	
The above method is not used in this lab, please follow the steps
below.

1.  First, we will change the system variable that allows us to see
	    deleted rows in the system, to do this exit the console with `\q`.
	
2.  Check the Netezza Performance Server system registry for the
	    parameters host.fpgaAllowXIDOverride and system.useFpgaPrep, each
	    should be set to yes:
	
	=== "Input"
		```
		nzsystem showregistry | grep -iE 'fpgaAllowXIDOverride|useFpgaPrep'
		```
	
	=== "Output"
		```
		host.fpgaAllowXIDOverride = no
		system.useFpgaPrep = yes
		```
	
3.  To change these system parameters, first pause the system with the
	    following command:
	
	=== "Input"
		```
		nzsystem pause
		Are you sure you want to pause the system (y|n)? [n] y
		```
	
4.  Next, update the system parameters with the following command:
	
	=== "Input"
		```
		nzsystem set -arg host.fpgaAllowXIDOverride=yes
		Are you sure you want to change the system configuration (y|n)? [n] y
		```
	
	Ensure both parameters `host.fpgaAllowXIDOverride` and
	`system.useFpgaPrep` are set to `yes`.
	
5. Resume the system with the following command:
	
	=== "Input"
		```
		nzsystem resume
		```
	
6. Re-check the Netezza Performance Server system registry for the
	    parameters `host.fpgaAllowXIDOverride` and `system.useFpgaPrep`, each
	    should be set to yes:
	
	=== "Input"
		```
		nzsystem showregistry | grep -iE 'fpgaAllowXIDOverride|useFpgaPrep'
		```
	
	=== "Output"
		```
		host.fpgaAllowXIDOverride = yes
		system.useFpgaPrep = yes
		```
	
7. Start `nzsql` from the Linux command line as following:
	
	=== "Input"
		```
		nzsql labdb labadmin password
		```
	
	
8. Now we will update the row we inserted in the last chapter to the
	    `REGION` table:
	
	=== "Input"
		```
		UPDATE REGION SET R_COMMENT='Australia' WHERE R_REGIONKEY=5;
		```
	
9. Do a `SELECT` on the `REGION` table again:
	
	=== "Input"
		```
		SELECT CREATEXID, DELETEXID, ROWID,* FROM REGION;
		```
	
	=== "Output"
		```
		 CREATEXID | DELETEXID |  ROWID   | R_REGIONKEY |          R_NAME           |          R_COMMENT
		-----------+-----------+----------+-------------+---------------------------+-----------------------------
		     17498 |         0 | 28765000 |           3 | emea                      | europe, middle east, africa
		     17498 |         0 | 28765001 |           1 | na                        | north america
		     17498 |         0 | 28765002 |           2 | sa                        | south america
		     17498 |         0 | 28765003 |           4 | ap                        | asia pacific
		     17514 |     21506 | 37428000 |           5 | as                        | australia
		     21506 |         0 | 37428000 |           5 | as                        | Australia
		(6 rows)
		```
	
	Normally you would now see 5 rows with the update value. But since we
	disabled the invisibility lists you now see 6 rows in the `REGION`
	table. Our transaction that updated the row had the transaction id
	369666. You can see that the original row with the lowercase australia
	in the comment column is still there and now has a `DELETXID` field that
	contains the transaction id of the transaction that deleted it.
	Transactions with a higher transaction id will not see a row with a
	`DELETEXID` that indicates that it has been logically deleted before the
	transaction is run.
	 
	We also see a newly inserted row with the new comment value Australia.
	It has the same `ROWID` as the deleted row and the same `CREATEXID` as the
	transaction that did the insert.
	
10. Finally let's clean up the table again by deleting the Australia
	    row:
	
	=== "Input"
		```
		DELETE FROM REGION WHERE R_REGIONKEY=5;
		```
	
	=== "Output"
		```
		DELETE 1
		```
	
11. Do a `SELECT` on the `REGION` table again:
	
	=== "Input"
		```
		SELECT CREATEXID, DELETEXID, ROWID,* FROM REGION;
		```
	
	=== "Output"
		```
		 CREATEXID | DELETEXID |  ROWID   | R_REGIONKEY |          R_NAME           |          R_COMMENT
		-----------+-----------+----------+-------------+---------------------------+-----------------------------
		     17498 |         0 | 28765000 |           3 | emea                      | europe, middle east, africa
		     17498 |         0 | 28765001 |           1 | na                        | north america
		     17498 |         0 | 28765002 |           2 | sa                        | south america
		     17498 |         0 | 28765003 |           4 | ap                        | asia pacific
		     17514 |     21506 | 37428000 |           5 | as                        | australia
		     21506 |     21510 | 37428000 |           5 | as                        | Australia
		(6 rows)
		
		```
	
	We can now see that we have logically deleted our updated row as well.
	It has now a `DELETEXID` field with the value of the new transaction.
	New transactions will see the original table from the start of this
	lab again.
	
	If you do a `SELECT`, the FPGA will filter out all rows that:
	
	-   have a `CREATEXID` which is bigger than the current transaction id.
	
	-   have a `CREATEXID` of an uncommitted transaction.
	
	-   have a `DELETENXID` which is smaller than the current transaction, but
	    only if the transaction of the `DELETEXID` field is committed.
	
	-   have a `DELETEXID` of 1 which means that the insert has been aborted.
	
### 3.3 Aborting Transactions

Netezza Performance Server never deletes a row during transactions even
if transactions are rolled back. In this section we will show what
happens if a transaction is rolled back. Since an update transaction
consists of a `DELETE` and `INSERT` transaction, we will demonstrate the
behavior for all tree transaction types with this.

1.  To start a transaction that we can later rollback we need to use the
	    `BEGIN` keyword.
	
	=== "Input"
		```
		BEGIN;
		```
	
	=== "Output"
		```
		BEGIN
		```
	
	Per default all SQL statements entered into the nzsql console are
	auto-committed. To start a multi command transaction the `BEGIN` keyword
	needs to be used. All SQL statements that are executed after it will
	belong to a single transaction. To end the transaction two keywords
	can be used `COMMIT` to commit the transaction or `ROLLBACK` to rollback
	the transaction and all changes since the `BEGIN` statement was
	executed.
	
2. Update the row for the AP region:
	
	=== "Input"
		```
		UPDATE REGION SET R_COMMENT='AP' WHERE R_REGIONKEY=4;
		```
	
	=== "Output"
		```
		UPDATE 1
		```
	
3. Do a `SELECT` on the `REGION` table again:
	
	=== "Input"
		```
		SELECT CREATEXID, DELETEXID, ROWID,* FROM REGION;
		```
	
	=== "Output"
		```
		 CREATEXID | DELETEXID |  ROWID   | R_REGIONKEY |          R_NAME           |          R_COMMENT
		-----------+-----------+----------+-------------+---------------------------+-----------------------------
		     17498 |         0 | 28765000 |           3 | emea                      | europe, middle east, africa
		     17498 |         0 | 28765001 |           1 | na                        | north america
		     17498 |         0 | 28765002 |           2 | sa                        | south america
		     17498 |     21514 | 28765003 |           4 | ap                        | asia pacific
		     17514 |     21506 | 37428000 |           5 | as                        | australia
		     21506 |     21510 | 37428000 |           5 | as                        | Australia
		     21514 |         0 | 28765003 |           4 | ap                        | AP
		(7 rows)
		```
	
	Note: we have the same results as in the last chapter, the original
	row for the AP region was logically deleted by updating its `DELETEXID`
	field, and a new row with the updated comment and new `ROWID` has been
	added. Note that its `CREATEXID` is the same as the `DELETEXID` of the old
	row, since they were updated by the same transaction.
	
4. Now let's rollback the transaction:
	
	=== "Input"
		```
		ROLLBACK;
		```
	
	=== "Output"
		```
		ROLLBACK
		```
	
5. Do a `SELECT` on the `REGION` table again:
	
	=== "Input"
		```
		SELECT CREATEXID, DELETEXID, ROWID,* FROM REGION;
		```
	
	=== "Output"
		```
		 CREATEXID | DELETEXID |  ROWID   | R_REGIONKEY |          R_NAME           |          R_COMMENT
		-----------+-----------+----------+-------------+---------------------------+-----------------------------
		     17498 |         0 | 28765000 |           3 | emea                      | europe, middle east, africa
		     17498 |         0 | 28765001 |           1 | na                        | north america
		     17498 |         0 | 28765002 |           2 | sa                        | south america
		     17498 |         0 | 28765003 |           4 | ap                        | asia pacific
		     17514 |     21506 | 37428000 |           5 | as                        | australia
		     21506 |     21510 | 37428000 |           5 | as                        | Australia
		     21514 |         1 | 28765003 |           4 | ap                        | AP
		(7 rows)
		```
	
	We can see that the transaction has been rolled back. The `DELETEXID` of
	the old version of the row has been reset to 0, which means that it is
	a valid row that can be seen by other transactions, and the `DELETEXID`
	of the new row has been set to 1 which marks it as aborted.
	
### 3.4 Cleaning up

In this section we will use the `GROOM` command to remove the logically
deleted rows we have entered, and we will remove the system parameter
from the configuration file. The `GROOM` command will be used in more
detail in the next chapter. It is the main maintenance command in
Netezza Performance Server, and we have already used it in the
Cluster-based Table labs to reorder a CBT. It also removes all logically
deleted rows from a table and frees up the space on the machine again.

1.  Execute the `GROOM` command on the `REGION` table:
	
	=== "Input"
		```
		GROOM TABLE REGION;
		```
	
	=== "Output"
		```
		NOTICE: Groom will not purge records deleted by transactions that
		started after 2020-04-02 19:18:49.
		
		NOTICE: Groom processed 1 pages; purged 3 records; scan size unchanged;
		table size unchanged.
		
		GROOM RECORDS ALL
		```
	
	You can see that the `GROOM` command purged 3 rows, exactly the number
	of aborted and logically deleted rows we have generated in the
	previous chapter.
	
2. Now select the rows from the `REGION` table again.
	
	=== "Input"
		```
		SELECT CREATEXID, DELETEXID, ROWID,* FROM REGION;
		```
	
	=== "Output"
		```
		 CREATEXID | DELETEXID |  ROWID   | R_REGIONKEY |          R_NAME           |          R_COMMENT
		-----------+-----------+----------+-------------+---------------------------+-----------------------------
		     17498 |         0 | 28765000 |           3 | emea                      | europe, middle east, africa
		     17498 |         0 | 28765001 |           1 | na                        | north america
		     17498 |         0 | 28765002 |           2 | sa                        | south america
		     17498 |         0 | 28765003 |           4 | ap                        | asia pacific
		(4 rows)
		```
		
	You can see that the `GROOM` command has removed all logically deleted
	rows from the table. Remember that we still have the parameter
	switched on that allows us to see any logically deleted rows.
	Especially in tables that are heavily changed with lots and updates
	and deletes running the groom command will free up hard drive space
	and increase performance.
	
3.  Finally, we will change the system variables back to the original
	    settings, to do this exit the console with `\q`.
	
	
4. To change these system parameters, first pause the system with the
	    following command:
	
	=== "Input"
		```
		nzsystem pause
		Are you sure you want to pause the system (y|n)? [n]  y
		```
	
5. Next, update the system parameters with the following command:
	
	=== "Input"
		```
		nzsystem set -arg host.fpgaAllowXIDOverride=no
		Are you sure you want to change the system configuration (y|n)? [n] y
		```
	
	Ensure both parameters `host.fpgaAllowXIDOverride=no` and
	`system.useFpgaPrep=yes`.
	
6. Resume the system with the following command:
	
	=== "Input"
		```
		nzsystem resume
		```
	
7.  Re-check the Netezza Performance Server system registry for the
	    parameters `host.fpgaAllowXIDOverride` and `system.useFpgaPrep`, each
	    should be set to yes:
	
	=== "Input"
		```
		nzsystem showregistry | grep -iE 'fpgaAllowXIDOverride|useFpgaPrep'
		```
	
	=== "Output"
		```
		host.fpgaAllowXIDOverride = no
		system.useFpgaPrep = yes
		```
	
## 4 Grooming Logically Deleted Rows

In this section we will delete rows and determine that they have not
really been deleted from the disk. Then using `GROOM` we will physically
delete the rows.

1.  First determine the physical size on disk of the table `ORDERS` using
	    the following command:
	
	You should see the following results:
	
	=== "Input"
		```
		/nz/support/bin/nz_db_size LABDB
		```
	
	=== "Output"
		```
		  Object   |               Name               |        Bytes         |        KB        |      MB      |     GB     |   TB
		-----------+----------------------------------+----------------------+------------------+--------------+------------+--------
		 Appliance | localhost                        |          452,984,832 |          442,368 |          432 |         .4 |     .0
		 Database  | LABDB                            |          452,984,832 |          442,368 |          432 |         .4 |     .0
		   .schema | ADMIN                            |          452,984,832 |          442,368 |          432 |         .4 |     .0
		 Table     | CUSTOMER                         |           13,107,200 |           12,800 |           13 |         .0 |     .0
		 Table     | LINEITEM                         |          284,688,384 |          278,016 |          272 |         .3 |     .0
		 Table     | NATION                           |              131,072 |              128 |            0 |         .0 |     .0
		 Table     | ORDERS                           |           76,283,904 |           74,496 |           73 |         .1 |     .0
		 Table     | PART                             |           11,534,336 |           11,264 |           11 |         .0 |     .0
		 Table     | PARTSUPP                         |           66,322,432 |           64,768 |           63 |         .1 |     .0
		 Table     | REGION                           |              131,072 |              128 |            0 |         .0 |     .0
		 Table     | SUPPLIER                         |              786,432 |              768 |            1 |         .0 |     .0
		
		```
		
	Notice that the `ORDERS` table is 75 MB in size.
	
2. Now we are going to delete some rows from `ORDERS` table. Delete all
	    rows where the `ORDERSTATUS` is marked as F for finished using the
	    following command:
	
	=== "Input"
		```
		nzsql LABDB LABADMIN password
		DELETE FROM ORDERS WHERE O_ORDERSTATUS='F';
		```
	
	=== "Output"
		```
		DELETE 729413
		```
	
3. Now check the physical table size for `ORDERS` and see if the size
	    decreased using the same command as before. You must first exit
	    nzsql to shell using `\q`.
	
	=== "Input"
		```
		\q
		/nz/support/bin/nz_db_size LABDB
		```
	
	=== "Output"
		```
		  Object   |               Name               |        Bytes         |        KB        |      MB      |     GB     |   TB
		-----------+----------------------------------+----------------------+------------------+--------------+------------+--------
		 Appliance | localhost                        |          452,984,832 |          442,368 |          432 |         .4 |     .0
		 Database  | LABDB                            |          452,984,832 |          442,368 |          432 |         .4 |     .0
		   .schema | ADMIN                            |          452,984,832 |          442,368 |          432 |         .4 |     .0
		 Table     | CUSTOMER                         |           13,107,200 |           12,800 |           13 |         .0 |     .0
		 Table     | LINEITEM                         |          284,688,384 |          278,016 |          272 |         .3 |     .0
		 Table     | NATION                           |              131,072 |              128 |            0 |         .0 |     .0
		 Table     | ORDERS                           |           76,283,904 |           74,496 |           73 |         .1 |     .0
		 Table     | PART                             |           11,534,336 |           11,264 |           11 |         .0 |     .0
		 Table     | PARTSUPP                         |           66,322,432 |           64,768 |           63 |         .1 |     .0
		 Table     | REGION                           |              131,072 |              128 |            0 |         .0 |     .0
		 Table     | SUPPLIER                         |              786,432 |              768 |            1 |         .0 |     .0
		```
		
	The output should be the same as above showing that the `ORDERS` table
	did not change in size and is still 75 MB. This is because the deleted
	rows were logically deleted but are still left on disk. The rows will
	still accessible to transactions that started before the `DELETE`
	statement which we just executed. (i.e. have a lower transaction id)
	
4. Next let's physically delete what we just logically deleted using
	    the `GROOM TABLE` command and specifying table `ORDERS`. When you run
	    the `GROOM TABLE` command, it removes outdated and deleted records
	    from tables.
	
	=== "Input"
		```
		nzsql LABDB LABADMIN password
		GROOM TABLE ORDERS;
		```
	
	=== "Output"
		```
		NOTICE: Groom will not purge records deleted by transactions that
		started after 2020-04-02 19:56:57.
		
		NOTICE: Groom processed 582 pages; purged 729413 records; scan size
		shrunk by 280 pages; table size shrunk by 12 extents.
		
		GROOM RECORDS ALL
		```
	
	You can see that 729413 rows were removed from disk resulting in the
	table size shrinking by 12 extents. Notice that this is the same
	number of rows we deleted in the previous step.
	
5. Check if the `ORDERS` table size on disk has shrunk using the
	    `nz_db_size` command. You must first exit `nzsql` to shell using `\q`.
	
	=== "Input"
		```
		\q
		/nz/support/bin/nz_db_size LABDB
		```
	
	=== "Output"
		```
		  Object   |               Name               |        Bytes         |        KB        |      MB      |     GB     |   TB
		-----------+----------------------------------+----------------------+------------------+--------------+------------+--------
		 Appliance | localhost                        |          416,284,672 |          406,528 |          397 |         .4 |     .0
		 Database  | LABDB                            |          416,284,672 |          406,528 |          397 |         .4 |     .0
		   .schema | ADMIN                            |          416,284,672 |          406,528 |          397 |         .4 |     .0
		 Table     | CUSTOMER                         |           13,107,200 |           12,800 |           13 |         .0 |     .0
		 Table     | LINEITEM                         |          284,688,384 |          278,016 |          272 |         .3 |     .0
		 Table     | NATION                           |              131,072 |              128 |            0 |         .0 |     .0
		 Table     | ORDERS                           |           39,583,744 |           38,656 |           38 |         .0 |     .0
		 Table     | PART                             |           11,534,336 |           11,264 |           11 |         .0 |     .0
		 Table     | PARTSUPP                         |           66,322,432 |           64,768 |           63 |         .1 |     .0
		 Table     | REGION                           |              131,072 |              128 |            0 |         .0 |     .0
		 Table     | SUPPLIER                         |              786,432 |              768 |            1 |         .0 |     .0
		
		```
	
	Notice the reduced size of the `ORDERS` table. We can see that `GROOM` did
	purge the deleted rows from disk. `GROOM` reported that the table size
	was reduced by 12 extents and we can confirm this because we can see
	that the size of the table reduced by 36MB which is the correct size
	for 12 extents. (1 extent's size is 3 MB).

## 5 Performance Benefits of GROOM

In this section we will show that grooming a table can also result in a
performance benefit because the amount of data that needs to be scanned
is smaller. Outdated rows are still present on the hard disc. They can
be dismissed by the FPGA but the system still needs to read them from
disc. In this example we need for accounting reasons increase the order
price of all columns. This means that we need to update every row in the
`ORDERS` table. We will measure query performance before and after
Grooming the table.

1.  Update the `ORDERS` table so that the price of everything is increased
	    by $1. Do this using the following command:
	
	=== "Input"
		```
		nzsql LABDB LABADMIN password
		UPDATE ORDERS SET O_TOTALPRICE = O_TOTALPRICE+1;
		```
	
	=== "Output"
		```
		UPDATE 770587
		```
	
	All rows will be affected by the update resulting in a doubled number
	of physical rows in the table. This is because the `UPDATE` operation
	leaves a copy of the rows before the `UPDATE` occurred in case a
	transaction is still operating on the rows. New rows are created, and
	the results of the `UPDATE` are put in these rows. The old rows that are
	left on disk are marked as logically deleted.
	
2. To measure the performance of our test query, we can configure the
	    nzsql console to show the elapsed execution time using the following
	    command:
	
	=== "Input"
		```
		\time
		```
	
	=== "Output"
		```
		Query time printout on
		```
	
3. Run our given test query and note the performance:
	
	=== "Input"
		```
		SELECT COUNT(*) FROM ORDERS;
		```
	
	=== "Output"
		```
	
		COUNT
		--------
		770587
		(1 row)
	
		Elapsed time: 0m0.641s
		```
	
4. Please rerun the query once or twice more to see roughly what a
	    consistent query time is on your machine.
	
	=== "Input"
		```
		SELECT COUNT(*) FROM ORDERS;
		```
	
	
5. Now run the `GROOM TABLE` command on the `ORDER` table again:
	
	=== "Input"
		```
		GROOM TABLE ORDERS;
		```
	
	=== "Output"
		```
		NOTICE: Groom will not purge records deleted by transactions that
		started after 2020-04-02 20:12:07.
		
		NOTICE: Groom processed 604 pages; purged 770587 records; scan size
		shrunk by 302 pages; table size shrunk by 13 extents.
		
		GROOM RECORDS ALL
		
		Elapsed time: 0m2.026s
		```
	
	Can you tell how much disk space this saved? (It's the number of
	extents times 3MB)
	
6. Now run our chosen test query again and you should see a difference
	    in performance:
	
	=== "Input"
		```
		SELECT COUNT(*) FROM ORDERS;
		```
	
	=== "Output"
		```
		COUNT
		--------
		770587
		(1 row)
		Elapsed time: 0m0.082s
		```
	
	You should see that the query ran faster than before. This is because
	`GROOM` reduced the number of rows that must be scanned to complete the
	query. The `COUNT(*)` command on the table will return the same number
	of rows before and after the `GROOM` command was run since it can only
	see the current version of the table, which means all rows that have
	not been deleted by a lower transaction id. Since our `UPDATE` command
	hasn't changed the number of logical rows this will not change.
	
	Nevertheless, the outdated rows, which have been logically deleted by
	our `UPDATE` command, are still present on disk. The `COUNT(*)` query
	cannot access these rows but they do take up space on disk and need to
	be scanned. `GROOM` is used to purge these logically deleted rows from
	disk which increase disk usage and scan distance. You should `GROOM`
	tables that receive frequent updates or deletes more often than tables
	that are seldom updated. You might want to schedule tasks that
	routinely `GROOM` the frequently updated tables or run a `GROOM` command
	as part of you ETL process.

## 6 Changing the Data Type of a Column

In some situations, you will realize that the initially used data types
are not suitable for long-term use, for example because new entries
exceed the range of an initially picked integer value. You cannot
directly change the data type by using the `ALTER` statement but there are
two approaches that allow you to do it without loading and unloading the
data.

The first approach is to:

-   Create a CTAS table from the old table with a `CAST` to the new
    datatype for the column you want to change

-   Drop the old table

-   Rename the new table to the name of the old table

In general, this is a good approach because it lets you keep the order
of the columns. But in this example, we will use a second approach to
highlight the groom command and its role during `ADD` and `DROP` column
commands. Its disadvantages are that the order of the columns will
change, which may result in difficulties for third party applications
that access columns by their order.

In this chapter we will:

-   Add a new column to the table with the new datatype

-   Copy over all values from the old row to the new one with an `UPDATE`
    command

-   Drop the old column

-   Rename the new column to the name of the old one

-   Use the groom command to materialize the results of our table
    changes

For our example we find out that we have a new Region we want to add to
our `REGIONS` table which has a name that exceeds the limits of the
`CHAR(25)` field `R_NAME`. "Australia, New Zealand, and Tasmania". And we
decide to increase the `R_NAME` field to a `CHAR(40)` field.

1.  Add a new column to the region table with name `R_NAME_TEMP` and data
	    type `CHAR(40)`
	
	=== "Timer ON"
		```
		\time
		```
	
	=== "Output"
		```
		Query time printout on
		```
	
	=== "ALTER TABLE"
		```
		ALTER TABLE REGION ADD COLUMN R_NAME_TEMP CHAR(40);
		```
	
	=== "Output"
		```
		ALTER TABLE
		```
	
	Notice that the `ALTER` command is practically instantaneous. This even
	holds true for huge tables. Under the cover the system will create a
	new empty version of the table. It will not lock and change the whole
	table.
	
2. Let's insert a row into the table using the new name column
	
	=== "Input"
		```
		INSERT INTO REGION VALUES
			(5,'', 'South Pacific Region',
			'Australia, New Zealand, and Tasmania');
		```
	
	=== "Output"
		```
		INSERT 0 1
		```
	
3. Now do a select on the table:
	
	=== "Input"
		```
		SELECT * FROM REGION;
		```
	
	=== "Output"
		```
		R_REGIONKEY  |          R_NAME           |          R_COMMENT          |               R_NAME_TEMP
		-------------+---------------------------+-----------------------------+------------------------------------------
		           3 | emea                      | europe, middle east, africa |
		           1 | na                        | north america               |
		           2 | sa                        | south america               |
		           4 | ap                        | asia pacific                |
		           5 |                           | South Pacific Region        | Australia, New Zealand, and Tasmania
		(5 rows)
		```
	
	You can see that the results are exactly as you would expect them to
	be, but how does the system actually achieve this. Remember inside the
	Netezza Performance Server appliances we have two versions of the
	table, one containing the old columns and rows and one containing the
	new row column.
	
4. Let's do an `EXPLAIN` on the `SELECT` query
	
	=== "Input"
		```
		EXPLAIN VERBOSE SELECT * FROM REGION;
		```
	
	=== "Output"
		```
		NOTICE:  QUERY PLAN:
		
		
		QUERY SQL:
		
		EXPLAIN VERBOSE SELECT * FROM REGION;
		
		
		
		QUERY VERBOSE PLAN:
		
		Node 1.
		  [SPU Sequential Scan table ""_TV_203063_2"" {("_TV_203063_2".R_REGIONKEY)}]
		      -- Estimated Rows = 1, Width = 221, Cost = 0.0 .. 0.0, Conf = 100.0
		      User table: REGION version 2
		      Projections:
		        1:"_TV_203063_2".R_REGIONKEY  2:"_TV_203063_2".R_NAME
		        3:"_TV_203063_2".R_COMMENT  4:"_TV_203063_2".R_NAME_TEMP
		Node 2.
		  [SPU Sub-query Scan table "*SELECT* 1" Node "1" {(0."1")}]
		      -- Estimated Rows = 1, Width = 221, Cost = 0.0 .. 0.0, Conf = 0.0
		      Projections:
		        1:0."1"  2:0."2"  3:0."3"  4:0."4"
		Node 3.
		  [SPU Sequential Scan table ""_TV_203063_1"" {("_TV_203063_1".R_REGIONKEY)}]
		      -- Estimated Rows = 4, Width = 221, Cost = 0.0 .. 0.0, Conf = 100.0
		      User table: REGION version 1
		      Projections:
		        1:"_TV_203063_1".R_REGIONKEY  2:"_TV_203063_1".R_NAME
		        3:"_TV_203063_1".R_COMMENT  4:(NULL::BPCHAR)::CHAR(40)
		Node 4.
		  [SPU Sub-query Scan table "*SELECT* 2" Node "3" {(0."1")}]
		      -- Estimated Rows = 4, Width = 221, Cost = 0.0 .. 0.0, Conf = 0.0
		      Projections:
		        1:0."1"  2:0."2"  3:0."3"  4:0."4"
		Node 5.
		  [SPU Append Nodes: , "2", "4 (stream)" {(0."1")}]
		      -- Estimated Rows = 5, Width = 221, Cost = 0.0 .. 0.0, Conf = 0.0
		      Projections:
		        1:0."1"  2:0."2"  3:0."3"  4:0."4"
		Node 6.
		  [SPU Sub-query Scan table "_BV_203063" Node "5" {("_BV_203063".R_REGIONKEY)}]
		      -- Estimated Rows = 5, Width = 221, Cost = 0.0 .. 0.0, Conf = 100.0
		      Projections:
		        1:"_BV_203063".R_REGIONKEY  2:"_BV_203063".R_NAME  3:"_BV_203063".R_COMMENT
		        4:"_BV_203063".R_NAME_TEMP
		  [SPU Return]
		  [Host Return]
		
		
		QUERY PLANTEXT:
		
		Sub-query Scan table "_BV_203063" (cost=0.0..0.0 rows=5 width=221 conf=100) {("_BV_203063".R_REGIONKEY)}
		(xpath_none, locus=spu subject=self)
		(xpath_none, locus=spu subject=self)
		(xpath_none, locus=spu subject=self)
		(spu_send, locus=host subject=self)
		(host_return, locus=host subject=self)
		   l: Append (cost=0.0..0.0 rows=5 width=221 conf=0) {(0."1")}
		      (xpath_none, locus=spu subject=self)
		      (xpath_none, locus=spu subject=self)
		      a: Sub-query Scan table "*SELECT* 1" (cost=0.0..0.0 rows=1 width=221 conf=0) {(0."1")}
		         (xpath_none, locus=spu subject=self)
		         l: Sequential Scan table ""_TV_203063_2"" (cost=0.0..0.0 rows=1 width=221 conf=100)  {("_TV_203063_2".R_REGIONKEY)}
		            (User table: REGION version 2)
		            (xpath_none, locus=spu subject=self)
		      a: Sub-query Scan table "*SELECT* 2" (cost=0.0..0.0 rows=4 width=221 conf=0) {(0."1")}
		         (xpath_none, locus=spu subject=self)
		         l: Sequential Scan table ""_TV_203063_1"" (cost=0.0..0.0 rows=4 width=221 conf=100)  {("_TV_203063_1".R_REGIONKEY)}
		            (User table: REGION version 1)
		            (xpath_none, locus=spu subject=self)
		
		
		
		EXPLAIN
		
		```
	
	Normally the query would result in a single table scan node. But now
	we see a more complicated query plan. The Optimizer automatically
	translates the simple `SELECT` into a `UNION` of two tables. The two
	tables are internal and are called `TV_203063_1`, which is the old
	version of the table before the `ALTER` statement. And
	`TV_203063_2`, which is the new version of the table after the
	table statement containing the new column `R_NAME_TEMP`.
	
	Notice that in the old table a 4th column of `CHAR(40)` with default
	value `NULL` is added. This is necessary for the `UNION` to succeed. The
	merger of those tables is done in Node 5, which takes both result sets
	and appends them.
	
	But let's proceed with our data type change operation.
	
5. Let's remove the new row again.
	
	=== "Input"
		```
		DELETE FROM REGION WHERE R_REGIONKEY > 4;
		```
	
	=== "Output"
		```
		DELETE 1
		```
	
6. Now we will move all values of the `R_NAME` column to the `R_NAME_TEMP`
	    column by updating them
	
	=== "Input"
		```
		UPDATE REGION SET R_NAME_TEMP = R_NAME;
		```
	
	=== "Output"
		```
		UPDATE 4
		```
	
7. Let's have a look at the table again:
	
	=== "Input"
		```
		SELECT * FROM REGION;
		```
	
	=== "Output"
		```
		R_REGIONKEY  |          R_NAME           |          R_COMMENT          |               R_NAME_TEMP
		-------------+---------------------------+-----------------------------+------------------------------------------
		           3 | emea                      | europe, middle east, africa | emea
		           1 | na                        | north america               | na
		           2 | sa                        | south america               | sa
		           4 | ap                        | asia pacific                | ap
		(4 rows)
		```
	
8. Now let's remove the old column:
	
	=== "Input"
		```
		ALTER TABLE REGION DROP COLUMN R_NAME RESTRICT;
		```
	
	=== "Output"
		```
		ALTER TABLE
		```
	
9. Rename the column name `R_NAME_TEMP` to `R_NAME`
	
	=== "Input"
		```
		ALTER TABLE REGION RENAME COLUMN R_NAME_TEMP TO R_NAME;
		```
	
	=== "Output"
		```
		ALTER TABLE
		```
	
10. Let's have a look at the table again:
	
	=== "Input"
		```
		SELECT * FROM REGION;
		```
	
	=== "Output"
		```
		R_REGIONKEY  |          R_COMMENT          |                  R_NAME
		-------------+-----------------------------+------------------------------------------
		           3 | europe, middle east, africa | emea
		           1 | north america               | na
		           2 | south america               | sa
		           4 | asia pacific                | ap
		(4 rows)
		```
	
	We have achieved to change the data type of the `R_NAME` column. The
	column order has changed but our `R_NAME` column has the same values as
	before and now supports longer region names.
	
	But we have one last step to do. Under the cover the system now has
	three different versions of the table which are merged for each call
	against the `REGION` table. This not only uses up space it is also bad
	for the query performance. So, we have to materialize these table
	changes with the `GROOM` command.
	
11. `GROOM` the `REGION` table with the `VERSIONS` keyword to merge table
	    versions:
	
	=== "Input"
		```
		GROOM TABLE REGION VERSIONS;
		```
		
	=== "Output"
		```
		NOTICE: Groom will not purge records deleted by transactions that
		started after 2020-04-03 04:04:56.
		NOTICE: If this process is interrupted please either repeat GROOM
		VERSIONS or issue 'GENERATE STATISTICS ON "REGION"'
		NOTICE: Groom processed 2 pages; purged 5 records; scan size shrunk by 1
		pages; table size shrunk by 1 extents.
		
		GROOM VERSIONS
		```
		
12. Finally, we will look at the `EXPLAIN` output again:
	
	=== "Input"
		```
		EXPLAIN VERBOSE SELECT * FROM REGION;
		```
	
	=== "Output"
		```
		NOTICE:  QUERY PLAN:
		
		
		QUERY SQL:
		
		EXPLAIN VERBOSE SELECT * FROM REGION;
		
		
		QUERY VERBOSE PLAN:
		
		Node 1.
		  [SPU Sequential Scan table "REGION" {(REGION.R_REGIONKEY)}]
		      -- Estimated Rows = 4, Width = 196, Cost = 0.0 .. 0.0, Conf = 100.0
		      Projections:
		        1:REGION.R_REGIONKEY  2:REGION.R_COMMENT  3:REGION.R_NAME
		  [SPU Return]
		  [Host Return]
		
		
		QUERY PLANTEXT:
		
		Sequential Scan table "REGION" (cost=0.0..0.0 rows=4 width=196 conf=100)  {(REGION.R_REGIONKEY)}
		(xpath_none, locus=spu subject=self)
		(spu_send, locus=host subject=self)
		(host_return, locus=host subject=self)
		
		
		EXPLAIN
		
		```
	
	Now this is much nicer. As we would expect we only have a single table
	scan snippet in the query plan and a single version of the `REGION`
	table.
	
13. Finally, we will return the `REGION` table to the old column ordering
	    to not interfere with future labs, to do this we will use a CTAS
	    statement
	
	=== "Input"
		```
		CREATE TABLE REGION_NEW AS
		  SELECT R.R_REGIONKEY, R.R_NAME, R.R_COMMENT
		FROM REGION R;
		```
	
	=== "Output"
		```
		INSERT 0 4
		```
	
14. Now drop the `REGION` table:
	
	=== "Input"
		```
		DROP TABLE REGION;
		```
	
	=== "Output"
		```
		DROP TABLE
		```
	
15. Finally, rename the `REGION_NEW` table to make the transformation
	    complete:
	
	=== "Input"
		```
		ALTER TABLE REGION_NEW RENAME TO REGION;
		```
	
	=== "Output"
		```
		ALTER TABLE
		```
	
	If a table can be inaccessible for a short period of time using CTAS
	tables can be the better solution to change data types than using an
	`ALTER TABLE` statement.
	
!!! success "Congratulations on finishing the chapter!"
	In this lab you have looked behind the scenes of the Netezza Performance
	Server appliances. You have seen how transactions are implemented and we
	have shown different reasons for using the groom command. It not only
	removes logically deleted rows from `INSERT` and `UPDATE` operations,
	aborted `INSERTS` and Loads, it also materializes table changes and
	reorders cluster-based tables.