# Sql CI


A very simple sql script migration utility for continuous integration and automated deployments

# Features
- Automate database deployments to MS Sql Server via command line utility
- Easily integrates with automated deployment solutions by accepting configuration using either a .config file (for use with octopus deploy) or from command line arguments (for nant, batch scripts, etc). 
- Roll back changes based on either a release number or a script number (can associate a batch of scripts with a single release number i.e. 1.3.4 could be associated with 15 sql scripts)
- Use exit code so that various automation tools can know if it was successful or not (don't just depend on console output)
- Optionally run drop/create database scripts for when running on developer workstations or in between automated test runs
- Support scripts that need to change databases. For example a script that runs in "MyDatabase" but also needs to create a job via msdb

**Non-Goal** - A specific non-goal is to support scripts that use the GO keyword to separate statements that are dependent on the prior statement being committed to the database. It's far easier to just make multiple scripts

# Configuration
All configuration is done through the SqlCi.Console.exe.config file. This format was chosen over command line arguments simply because Octopus Deploy has built in support for modifying the .config files with variables that are specific to each environment/role and it also does not exclude the tool being used by other automation platforms. You'll need to specify a connection string to the database to run the scripts against as well as some appSetting entries as shown below.

## Connection String
The connection string should be specified under the connectionStrings element and have a name of "Database"
```csharp
<connectionStrings>
	<add name="Database" connectionString="server=myserver; database=mydatabase; user=myuser; password=mypassword;"/>
</connectionStrings>
```

## App Settings
**ScriptTable** - The name of the table that should be used to store the scripts that have been ran. Defaults to "ScriptTable"
**ReleaseVersion** - The version of the release to associate with the current script deployment. Defaults to "1.0.0"
**ResetScriptsFolder** - The name of the folder that holds the scripts to reset (drop/create) the database. Defaults to "ResetScripts". This should be a relative path from the directory where SqlCi.Console.exe is ran from.
**ResetDatabase** - A boolean value that determines if the database should be reset (dropped/created) by running the scripts in the folder specified by the ResetScriptsFolder value.
**ScriptsFolder** - The name of the folder that holds the schema and data population scripts that should be ran against the database specified in the "Database" connection string setting. This should be a relative path from the directory where SqlCi.Console.exe is ran from. 

# Usage
## If using SqlCi.Console.exe.config
Once the SqlCi.Console.exe.config file has been setup with the appropriate connection string and apps settings simple run SqlCi.Console.exe. Output will look similar to the following:

	Verifying configuration ...
	Configuration verification complete. Starting deployment ...
	Loading sql script files from .\Scripts ...
	Loaded 4 sql script files from .\Scripts ...
	Checking for existance of script tracking table in the database ...
	Opening connection to sql server using connection string: server=sqlserver2012; database=esbsecurity; user=wshaddix; password=Airplane500; ...
	Script tracking table already exists ...
	Checking to see which scripts have already been applied ...
	Found 2 script(s) that have already been applied ...
	Calculating which new script(s) need to be applied ...
	2 new script(s) need to be applied ...
	Applying script 0003_Change_Databases.sql ...
	Applying script 0004_Insert_Data.sql ...
	Deployment complete.
	Closing connection to sql server ...


## If using command line arguments

## Rolling back changes
Rolling back changes must be done using the command line with the -rollback argument as in the example below.

	SqlCi.Console.exe -rollback:1.3.2

Running this command will rollback all of the scripts from version 1.3.2 and all scripts applied after version 1.3.2, so it is inclusive of the scripts ran for the 1.3.2 release.

# Script Conventions
## Naming
Every script must be named with a sequence number followed by an underscore. An example would be

	0001_Create_Customer_Table.sql
	0002_Create_Order_Table.sql
	0003_Create_OrderItem_Table.sql
	0004_Create_States_Table.sql
	0005_Populate_States_Table.sql


SqlCi will take the file name and strip the first N characters before the first underscore and use that as the sequence to sort by when running the scripts. Technically you can use any naming convention where the characters before the first underscore sorts sequentially .

## Rollbacks
In order to rollback a release version you must ensure that for every script that needs compensation you have both the update script (described above) as well as a compensating script with the file name ending in _Rollback.sql as shown in the example below

	0001_Create_Customer_Table.sql
	0001_Create_Customer_Table_Rollback.sql
	0002_Create_Order_Table.sql
	0002_Create_Order_Table_Rollback.sql
	0003_Create_OrderItem_Table.sql
	0003_Create_OrderItem_Table_Rollback.sql
	0004_Create_States_Table.sql
	0004_Create_States_Table_Rollback.sql
	0005_Populate_States_Table.sql
	0005_Populate_States_Table_Rollback.sql
