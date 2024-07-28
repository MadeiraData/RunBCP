# RunBCP

**APPLIES TO:** ![Yes](media/yes-icon.png)SQL Server ![No](media/no-icon.png)Azure SQL Database ![No](media/no-icon.png)Azure SQL Managed Instance ![No](media/no-icon.png)Azure Synapse Analytics (SQL DW) ![No](media/no-icon.png)Parallel Data Warehouse 

SQL-CLR project for running the [BCP utility](https://learn.microsoft.com/en-us/sql/tools/bcp-utility) from within a SQL Server instance without needing [xp_cmdshell](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/xp-cmdshell-transact-sql).

Credit to Marcin from [this Stack Overflow thread](https://stackoverflow.com/questions/36397427/what-is-the-best-to-remove-xp-cmdshell-calls-from-t-sql-code) for the original code.

## Example

```sql
/* Example usage: */

USE [myTestDB]
GO

DECLARE @arguments nvarchar(max) = 'myTestDB.dbo.OrderDetails out "C:\temp\OrderDetails.bcp" -T -c -C ACP'
DECLARE @output_msg nvarchar(max)
DECLARE @error_msg nvarchar(max)
DECLARE @return_val int

EXECUTE [dbo].[RunBCP] 
   @arguments
  ,@output_msg OUTPUT
  ,@error_msg OUTPUT
  ,@return_val OUTPUT

SELECT
 @output_msg
,@error_msg 
,@return_val
```


## Installation

**More detailed instructions TBA**

Before being able to import this SQL-CLR into your database, you need to import its digital signing key.

You can use a script such as below to do this (remember to enable SQLCMD mode):

```sql
/* Remove the "setvar" lines below unless you want to run this script manually */
:setvar PathToSignedDLL C:\Madeira\RunBCP.dll
:setvar CLRKeyName RunBCPAssemblyKey
:setvar CLRLoginName RunBCPAssemblyLogin
:setvar DatabaseName myTestDB
/*
 Pre-Deployment Script Template	For Importing a Signed CLR Assembly (SSDT Project)
--------------------------------------------------------------------------------------
In order to use this script, you must configure the following SQLCMD Variables in your project:

$(PathToSignedDLL)
$(CLRKeyName)
$(CLRLoginName)

To configure your SQLCMD Variables: Right-click on your DB project, select "Properties", and go to "SQLCMD Variables".
Add your new SQLCMD Variables in the grid.

The $(PathToSignedDLL) variable is particularly tricky, because it requires you to know, in advance,
the path to your signed assembly DLL. That file may not be available until after you first build your project.
Consistent file paths must be maintained, especially when using CI/CD pipelines.

To sign your assembly:
Right-click on your DB project, select "Properties", and go to "SQLCLR".
Click on the "Signing..." button.
Enable the "Sign the assembly" checkbox.
Using the dropdown box, choose an existing strong name key file (*.snk) or create a new one.
Set a password for the key file.
This password is required for this phase only (initial signing config).
If you plan to use the same strong name key file for other assemblies, make sure to save your password.
--------------------------------------------------------------------------------------
*/
-- Make sure clr is enabled
IF EXISTS (select * from sys.configurations where name IN ('clr enabled') and value_in_use = 0)
BEGIN
	DECLARE @InitAdvanced INT;
	SELECT @InitAdvanced = CONVERT(int, value) FROM sys.configurations WHERE name = 'show advanced options';

	IF @InitAdvanced = 0
	BEGIN
		EXEC sp_configure 'show advanced options', 1;
		RECONFIGURE;
	END

	EXEC sp_configure 'clr enabled', 1;
	RECONFIGURE;

	IF @InitAdvanced = 0
	BEGIN
		EXEC sp_configure 'show advanced options', 0;
		RECONFIGURE;
	END
END
GO
-- Database context must be switched to [master] when creating the key and login
use [master];
GO
-- Create asymmetric key from DLL
IF NOT EXISTS (SELECT * FROM master.sys.asymmetric_keys WHERE name = '$(CLRKeyName)')
	CREATE ASYMMETRIC KEY [$(CLRKeyName)]
	FROM EXECUTABLE FILE = '$(PathToSignedDLL)'
GO
-- Create server login from asymmetric key
IF NOT EXISTS (SELECT name FROM master.sys.syslogins WHERE name = '$(CLRLoginName)')
	CREATE LOGIN [$(CLRLoginName)] FROM ASYMMETRIC KEY [$(CLRKeyName)];
GO
-- Grant UNSAFE/EXTERNAL_ACCESS/SAFE ASSEMBLY permissions to login which was created from DLL signing key
GRANT UNSAFE ASSEMBLY TO [$(CLRLoginName)];
GO
-- Return execution context to intended target database
USE [$(DatabaseName)];
GO
```

## License

All materials in this repository are released under the [MIT license](https://github.com/MadeiraData/RunBCP/blob/master/LICENSE.txt).
