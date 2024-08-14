# Database Administration

A factory-configured and installed IBM Netezza Performance Server will
include some of the following components:

-   An IBM Netezza Performance Server warehouse appliance with
    pre-installed IBM Netezza Performance Server software

-   A preconfigured Linux operating system (with IBM Netezza Performance
    Server modifications).

-   Several preconfigured Linux users and groups:


  |User      |Password    |Description
  |----------|------------|-----------
  |nz        |nz          | Linux user with login permission to the IBM Netezza Performance Server host container. nz has ownership of various IPS files in the installation directory.
  |root      |netezza     | Linux superuser with login permission to the IBM Netezza Performance Server host container. Has ownership of various kernel and operating system files.
  
-   An IBM Netezza Performance Server database user named ADMIN is the
    database super-user, and has full access to all system functions and
    objects

  |User      |Password    |Description
  |----------|------------|-----------
  |admin     |password    | IPS database superuser, with full access to all database administration privileges and objects
  
-   A preconfigured database group named `PUBLIC`. All database users are
    automatically placed in the group `PUBLIC` and therefore inherit all
    of its privileges

The IBM Netezza Performance Server warehouse appliance includes a highly
optimized SQL dialect called IBM Netezza Performance Server Structured
Query Language (SQL). You can use SQL commands to create and manage your
IBM Netezza Performance Server databases, user access, and permissions
for the databases, as well as to query and modify the contents of the
databases

On a new IBM Netezza Performance Server system, there is typically one
main database, `SYSTEM`, and a database template, `MASTER_DB`. IBM Netezza
Performance Server uses the `MASTER_DB` as a template for all other user
databases that are created on the system. Initially, only the `ADMIN` user
can create new databases, but the `ADMIN` user can grant other users'
permission to create databases as well. The `ADMIN` user can also make
another user the owner of a database, which gives that user `ADMIN`-like
control over that database and its contents. The database creator
becomes the default owner of the database. The owner can remove the
database and all its objects, even if other users own objects within the
database. Within a database, permitted users can create tables and
populate them with data and query its contents.

## 1 Objectives

This lab will guide you through the typical steps to create and manage
new IBM Netezza Performance Server users and groups after an IBM Netezza
Performance Server has been delivered and configured. This will include
creating a new database and assigning the appropriate privileges. The
users and the database that you create in this lab will be used as a
basis for the remaining labs in this bootcamp. After this lab you will
have a basic understanding on how to plan and create an IBM Netezza
Performance Server database environment.

-   The first part of this lab will examine creating IBM Netezza
    Performance Server users and groups

-   The second part of this lab will explore creating and using a
    database and tables. The table schema to be used within this
    bootcamp will be explained in the Data Distribution lab.

## 2 Lab Setup

-   Prepare for this lab by running the setup script. To do this use the
    following two commands:

	=== "Input"
		```
		cd ~/labs/databaseAdministration/setupLab
		./setupLab.sh
		```
	
	=== "Output"
		```
		drop database labdb;
		DROP DATABASE
		drop user labadmin;
		DROP USER
		drop user labuser;
		nzsql:drop/drop_users:2: ERROR:  DROP USER: object LABUSER does not exist.
		drop user dbuser;
		nzsql:drop/drop_users:3: ERROR:  DROP USER: object DBUSER does not exist.
		drop group lagrp;
		nzsql:drop/drop_groups:1: ERROR:  DROP GROUP: object LAGRP does not exist.
		drop group lugrp;
		nzsql:drop/drop_groups:2: ERROR:  DROP GROUP: object LUGRP does not exist.
		```
	
	Errors are expected if this is the first time running this lab on this
	virtual machine.

-   Connect to the system database as the IBM Netezza Performance Server
    database super-user, `ADMIN`, using the nzsql interface:

	=== "Input"
		```
		nzsql
		or
		nzsql -d system -u admin -pw password
		```
		
	=== "Output"
		```
		Welcome to nzsql, the Netezza SQL interactive terminal.
	
		Type: \h for help with SQL commands
		\? for help on internal slash commands
		\g or terminate with semicolon to execute query
		\q to quit
	
		SYSTEM(ADMIN)=>
		```	

There are different options you can use with the nzsql interface. Here
we present two options, where the first option uses information set in
the NZ environment variables, `NZ_DATABASE`, `NZ_USER`, and `NZ_PASSWORD`. You
can add these the environment variables (set in .bashrc) as follows:

=== "Set Environment Variables"
	```
	export NZ_DATATASE=system
	export NZ_USER=admin
	export NZ_PASSWORD=password
	```

Now you do not need to specify the database name or the user. In the
second option the information is explicitly stated using the `-d`,`-u`, and
`-pw` options, which specifies the database name, the user, and the user's
password, respectively. This option is useful when you want to connect
to a different database or use a different user than specified in the NZ
environment variables.
	
## 3 Creating IBM Performance server Users and Groups

The initial task after an IBM Netezza Performance Server has been set up
is to create the database environment. This typically begins by creating
a new set of database users and user groups before creating the
database. You will use the `ADMIN` user to start creating additional
database users and user groups. Then you will assign the appropriate
authorities after the database has been created in the next section. The
`ADMIN` user should only be used to perform administrative tasks within
the IBM Netezza Performance Server and is not recommended for regular
use. Also, it is highly advisable to develop a security access model to
control user access against the database and the database objects in an
IBM Netezza Performance Server. This will involve creating separate
users to perform certain tasks.

The security access model for this bootcamp environment will use three
IBM Netezza Performance Server database users and two groups:

Users:

-   LABADMIN
-   LABUSER
-   DBUSER

Groups:

-   LAGRP
-   LUGRP
-   Connect to the Netezza image using PuTTy or another terminal
    program. Login to `<VM-IP-ADDRESS>` as user nz with password nz.
    (192.168.9.2)

### 3.1 Creating New IBM Netezza Performance Server Users and Groups

The three new IBM Netezza Performance Server database users will be
initially created using the `ADMIN` user. The `LABADMIN` user will be the
full owner of the bootcamp database. The `LABUSER` user will be allowed to
perform data manipulation language (DML) operations (`INSERT`, `UPDATE`,
`DELETE`) against all of the tables in the database, but they will not be
allowed to create new objects like tables in the database. And lastly,
the `DBUSER` user will only be allowed to read tables in the database,
that is, they will only have `LIST` and `SELECT` privilege against tables in
the database.

The basic syntax to create a user is:

=== "CREATE USER Syntax"
	```
	CREATE USER username WITH PASSWORD 'string';
	```
	
1.  As the IBM Netezza Performance Server database super-user, `ADMIN`,
	    you can now start to create the first user, `LABADMIN`, which will be
	    the administrator of the database: (Note user and group names are
	    not case sensitive) unless surrounded by quotes.
	
	Note: The default case is Upper case, meaning all objects created will
	be created as upper case unless quoted, in which case the case will be
	maintained. The IPS system could be reinit with Lower case as the
	default.
	
	Creating New IBM Netezza Performance Server Users:
	
	=== "Input"
		```
		create user labadmin with password	'password';
		```
	
	=== "Output"
		```
		CREATE USER
		```
		
	Later in this lab you will assign administrative ownership of the lab
	database to this user.
	
2.  Now you will create two additional IBM Netezza Performance Server
	    database users that will have restricted access to the database.
	
	The first user, `LABUSER`, will have full DML access to the data in the
	tables, but will not be able to create or alter tables.
	 
	For now, you will just create the user. We will set the privileges
	after the database is created:
	 
	=== "Input"
		```
	 	create user labuser with password 'password';
		```
		
	=== "Output"
		```
		CREATE USER
		```
		
3.  Finally, we create the user `DBUSER`. This user will have even more
	    limited access to the database since it will only be allowed to
	    select data from the tables within the database. Again, you will set
	    the privileges after the database is created:
	
	=== "Input"
		```
		create user dbuser with password 'password';
		```
	
	=== "Output"
		```
		CREATE USER
		```
		
4.  To list the existing database users in the environment, use the `\du`
	    internal slash option:
	
	=== "Input"
		```
		\du
		```
		
	=== "Output"
		```
		                                                                        List of Users
		 USERNAME | VALIDUNTIL | ROWLIMIT | SESSIONTIMEOUT | QUERYTIMEOUT | DEF_PRIORITY | MAX_PRIORITY | USERESOURCEGRPID | USERESOURCEGRPNAME | CROSS_JOINS_ALLOWED
		----------+------------+----------+----------------+--------------+--------------+--------------+------------------+--------------------+---------------------
		 ADMIN    |            |          |              0 |            0 | NONE         | NONE         |                  | _ADMIN_            | NULL
		 DBUSER   |            |        0 |              0 |            0 | NONE         | NONE         |             4901 | PUBLIC             | NULL
		 LABADMIN |            |        0 |              0 |            0 | NONE         | NONE         |             4901 | PUBLIC             | NULL
		 LABUSER  |            |        0 |              0 |            0 | NONE         | NONE         |             4901 | PUBLIC             | NULL
		(4 rows)
		```
		
	The other columns are explained in the WLM presentation.

### 3.2 Creating New IBM Netezza Performance Server Groups

IBM Netezza Performance Server database user groups are useful for
organizing and managing database users. By default, IBM Netezza
Performance Server contains one group with the name `PUBLIC`. All users
are members in the `PUBLIC` group when they are created. Users can be
members of other groups as well. In this section we will create two new
IBM Netezza Performance Server database user groups. They will be
initially created by the `ADMIN` user.

We will create an administrative group `LAGRP` which is short for Lab
Admin Group. This group will contain the `LABADMIN` user. The second group
we create will be the `LUGRP` or Lab User Group. This group will contain
the users `LABUSER` and `DBUSER`.

Two different methods will be used to add the existing users to the
newly created groups. Alternatively, the groups could be created first
and then the users. The basic command to create a group is:

=== "CREATE GROUP Syntax"
	```
	CREATE GROUP groupname;
	```
	
1.  As the IBM Netezza Performance Server database super-user, `ADMIN`,
	    you will now create the first group, `LAGRP`, which will be the
	    administrative group for the `LABADMIN` user:
	
	=== "Input"
		```
		create group lagrp;
		```
		
	=== "Output"
		```
		CREATE GROUP
		```
		
2.  After the `LAGRP` group is created you will now add the `LABADMIN` user
	    to this group. This is accomplished by using the `ALTER` statement.
	    You can either `ALTER` the user or the group, for this task you will
	    `ALTER` the group to add the `LABADMIN` user to the `LAGRP` group:
	
	=== "Input"
		```
		alter group lagrp with add user labadmin;
		```
	
	=== "Output"
		```
		ALTER GROUP
		```
		
	To `ALTER` the user, you would use the following command:
	
	=== "Input"
		```
		alter user LABADMIN with in group LAGRP;
		```
	
3.  Now you will create the second group, `LUGRP`, which will be the user
	    group for both the `LABUSER` and `DBUSER` users. You can specify the
	    users to be included in the group when creating the group:
	
	=== "Input"
		```
		create group lugrp with add user labuser dbuser;
		```
	
	=== "Output"
		```
		CREATE GROUP
		```
		
4.  To list the existing IBM Netezza Performance Server groups in the
	    environment, use the `\dg` internal slash option:
	
	=== "Input"
		```
		\dg
		```
		
	=== "Output"
		```
																		  List of Groups
		 GROUPNAME | ROWLIMIT | SESSIONTIMEOUT | QUERYTIMEOUT | DEF_PRIORITY | 	MAX_PRIORITY | GRORSGPERCENT | RSGMAXPERCENT | JOBMAX | CROSS_JOINS_ALLOWED
		-----------+----------+----------------+--------------+--------------	+--------------+---------------+---------------+--------+---------------------
		 LAGRP     |        0 |              0 |            0 | NONE         | 	NONE         |             0 |           100 |      0 | NULL
		 LUGRP     |        0 |              0 |            0 | NONE         | 	NONE         |             0 |           100 |      0 | NULL
		 PUBLIC    |        0 |              0 |            0 | NONE         | 	NONE         |            20 |           100 |      0 | NULL
		(3 rows)
		```
	
	The default database group is `PUBLIC`. The other columns are explained
	in the WLM presentation.
	
5.  To list the users in a group you can use one of the two internal
	    slash options, `\dG`, or `\dU`. The internal slash option `\dG` will
	    list the groups with the associated users:
	
	=== "Input"
		```
		\dG
		```
	
	=== "Output"
		```
		List of Users in a Group
		 GROUPNAME | USERNAME
		-----------+----------
		 LAGRP     | LABADMIN
		 LUGRP     | DBUSER
		 LUGRP     | LABUSER
		 PUBLIC    | DBUSER
		 PUBLIC    | LABADMIN
		 PUBLIC    | LABUSER
		(6 rows)
	
		```
	
6.  The internal slash option `\dU` will list the users with the
	    associated group:
	
	=== "Input"
		```
		\dU
		```
		
	=== "Output"
		```
		List of Groups a User is a member
		 USERNAME | GROUPNAME
		----------+-----------
		 DBUSER   | LUGRP
		 DBUSER   | PUBLIC
		 LABADMIN | LAGRP
		 LABADMIN | PUBLIC
		 LABUSER  | LUGRP
		 LABUSER  | PUBLIC
		(6 rows)
		```
		
## 4 Creating an IBM Netezza Performance Server Database

The next step after the Netezza Performance Server database users and
user groups have been created is to create the lab database. You will
continue to use the `ADMIN` user to create the lab database then assign
the appropriate authorities and privileges to the users created in the
previous sections. The `ADMIN` user can also be used to create tables
within the new database. However, the `ADMIN` user should only be used to
perform administrative tasks. After the appropriate privileges have been
assigned by the `ADMIN` user, the database can be handed over to the
end-users to start creating and populating the tables in the database.

### 4.1 Creating a Database and Transferring Ownership

The lab database that will be created will be named `LABDB`. It will be
initially created by the `ADMIN` user and then ownership of the database
will be transferred to the `LABADMIN` user. The `LABADMIN` user will have
full administrative privileges against the `LABDB` database. The basic
syntax to create a database is:

=== "CREATE DATABASE Syntax"
	```
	CREATE DATABASE database_name;
	```
	
1.  As the Netezza Performance Server database super-user, `ADMIN`, you
	    will create the first database, `LABDB`, using the `CREATE DATABASE`
	    command:
	
	=== "Input"
		```
		create database labdb;
		```
		
	=== "Output"
		```
		CREATE DATABASE
		```
		
	The database `LABDB` has been created.
	
2.  To view the existing databases use the internal slash option `\l` :
	
	=== "Input"
		```
		\l
		```
		
	=== "Output"
		```
		List of databases
		 DATABASE | OWNER
		----------+-------
		 LABDB    | ADMIN
		 SYSTEM   | ADMIN
		(2 rows)
		```
		
	The owner of the newly created `LABDB` database is the `ADMIN` user. The
	other databases are the default database `SYSTEM` and the template
	database `MASTER_DB`.
	
3.  At this point you could continue by creating new tables as the `ADMIN`
	    user. However, the `ADMIN` user should only be used to create users,
	    groups, and databases, and to assign authorities and privileges.
	    Therefore, we will transfer ownership of the `LABDB` database from the
	    `ADMIN` user to the `LABADMIN` user we created previously. The `ALTER DATABASE` 
	    command is used to transfer ownership of an existing database:
	
	=== "Input"
		```
		alter database labdb owner to labadmin;
		```
		
	=== "Output"
		```
		ALTER DATABASE
		```
		
	This is the only method to transfer ownership of a database to an
	existing user. The `CREATE DATABASE` command does not include this
	option.
	
4.  Check that the owner of the `LABDB` database is now the `LABADMIN` user:
	
	=== "Input"
		```
		\l
		```
		
	=== "Output"
		```
		  List of databases
		 DATABASE |  OWNER
		----------+----------
		 LABDB    | LABADMIN
		 SYSTEM   | ADMIN
		(2 rows)
		```
		
	The owner of the `LABDB` database is now the `LABADMIN` user.
	
	The `LABDB` database is now created and the `LABADMIN` user has full
	privileges on the `LABDB` database. The user can create and alter objects
	within the database. You could now continue and start creating tables as
	the `LABADMIN` user. However, we will first finish assigning privileges to
	the two remaining database users that were created in the previous
	section.

### 4.2 Assigning Authorities and Privileges

One last task for the `ADMIN` user is to assign the privileges to the two
users we created earlier, `LABUSER` and `DBUSER`. `LABUSER` user will have
full DML rights against all tables in the `LABDB` database. It will not be
allowed to create or alter tables within the database. User `DBUSER` will
have more restricted access in the database and will only be allowed to
read data from the tables in the database. The privileges will be
controlled by a combination of setting the privileges at the group and
user level.

The `LUGRP` user group will be granted `LIST` and `SELECT` privileges against
the database and tables within the database. So any member of the `LUGRP`
will have these privileges. The full data manipulation privileges will
be granted individually to the `LABUSER` user.

The `GRANT` command that is used to assign object privileges has the
following syntax:

=== "GRANT Syntax"
	```
	GRANT object_privilege [, ...] ON object [, ...]
		TO { PUBLIC | GROUP group | username } [ WITH GRANT OPTION ]
	
	GRANT admin_privilege [, ...] [ IN { [database_name.]schema_name
	| database_name.ALL | ALL.ALL } ]
		TO { PUBLIC | GROUP group | username } [ WITH GRANT OPTION ]
	
	object_privilege
		ALL, ABORT, ALTER, DELETE, DROP, EXECUTE, EXECUTE AS,
		GENSTATS, GROOM, INSERT, LABEL ACCESS, LABEL RESTRICT, LABEL EXPAND,
		LIST, SELECT, TRUNCATE, UPDATE
	
	admin_privilege
	ALL ADMIN, BACKUP, [ CREATE ] DATABASE, [ CREATE ] GROUP,
	[ CREATE ] INDEX, [ CREATE ] SCHEMA, [ CREATE ] SEQUENCE, [
	CREATE ] SYNONYM,
	[ CREATE ] TABLE, [ CREATE ] EXTERNAL TABLE, [ CREATE ] TEMP
	TABLE,
	[ CREATE ] FUNCTION, [ CREATE ] AGGREGATE, [ CREATE ] USER,
	[ CREATE ] VIEW, [ CREATE ] MATERIALIZED VIEW, [CREATE] PROCEDURE,
	[ CREATE ] LIBRARY, [ CREATE ] SCHEDULER RULE, [ MANAGE ]
	HARDWARE,
	[ MANAGE ] SYSTEM, [ MANAGE ] SECURITY, RESTORE, UNFENCE, VACUUM
	```
	
1.  As the Netezza Performance Server database super-user, `ADMIN`,
	    connect to the `LABDB` database using the internal slash option `\c`
	
	=== "Input"
		```
		\c labdb admin password
		```
		
	=== "Output"
		```
		You are now connected to database labdb as user admin.
		```
	
	You will notice that the database name in command prompt has changed
	from SYSTEM to `LABDB`.
	
2.  First you will grant LIST privilege on the `LABDB` database to the
	    group `LUGRP`. This will allow members of the `LUGRP` to view and
	    connect to the `LABDB` database :
	
	=== "Input"
		```
		grant list on labdb to lugrp;
		```
		
	=== "Output"
		```
		GRANT
		```
		
3. To list the object permissions for a group use the following
	    internal slash option, `\dpg` :
	
	=== "Input"
		```
		\dpg lugrp
		```
		
	=== "Output"
		```
		                                            Group object permissions for group 'LUGRP'
		 Database Name | Schema Name | Object Name | L S I U D T L A D B L G O E C R X A | D G U S T E X Q Y V M I B R C S H F A L P N S R
		---------------+-------------+-------------+-------------------------------------+-------------------------------------------------
		 GLOBAL        | GLOBAL      | LABDB       | X                                   |
		(1 rows)
		Object Privileges
		    (L)ist (S)elect (I)nsert (U)pdate (D)elete (T)runcate (L)ock
		    (A)lter (D)rop a(B)ort (L)oad (G)enstats Gr(O)om (E)xecute
		    Label-A(C)cess Label-(R)estrict Label-E(X)pand Execute-(A)s
		Administration Privilege
		    (D)atabase (G)roup (U)ser (S)chema (T)able T(E)mp E(X)ternal
		    Se(Q)uence S(Y)nonym (V)iew (M)aterialized View (I)ndex (B)ackup
		    (R)estore va(C)uum (S)ystem (H)ardware (F)unction (A)ggregate
		    (L)ibrary (P)rocedure U(N)fence (S)ecurity Scheduler (R)ule
		```
	
	The `X` in the L column of the list denotes that the `LUGRP` group has
	`LIST` object privileges on the `LABDB` global object.
	
4. With the current privileges set for the `LABUSER` and `DBUSER`, they can
	    now view and connect to the `LABDB` database as members of the `LUGRP`
	    group. But these two users have no privileges to access any of the
	    objects within the database. So you will grant `LIST` and `SELECT`
	    privilege to the tables within the `LABDB` database to the members of
	    the `LUGRP` :
	
	=== "Input"
		```
		grant list, select on table to lugrp;
		```
	
	=== "Output"
		```
		GRANT
		```
		
5. View the object permissions for the `LUGRP` group :
	
	=== "Input"
		```
		\dpg lugrp
		```
		
	=== "Output"
		```
		                                            Group object permissions for group 'LUGRP'
		 Database Name | Schema Name | Object Name | L S I U D T L A D B L G O E C R X A | D G U S T E X Q Y V M I B R C S H F A L P N S R
		---------------+-------------+-------------+-------------------------------------+-------------------------------------------------
		 GLOBAL        | GLOBAL      | LABDB       | X                                   |
		 LABDB         | ADMIN       | TABLE       | X X                                 |
		(2 rows)
		Object Privileges
		    (L)ist (S)elect (I)nsert (U)pdate (D)elete (T)runcate (L)ock
		    (A)lter (D)rop a(B)ort (L)oad (G)enstats Gr(O)om (E)xecute
		    Label-A(C)cess Label-(R)estrict Label-E(X)pand Execute-(A)s
		Administration Privilege
		    (D)atabase (G)roup (U)ser (S)chema (T)able T(E)mp E(X)ternal
		    Se(Q)uence S(Y)nonym (V)iew (M)aterialized View (I)ndex (B)ackup
		    (R)estore va(C)uum (S)ystem (H)ardware (F)unction (A)ggregate
		    (L)ibrary (P)rocedure U(N)fence (S)ecurity Scheduler (R)ule
		```
		
	
	The X in the L and S column denotes that the `LUGRP` group has both `LIST`
	and `SELECT` privileges on all of the tables in the `LABDB` database. (The
	`LIST` privilege is used to allow users to view the tables using the
	internal slash opton `\d`.)
	
6.  The current privileges satisfy the `DBUSER` user requirements, which
	    is to allow access to the `LABDB` database and SELECT access to all
	    the tables in the database. But these privileges do not satisfy the
	    requirements for the `LABUSER` user. The `LABUSER` user is to have full
	    DML access to all the tables in the database. So you will grant
	    `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `LIST`, and `TRUNCATE` privileges on
	    tables in the `LABDB` database to the `LABUSER` user:
	
	=== "Input"
		```
		grant select, insert, update, delete, list, truncate on table to labuser;
		```
	
	=== "Output"
		```
		GRANT
		```
	
7. To list the object permissions for a user use the `\dpu <user
	    name>` internal slash option,:
	
	=== "Input"
		```
		\dpu labuser
		```
		
	=== "Output"
		```
		                                           User object permissions for user 'LABUSER'
		 Database Name | Schema Name | Object Name | L S I U D T L A D B L G O E C R X A | D G U S T E X Q Y V M I B R C S H F A L P N S R
		---------------+-------------+-------------+-------------------------------------+-------------------------------------------------
		 LABDB         | ADMIN       | TABLE       | X X X X X X                         |
		(1 rows)
		Object Privileges
		    (L)ist (S)elect (I)nsert (U)pdate (D)elete (T)runcate (L)ock
		    (A)lter (D)rop a(B)ort (L)oad (G)enstats Gr(O)om (E)xecute
		    Label-A(C)cess Label-(R)estrict Label-E(X)pand Execute-(A)s
		Administration Privilege
		    (D)atabase (G)roup (U)ser (S)chema (T)able T(E)mp E(X)ternal
		    Se(Q)uence S(Y)nonym (V)iew (M)aterialized View (I)ndex (B)ackup
		    (R)estore va(C)uum (S)ystem (H)ardware (F)unction (A)ggregate
		    (L)ibrary (P)rocedure U(N)fence (S)ecurity Scheduler (R)ule
		```
	
	The X under the L, S, I, U, D, T columns indicates that the `LABUSER`
	user has `LIST`, `SELECT`, `INSERT`, `UPDATE`, `DELETE`, and `TRUNCATE` privileges
	on all of the tables in the `LABDB` database.
	
	Now that all of the privileges have been set by the `ADMIN` user the `LABDB`
	database can be handed over to the end-users. The end-users can use the
	`LABADMIN` user to create objects, which include tables, in the database
	and also maintain the database.
	
## 5 Creating Tables

The `LABADMIN` user will be used to create tables in the `LABDB` database
instead of the `ADMIN` user. Two tables will be created in this lab. The
remaining tables for the `LABDB` database schema will be created in the
Data Distribution lab. Data Distribution is an important aspect that
should be considered when creating tables. This concept is not covered
in this lab since it is discussed separately in the Data Distribution
presentation. The two tables that will be created are the `REGION` and
`NATION` tables. These two tables will be populated with data in the next
section using `LABUSER` user. Two methods will be utilized to create these
tables. The basic syntax to create a table is:

=== "CREATE TABLE Syntax"
	```
	CREATE [ TEMPORARY | TEMP ] TABLE [IF NOT EXISTS] <table>
	( <col> <type> [<col_constraint>][,<col> <type>
	[<col_constraint>]...]
	[<table_constraint>[,<table_constraint>... ] )
	[ DISTRIBUTE ON { RANDOM | [HASH] (<col>[,<col>...]) } ]
	
	[ ORGANIZE ON { (<col>) | NONE } ]
	
	[ ROW SECURITY ]
	```
	
=== "Column Constraints"
	```
	CREATE [ TEMPORARY | TEMP ] TABLE [IF NOT EXISTS] <table>
	( <col> <type> [<col_constraint>][,<col> <type> [<col_constraint>]…]
	[<table_constraint>[,<table_constraint>… ] )
	[ DISTRIBUTE ON { RANDOM | [HASH] (<col>[,<col>…]) } ]
	[ ORGANIZE ON { (<col>) | NONE } ]
	[ ROW SECURITY ]
	```
	
=== "Table Constraints"
	```
	[ CONSTRAINT <constraint_name> ] 
	{NOT NULL | NULL | UNIQUE | PRIMARY KEY | DEFAULT <value> | <ref>}
	[ [ [ NOT ] DEFERRABLE ] { INITIALLY DEFERRED | INITIALLY IMMEDIATE } |
	[ INITIALLY DEFERRED | INITIALLY IMMEDIATE ] [ NOT ] DEFERRABLE ]
	```
=== "REF Constraints"
	```
	[ CONSTRAINT <constraint_name> ] 
	{UNIQUE (<col>[,<col>…] ) |
	PRIMARY KEY (<pkcol_name>[,<pkcol_name>…] ) |
	FOREIGN KEY (<fkcol_name>[,<fkcol_name>…] ) <ref>}
	[ [ [ NOT ] DEFERRABLE ] { INITIALLY DEFERRED | INITIALLY IMMEDIATE } |
	[ INITIALLY DEFERRED | INITIALLY IMMEDIATE ] [ NOT ] DEFERRABLE ]
	```
	
1.  Connect to the `LABDB` database as the `LABADMIN` user using the
	    internal slash option `\c`:
	
	=== "Input"
		```
		\c labdb labadmin password
		```
		
	=== "Output"
		```
		You are now connected to database labdb as user labadmin.
		```
	
	You will notice that the username in the command prompt has changed
	from `ADMIN` to `LABADMIN`. Since you already had an opened session, you
	could use the internal slash option `\c` to connect to the database.
	However, if you had handed over this environment to the end user they
	would need to initiate a new connection using the nzsql interface.
	 
	To use the nzsql interface to connect to the `LABDB` database as the
	`LABADMIN` user you could use the following options (no need to run
	these command, informational):
	```
	nzsql -d labdb -u labadmin -pw password
	```
	or with the short form, omitting the options:
	```
	nzsql labdb labadmin password
	```
	or you could set the environment variables to the following values and
	issue nzsql without options.
	```
	export NZ_DATABASE=LABDB
	export NZ_USER=LABADMIN
	export NZ_PASSWORD=password
	```
	
	In further labs we will often leave out the password parameter since
	it has been set to the same value "password" for all users.
	
2. Now you can create the first table in the `LABDB` database. The first
	    table you will create is the `REGION` table with the following columns
	    and datatypes :
	
	| **Column Name**           | **Data Type**                            |
	|---------------------------|------------------------------------------|
	| R_REGIONKEY               | INTEGER                                  |
	| R_NAME                    | CHAR(25)                                 |
	| R_COMMENT                 | VARCHAR(152)                             |
	
	To create the above table execute the following command:
	 
	=== "Input"
		```
		create table region (
			r_regionkey integer,
			r_name char(25), 
			r_comment varchar(152)
		);
		```
	
	=== "Output"
		```
		CREATE TABLE
		```
		
3. To list the tables in the `LABDB` database use the `\dt` internal slash
	    option:
	
	=== "Input"
		```
		\dt
		```
		
	=== "Output"
		```
		         List of relations
		 Schema |  Name  | Type  |  Owner
		--------+--------+-------+----------
		 ADMIN  | REGION | TABLE | LABADMIN
		(1 row)
		```
		
	
4. To describe a table you can use the internal slash option `\d
	    <table name>`:
	
	=== "Input"
		```
		\d region
		```
		
	=== "Output"
		```
		                        Table "REGION"
		  Attribute  |          Type          | Modifier | Default Value
		-------------+------------------------+----------+---------------
		 R_REGIONKEY | INTEGER                |          |
		 R_NAME      | CHARACTER(25)          |          |
		 R_COMMENT   | CHARACTER VARYING(152) |          |
		Distributed on hash: "R_REGIONKEY"
		```
	
	The `Distributed on hash` clause is the distribution method used by the
	table. If you do not explicitly specify a distribution method, a
	default distribution is used. In our system this is hash distributed
	on the first column `R_REGIONKEY`. This concept is discussed in the Data
	Distribution presentation and lab.
	 
	Note: the default distribution can be change to `RANDOM`. This might be
	a preferred default distribution when the Netezza environment is
	supporting ad-hoc user table creation. In the event a user forgets to
	add a distribution key, `RANDOM` would at a minimum guarantee even
	distribution. However, you table if joined to other tables would
	require data movement to perform the join.
	
5. Instead of typing out the entire create table statement at the nzsql
	    command line you can read and execute commands from a file. You'll
	    use this method to create the `NATION` table in the `LABDB` database
	    with the following columns and data types:
	
	  |Column Name                |Data Type                  | Constraint
	  |---------------------------|---------------------------|-----------------
	  |N_NATIONKEY                |INTEGER                    |NOT NULL
	  |N_NAME                     |CHAR(25)                   |NOT NULL
	  |N_REGIONKEY                |INTEGER                    |NOT NULL
	  |N_COMMENT                  |VARCHAR(152)               |---
	
	The full create table statement for the NATION table:
	```
	create table nation (
	  n_nationkey integer not null,
	  n_name char(25) not null,
	  n_regionkey integer not null,
	  n_comment varchar(152)
	)
	distribute on random;
	```
	
6. The statement can be found in the nation.ddl file under the
	    `/home/nz/labs/databaseAdministration` directory. To read and execute
	    commands from a file use the `\i <*file*>` internal slash option:
	
	=== "Input"
		```
		\i /home/nz/labs/databaseAdministration/nation.ddl
		```
	
	=== "Output"
		```
		CREATE TABLE
		```
		
7. You can run Linux commands while in the nzsql console. For example:
	    list the nation.ddl file
	
	=== "Input"
		```
		! ls -l /home/nz/labs/databaseAdministration/nation.ddl
		```
	
	=== "Output"
		```
		-rwxr-xr-x. 1 nz nz 176 May 13 2020	/home/nz/labs/databaseAdministration/nation.ddl
		```
	
8. List all the tables in the `LABDB` database:
	
	=== "Input"
		```
		\dt
		```
		
	=== "Output"
		```
		         List of relations
		 Schema |  Name  | Type  |  Owner
		--------+--------+-------+----------
		 ADMIN  | NATION | TABLE | LABADMIN
		 ADMIN  | REGION | TABLE | LABADMIN
		(2 rows)
		```
		
	
9. Describe the NATION table :
	
	=== "Input"
		```
		\d nation
		```
		
	=== "Output"
		```
		                         Table "NATION"
		  Attribute  |          Type          | Modifier | Default Value
		-------------+------------------------+----------+---------------
		 N_NATIONKEY | INTEGER                | NOT NULL |
		 N_NAME      | CHARACTER(25)          | NOT NULL |
		 N_REGIONKEY | INTEGER                | NOT NULL |
		 N_COMMENT   | CHARACTER VARYING(152) |          |
		Distributed on random: (round-robin)
		```
	
	The distributed on random is the distribution method used, in this
	case the rows in the NATION table are distributed in round-robin
	fashion. This concept will be discussed separately in the Data
	Distribution presentation and lab.
	
	It is possible to continue to use `LABADMIN` user to perform DML queries
	since it is the owner of the database and holds all privileges on all of
	the objects in the databases. However, the `LABUSER` and `DBUSER` users will
	be used to perform DML queries against the tables in the database.

## 6 Using DML Queries

We will now use the `LABUSER` user to populate data into both the REGION
and NATION tables. This user has full data manipulation language (DML)
privileges in the database, but no data definition language (DDL)
privileges. Only the `LABADMIN` has full DDL privileges in the database.
Later in this course more efficient methods to populate tables with data
are discussed. The `DBUSER` will also be used to read data from the
tables, but it can not insert data into the tables since it has limited
DML privileges in the database.
	
1.  Connect to the `LABDB` database as the `LABUSER` user using the internal
	    slash option, `\c`:
	
	=== "Input"
		```
		\c labdb labuser password
		```
		
	=== "Output"
		```
		You are now connected to database LABDB as user labuser.
		
		LABDB.ADMIN(LABUSER)=>
		```
	
	You will notice that the user name in the command prompt has changed
	from `LABADMIN` to `LABUSER`.
	
2. First check which tables exist in the `LABDB` database using the `\dt`
	    internal slash option:
	
	=== "Input"
		```
		\dt
		```
		
	=== "Output"
		```
		        List of relations
		 Schema |  Name  | Type  |  Owner
		--------+--------+-------+----------
		 ADMIN  | NATION | TABLE | LABADMIN
		 ADMIN  | REGION | TABLE | LABADMIN
		(2 rows)
		```
		
	Remember that the `LABUSER` user is a member of the `LUGRP` group which
	was granted LIST privileges on the tables in the `LABDB` database. This
	is the reason why it can list and view the tables in the `LABDB`
	database. If it did not have this privilege, it would not be able to
	see any of the tables in the `LABDB` database.
	
3. The `LABUSER` user was created to perform DML operations against the
	    tables in the `LABDB` database. However, it was restricted on
	    performing DDL operations against the database. Let's see what
	    happens when you try create a new table, t1, with one column, C1,
	    using the `INTEGER` data type:
	
	=== "Input"
		```
		create table t1 (c1 integer);
		```
	
	=== "Output"
		```
		ERROR: CREATE TABLE: permission denied.
		```
	
	As expected, the create table statement is not allowed since `LABUSER`
	user does not have the privilege to create tables in the `LABDB`
	database.
	
4. Let's continue by performing DML operations that the `LABUSER` user is
	    allowed to perform against the tables in the `LABDB` database. Insert
	    a new row into the `REGION` table:
	
	=== "Input"
		```
		insert into region values (1, 'NA', 'north america');
	
	=== "Output"
		```
		INSERT 0 1
		```
	
	As expected, this operation is successful. The output of the `INSERT`
	gives feedback about the number of successfully inserted rows.
	
5. Issue the `SELECT` statement against the `REGION` table to check the new
	    row you just added to the table:
	
	=== "Input"
		```
		select * from region;
		```
		
	=== "Output"
		```
		 R_REGIONKEY |          R_NAME           |   R_COMMENT
		-------------+---------------------------+---------------
		           1 | NA                        | north america
		(1 row)
		```
		
	
6. Instead of typing DML statements at the nzsql command line, you can
	    read and execute statements from a file. You will use this method to
	    add the following three rows to the `REGION` table:
	
	| R_REGIONKEY      | R_NAME         | R_COMMENT                        |
	|------------------|----------------|----------------------------------
	| 2                | SA             | South America                    |
	| 3                | EMEA           | Europe, Middle East, Africa      |
	| 4                | AP             | Asia Pacific                     |
	
	This is done with a SQL script containing the following commands:
	```
	insert into region values (2, 'sa', 'south america');
	insert into region values (3, 'emea', 'europe, middle east, africa'); 
	insert into region values (4, 'ap', 'asia pacific');
	```
	
	It can be found in the region.dml file under the
	`/home/nz/labs/databaseAdministration` directory. To read and execute
	commands from a file use the `\i <*file*>` internal slash option:
	 
	=== "Input"
		```
		\i /home/nz/labs/databaseAdministration/region.dml
		```
		
	=== "Output"
		```
		INSERT 0 1
		INSERT 0 1
		INSERT 0 1
		```
	
	You can see from the output that the SQL script contained three `INSERT`
	statements.
	
7. You will load data into the `NATION` table using an external table
	    with the following command:
	
	=== "Input"
		```
		insert into nation select * from external
		'/home/nz/labs/databaseAdministration/nation.del';
		```
		
	=== "Output"
		```
		INSERT 0 14
		```
		
	You loaded 14 rows into the table.
	 
	Loading data into a table is covered in the Loading and Unloading Data
	presentation and lab.
	
8. Now you will switch over to the `DBUSER` user, who only has SELECT
	    privilege on the tables in the `LABDB` database. This privilege is
	    granted to this user since he is a member of the `LUGRP` group. Use
	    the internal slash option, `\c <database name> <user>
	    <password>` to connect to the `LABDB` database as the `DBUSER` user:
	
	=== "Input"
		```
		\c labdb dbuser password
		```
		
	=== "Output"
		```
		You are now connected to database labdb as user dbuser.
		```
		
	You will notice that the username in the command prompt changes from
	`LABUSER` to `DBUSER`.
	
9. Before trying to view rows from tables in the `LABDB` database, try to
	    add a new row to the `REGION` table:
	
	=== "Input"
		```
		insert into region values (5, 'NP', 'north pole');
		```
	
	=== "Output"
		```
		ERROR: Permission denied on "REGION".
		```
		
	As expected, the `INSERT` statement is not allowed since the `DBUSER` does
	not have the privilege to add rows to any tables in the `LABDB`
	database.
	
10. Now select all rows from the `REGION` table:
	
	=== "Input"
		```
		select * from region;
		```
		
	=== "Output"
		```
		 R_REGIONKEY |          R_NAME           |          R_COMMENT
		-------------+---------------------------+-----------------------------
		           1 | NA                        | north america
		           2 | sa                        | south america
		           3 | emea                      | europe, middle east, africa
		           4 | ap                        | asia pacific
		(4 rows)
		```
		
	
11. Finally let's run a slightly more complex query. We want to return
	    all nation names in Asia Pacific, together with their region name.
	    To do this you need to execute a simple join using the `NATION` and
	    `REGION` tables. The join key will be the region key, and to restrict
	    results on the AP region you need to add a `WHERE` condition:
	
	=== "Input"
		```
		select n_name, r_name from nation, region where
			n_regionkey = r_regionkey and r_name = 'ap';
		```
	
	=== "Output"
		```
		          N_NAME           |          R_NAME
		---------------------------+---------------------------
		 australia                 | ap
		 japan                     | ap
		 macau                     | ap
		 hong kong                 | ap
		 new zealand               | ap
		(5 rows)
		```
	
	This should return a result containing all countries
	from the ap region.

## 7 Roles

IBM Netezza Performance Server introduced a new object called role. A
role is a potential grantee or grantor of privileges and of other roles.
Similar to a user, a role can own schemas and other objects. Roles can
own database objects such as tables or functions.

By using roles, you can also assign privileges. You can create roles
that are available across all databases on the IBM Netezza Performance
Server system. Similar to group objects, roles are also **global
objects**.

A user or group can be grant permission to a role. A session can then be
set to run under a ROLE's authorization. Once a session is set to a
ROLE the permissions used are from that ROLE's authorization and/or
privileges, not a user or group permissions. This allows a
user/application to isolate a database session to execute with a ROLE's
permission.

In this section of the lab we will create roles to demonstrate how roles
can be used.

#### 7.1.1 Creating roles

Roles are create using the following syntax:

=== "CREATE ROLE Syntax"
	```
	CREATE ROLE role name
	[ WITH ADMIN owner name ]
	[ AS ADMIN ]
	```

1. Login to the NPS command line as the nz user.


2. As the NPS database super-user, `ADMIN`, use `nzsql` to create the first
    role, `LAROLE`, which will be the administrator of the system: (Note:
    roles are not case sensitive)

	=== "Input"
		```
		create role DBAROLE AS ADMIN;
		```
	
	=== "Output"
		```
		CREATE ROLE
		```
		
	This role will act as the super-user `ADMIN` and have all permissions on
	the database system. This is a simple way to give `ADMIN` rights to a user
	through the use of a role.
	
#### 7.1.2 Display roles

3. Display the newly created role with the following command:

	=== "Input"
		```
		show role;
		```
		
	=== "Output"
		```
		ROLENAME | ROLEGRANTOR | ASADMIN
		----------+-------------+---------
		 DBAROLE  | ADMIN       | t
		(1 row)
		```
		
#### 7.1.3 Create a new user without any permissions

4. Create a new DBA user without any permissions.

	=== "Input"
		```
		create user DBAUSER password 'password';
		```
	
	=== "Output"
		```
		CREATE USER
		```
		
5. Connect to the system database as `DBAUSER` and `CREATE` a database:

	=== "Connect"
		```
		\c system dbauser password
		```
		
	=== "Output"
		```
		You are now connected to database system as user dbauser.
		```
		
	=== "Create Database"
		```
		create database testdb;
		```
		
	=== "Output"
		```
		ERROR: CREATE DATABASE: permission denied.
		```
		
	Notice that `DBUSER` cannot create a database as is expected since we
	didn't grant `DATABASE` permission to the `DBUSER`.

6. Connect to the system database as `ADMIN`:
	
	=== "Input"
		```
		\c system
		```
		
	=== "Output"
		```
		You are now connected to database system.
		```
	
#### 7.1.4 Grant permission to use a role


7. Grant `LIST` on `DBAROLE` to `DBAUSER`:
	
	=== "Input"
		```
		grant list on dbarole to DBAUSER;
		```
		
	=== "Output"
		```
		GRANT
		```
	
	This gives the `DBAUSER` permission to use the role `DBAROLE` which has
	`ADMIN` rights.

8. Connect to the system database as DBAUSER:
	
	=== "Input"
		```
		\c system dbauser password
		```
		
	=== "Output"
		```
		You are now connected to database system as user dbauser.
		```
	
9. List the roles as `DBUSER`:

	=== "Input"
		```
		show role;
		```
		
	=== "Output"
		```
		ROLENAME | ROLEGRANTOR | ASADMIN
		----------+-------------+---------
		 DBAROLE  | ADMIN       | t
		(1 row)
		```
		

	The `LIST` permission gives the `DBAUSER` the ability to show and set roles.

10. As the `DBUSER` set the session to the role `DBAROLE`:

	=== "Input"
		```
		set role dbarole;
		```
		
	=== "Output"
		```
		SET ROLE
		```
		
	All commands will run with `ADMIN` rights, thus, full control over the
	database system.
	
11. Create a database with the `DBAROLE` set:

	=== "Input"
		```
		create database testdb;
		```
		
	=== "Output"
		```
		CREATE DATABASE
		```
		
12. List the database with the slash: option `\l`:

	=== "Input"
		```
		\l
		```
		
	=== "Output"
		```
		List of databases
		 DATABASE | OWNER
		----------+-------
		 LABDB    | ADMIN
		 SYSTEM   | ADMIN
		 TESTDB   | ADMIN
		(3 rows)
		```
		
	Notice the owner of the `TESTDB` database is `ADMIN`. This is because the
	role `DBAROLE` (current session set to `DBAROLE`) was created with the `AS ADMIN` option.
	
13. Drop the database `TESTDB`:

	=== "Input"
		```
		drop database testdb;
		```
		
	=== "Output"
		```
		DROP DATABASE
		```
		
#### 7.1.5 Reset session from a role

14. Return your nzsql session to the current user with the following
    command:
	```
	set role none;
	```
	
	=== "Input"
		```
		set role none;
		```
		
	=== "Output"
		```
		SET ROLE
		```
		
15. You are now back to using the permission of `DBAUSER`. Try creating
    the `TESTDB` database again and see what happens:

	=== "Input"
		```
		create database testdb;
		```
		
	=== "Output"
		```
		ERROR: CREATE DATABASE: permission denied.
		```
		
	The command was run with the `DBAUSER` privileges and the `DBAUSER` doesn't
	have permissions to create a database.
	
#### 7.1.6 Create a second user

16. Connect to the system database as `ADMIN`:
	
	=== "Input"
		```
		\c system
		```
		
	=== "Output"
		```
		You are now connected to database system.
		```
		
17. Create a second user called `DBOPS`:

	=== "Input"
		```
		create user dbops with password 'password';
		```
		
	=== "Output"
		```
		CREATE USER
		```
			
#### 7.1.7 Create a new role without permissions
	
	
18. Create a new role called `DBOPSROLE`:
	
	=== "Input"
		```
		create role dbopsrole with admin dbops;
		```
		
	=== "Output"
		```
		CREATE ROLE
		```
		
19. Connect to the system database as the `LABADMIN` user:
	
	=== "Input"
		```
		\c system dbops password
		```
		
	=== "Output"
		```
		You are now connected to database system as user dbops.
		```
		
20. Show the role:
	
	=== "Input"
		```
		show role;
		```
		
	=== "Output"
		```
		 ROLENAME  | ROLEGRANTOR | ASADMIN
		-----------+-------------+---------
		 DBOPSROLE | DBOPS       | f
		(1 row)
		```
	
	`DBOPS` can see the `DBOPSROLE` because the role was created with `DBOPS` user
	as the admin. `DBOPS` user is the `GRANTOR` for the role.
	
21. Set the session role to `DBOPSROLE` as the `DBOPS` user:
	
	=== "Input"
		```
		set role dbopsrole;
		```
		
	=== "Output"
		```
		SET ROLE
		```
		
22. Create a database called `TESTDB`:
	
	=== "Input"
		```
		create database testdb;
		```
		
	=== "Output"
		```
		ERROR: CREATE DATABASE: permission denied.
		```
		
	The `CREATE DATABASE` failed because the `DBOPS` user does not have any
	permissions on the database system other than the `ADMIN` of the role
	`DBOPSROLE` (i.e.: drop).
	
	After creating a role that is not defined with the `AS ADMIN` option you
	will need to `GRANT` the appropriate permissions similar to how you grant
	permissions to users and groups.
	
You completed the section on Users, Groups and Roles. You should now
understand how Users, Groups and Roles are all related and used.

!!! success "Congratulations on finishing the chapter!"
	Congratulations you have completed the lab. You have successfully
	created the lab database, 2 tables, and database users, groups and roles
	with various privileges. You also ran a couple of simple queries.
