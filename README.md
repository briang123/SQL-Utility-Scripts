# T-SQL Utility Scripts

## Scripts were created for SQL Server 2005

### Granting Permissions
### ADD User to database if user does not exist
```sql
DECLARE @search varchar(200), 
	@searchCount int, 
	@name varchar(200),
	@user varchar(50),
	@grantLogin varchar(128),
	@grantDbAccess varchar(128)

SELECT 	@user = QUOTENAME('<user>\ASPNET')
SELECT 	@grantLogin = 'EXEC sp_grantlogin ' + @user
SELECT 	@grantDbAccess = 'EXEC sp_grantdbaccess ' + @user

USE '<database-name>'

IF NOT EXISTS (SELECT * FROM master.dbo.syslogins WHERE loginname = @user)
BEGIN
	PRINT ''
	PRINT 'GRANTING SQL LOGIN'
	EXEC(@grantLogin)
END

IF NOT EXISTS (SELECT * FROM dbo.sysusers WHERE name = @user and uid < 16382)
BEGIN
	PRINT ''
	PRINT 'GRANTING DATABASE ACCESS'
	EXEC(@grantDbAccess)
END
```

### Stored Procedure Reference Count 
*Total number of stored procedures for each table that reference it*
```sql
DECLARE @search varchar(200)
DECLARE @searchCount int
DECLARE @name varchar(200)

DECLARE perm_cursor CURSOR FOR

SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES 
WHERE TABLE_NAME LIKE 'tbl%' ORDER BY TABLE_NAME ASC

OPEN perm_cursor
FETCH NEXT FROM perm_cursor
INTO @name

WHILE @@FETCH_STATUS = 0
BEGIN

	SELECT 	@search = '%' + @name + '%'
	SELECT 	@searchCount = COUNT(*)
	FROM 	INFORMATION_SCHEMA.ROUTINES
	WHERE 	ROUTINE_DEFINITION LIKE @search

	PRINT 'COUNT OF STORED PROCEDURES REFERENCING ' + QUOTENAME(@name) + ':' + STR(@searchCount)
	
   FETCH NEXT FROM perm_cursor
   INTO @name
END

CLOSE perm_cursor
DEALLOCATE perm_cursor
```

### Table Reference list within Stored Procedures 
*List of stored procedures by table name that reference table*
```sql
DECLARE @search varchar(200)
DECLARE @name varchar(200)

DECLARE perm_cursor CURSOR FOR

SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES 
WHERE TABLE_NAME LIKE 'tbl%' ORDER BY TABLE_NAME ASC

OPEN perm_cursor
FETCH NEXT FROM perm_cursor INTO @name

WHILE @@FETCH_STATUS = 0
BEGIN

	PRINT ''
	PRINT @name
	PRINT '-----------------------------'

	DECLARE @procName varchar(100), 
		@proc_cursor CURSOR

	SELECT @search = '%' + @name + '%'
	SET @proc_cursor = CURSOR FOR
		SELECT ROUTINE_NAME FROM INFORMATION_SCHEMA.ROUTINES 
		WHERE ROUTINE_DEFINITION LIKE @search

	OPEN @proc_cursor
	FETCH NEXT FROM @proc_cursor INTO @procName

	WHILE @@FETCH_STATUS = 0
	BEGIN

		PRINT @procName

		FETCH NEXT FROM @proc_cursor
		INTO @procName
	END

	CLOSE @proc_cursor
	DEALLOCATE @proc_cursor

    PRINT ''
    FETCH NEXT FROM perm_cursor
    INTO @name
END

CLOSE perm_cursor
DEALLOCATE perm_cursor
```

### Function Reference list within Stored Procedures 
*List of stored procedures by function name that reference function*
```sql
DECLARE @search varchar(200)
DECLARE @name varchar(200)

DECLARE perm_cursor CURSOR FOR

-- get all functions to set permissions for
SELECT name FROM SYSOBJECTS 
WHERE name LIKE 'fn__%' AND type IN ('FN','TF') 
ORDER BY NAME ASC

OPEN perm_cursor
FETCH NEXT FROM perm_cursor INTO @name

WHILE @@FETCH_STATUS = 0
BEGIN

	PRINT ''
	PRINT @name
	PRINT '-----------------------------'

	DECLARE @funcName varchar(100), 
		@func_cursor CURSOR

	SELECT @search = '%' + @name + '%'
	SET @func_cursor = CURSOR FOR
		SELECT ROUTINE_NAME FROM INFORMATION_SCHEMA.ROUTINES 
		WHERE ROUTINE_DEFINITION LIKE @search

	OPEN @func_cursor
	FETCH NEXT FROM @func_cursor INTO @funcName

	WHILE @@FETCH_STATUS = 0
	BEGIN

		IF (@funcName <> @name)
		BEGIN
			PRINT @funcName
		END

		FETCH NEXT FROM @func_cursor
		INTO @funcName
	END

	CLOSE @func_cursor
	DEALLOCATE @func_cursor

	PRINT ''
    FETCH NEXT FROM perm_cursor
    INTO @name
END

CLOSE perm_cursor
DEALLOCATE perm_cursor
```

### GRANT EXECUTE permissions on Stored Procedures 
*GRANTs permissions to EXECUTE stored procedures from .NET app*
```sql
DECLARE @user varchar(50)
SELECT 	@user = QUOTENAME(@@servername + '\ASPNET')
DECLARE @name varchar(200)

DECLARE perm_cursor CURSOR FOR

-- get all stored procedures to set permissions for
SELECT ROUTINE_NAME FROM INFORMATION_SCHEMA.ROUTINES 
WHERE ROUTINE_NAME LIKE 'sp__%' ORDER BY ROUTINE_NAME ASC

OPEN perm_cursor
FETCH NEXT FROM perm_cursor INTO @name

WHILE @@FETCH_STATUS = 0
BEGIN

	exec('GRANT EXECUTE on ' + @name + ' to ' + @user)
   	PRINT ('GRANT EXECUTE on ' + @name + ' to ' + @user)	

   	FETCH NEXT FROM perm_cursor
   	INTO @name
END

CLOSE perm_cursor
DEALLOCATE perm_cursor
```

### GRANT EXECUTE permissions on Functions
*GRANTs permissions to EXECUTE functions called by stored procedures*
```sql
DECLARE @user varchar(50)
SELECT 	@user = QUOTENAME(@@servername + '\ASPNET')

DECLARE @name varchar(200)
DECLARE @type varchar(2)

DECLARE perm_cursor CURSOR FOR
SELECT name, type FROM SYSOBJECTS 
WHERE name LIKE 'fn__%' AND type IN ('FN','TF') 
ORDER BY NAME ASC

OPEN perm_cursor
FETCH NEXT FROM perm_cursor INTO @name, @type

WHILE @@FETCH_STATUS = 0
BEGIN

	IF (@type = 'FN')
	BEGIN
		exec('GRANT EXECUTE on ' + @name + ' to ' + @user)
	   	PRINT 'GRANT EXECUTE on ' + @name + ' to ' + @user	
	END
	ELSE IF (@type = 'TF')
	BEGIN
		exec('GRANT SELECT on ' + @name + ' to ' + @user)
		PRINT('GRANT SELECT on ' + @name + ' to ' + @user)
	END

   	FETCH NEXT FROM perm_cursor
   	INTO @name, @type
END

CLOSE perm_cursor
DEALLOCATE perm_cursor
```

### GRANT SELECT, UPDATE, INSERT, DELETE permissions on tables 
*GRANTs permissions to perform actions on tables from .NET app*
```sql
DECLARE @user varchar(50)
SELECT 	@user = QUOTENAME(@@servername + '\ASPNET')
DECLARE @name varchar(200)

DECLARE perm_cursor CURSOR FOR
SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE 
TABLE_NAME LIKE 'tbl%' ORDER BY TABLE_NAME ASC

OPEN perm_cursor
FETCH NEXT FROM perm_cursor INTO @name

WHILE @@FETCH_STATUS = 0
BEGIN

	exec('GRANT SELECT, UPDATE, DELETE, INSERT on ' + @name + ' to ' + @user)
   	PRINT 'GRANT SELECT, UPDATE, DELETE, INSERT on ' + @name + ' to ' + @user
	
   FETCH NEXT FROM perm_cursor
   INTO @name
END

CLOSE perm_cursor
DEALLOCATE perm_cursor
```

### GRANT SELECT permissions on list of tables 
*GRANTs only permissions to query from a comma-separated list of tables*
```sql
DECLARE @user varchar(50)
SELECT 	@user = QUOTENAME(@@servername + '\ASPNET')
DECLARE @name varchar(200)

USE <database-name>

DECLARE perm_cursor CURSOR FOR
SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES 
WHERE TABLE_NAME IN ('<t1>','<t2>','<t3>') 
ORDER BY TABLE_NAME ASC

OPEN perm_cursor
FETCH NEXT FROM perm_cursor INTO @name

WHILE @@FETCH_STATUS = 0
BEGIN

   exec('GRANT SELECT on ' + @name + ' to ' + @user)
   PRINT 'GRANT SELECT on ' + @name + ' to ' + @user	

   FETCH NEXT FROM perm_cursor
   INTO @name
END

CLOSE perm_cursor
DEALLOCATE perm_cursor
```

### DROP tables without a stored procedure reference
*Drops any tables that are not called from stored procedure*
```sql
DECLARE @user varchar(50)
SELECT 	@user = QUOTENAME(@@servername + '\ASPNET')
DECLARE @search varchar(200)
DECLARE @searchCount int
DECLARE @name varchar(200)

USE <<database-name>>

DECLARE perm_cursor CURSOR FOR
SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE 
TABLE_NAME LIKE 'tbl%' ORDER BY TABLE_NAME ASC

OPEN perm_cursor
FETCH NEXT FROM perm_cursor INTO @name

WHILE @@FETCH_STATUS = 0
BEGIN

	SELECT 	@search = '%' + @name + '%'
	SELECT 	@searchCount = COUNT(*)
	FROM 	information_schema.routines
	WHERE 	routine_definition LIKE @search

	IF (@searchCount = 0)
	BEGIN
		PRINT 'COUNT OF STORED PROCEDURES REFERENCING ' + QUOTENAME(@name) + ':' + str(@searchCount)
		DECLARE @dropSql varchar(200)
		SELECT @dropSql = 'DROP TABLE ' + @name
		EXEC(@dropSql)
		PRINT 'DROPPED TABLE '	+ QUOTENAME(@name) + ' (NO STORED PROCEDURES REFERENCE TABLE)'
	END
	ELSE
	BEGIN
		PRINT QUOTENAME(@name) + ' IGNORED'
	END
	
   FETCH NEXT FROM perm_cursor
   INTO @name

END

CLOSE perm_cursor
DEALLOCATE perm_cursor
```

### VIEW a list of all users across all Databases
```sql
DECLARE @Exec NVARCHAR(100)
DECLARE @Statement NVARCHAR(300)

DECLARE User_Cursor CURSOR FOR
Select Name from master.dbo.sysdatabases

OPEN User_Cursor
    FETCH NEXT FROM User_Cursor INTO @Exec

    WHILE @@FETCH_STATUS = 0
    BEGIN
	    SET @Statement = N'USE ' + @Exec + CHAR(13) + N'Exec SP_HelpUser' + CHAR(13)
	    PRINT @Statement	
	    EXEC sp_executesql @Statement	

        FETCH NEXT FROM User_Cursor INTO @Exec
    END

CLOSE User_Cursor
DEALLOCATE User_Cursor
```
