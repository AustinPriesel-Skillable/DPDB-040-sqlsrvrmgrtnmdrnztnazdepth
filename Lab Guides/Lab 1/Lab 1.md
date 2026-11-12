# Lab 1: Assess Migration Readiness for a SQL Server Estate

## Introduction

You've just joined Fabrikam Retail as the migration DBA. The company plans to move its SQL Server estate to Azure, but the server SQLVM01 has grown for years without documentation, and the DBA who built it has left. Before anyone picks an Azure destination, leadership wants a clear answer: what's on this server, which databases are ready to migrate, and what stands in the way? Your job is to find out, using the same tools and checks you'd use on a real customer engagement.

### Objectives

- Discover the SQL services, databases and server-level objects on a
  host, including ones that are easy to miss.

- Run an SSMS migration readiness assessment and interpret its results
  for each Azure SQL target.

- Validate the assessment with your own queries and uncover risks it
  doesn't report.

- Fix the issues that are safe to fix before choosing a destination, and
  prove each fix.

- Log the remaining risks as inputs for Lab 2.


## Exercise 1: Set up your Environment

### Task 1: Deploy the VM

1. Open +++https://portal.azure.com+++ and sign in with your lab account.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image1.png)

1. In the search bar at the top, type **Azure SQL** and select **Azure SQL** under Services.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image2.png)

1. In the left menu of the **Azure SQL** page, expand **SQL Server** and click **SQL Server on Azure VMs**. On that page, click **+Create**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image3.png)

1. In the **Image** drop-down, select **Free SQL Server License: SQL 2022 Developer on Windows Server 2022** from the list.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image4.png)

1. Select **Create SQL Virtual Machine** to proceed.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image5.png)

1. On the **Basics** tab, set:

    - **Resource group**: your group - **ResourceGroup1**
    - **Virtual machine name**: `SQLVM01`
    - **Region**: East US
    - **Availability options**: No infrastructure redundancy required
    - **Size**: click **See all sizes**, search and select **D4s_v5**
    - **Username**: `labadmin`
    - **Password**: `P@ssword12345`
    - **Public inbound ports**: Allow selected ports, with **RDP
    (3389)** selected

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image6.png)

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image7.png)

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image8.png)

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image9.png)

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image10.png)


1. Click the **SQL Server settings** tab at the top (skip the tabs in between). Set:

    - **SQL connectivity**: Public (Internet)
    - **Enable SQL Authentication**
    - **SQL Authentication**: Enable, with **Login name-** `labadmin`
    and the password – +++P@ssword12345+++

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image11.png)


1. Click **Review + create**. When validation passes, click **Create**. Go to the lecture while it deploys.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image12.png)

1. When you return, click **Go to resource**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image13.png)


### Task 2: Lock port 1433 to your own IP

The SQL port is open to the whole internet until you do this. Don't skip it.

1. On the VM's Overview page, copy the **Public IP address** and keep it handy. This is the Azure VM's address, the one SSMS connects to. Don't put it in the port rule. The rule's source is a different IP: your system’s VM address, the one SSMS connects from.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image14.png)

1. In the left menu, click Networking \   Network settings. click **+ Create port rule \   Inbound port rule**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image15.png)

1. Set the following settings:

    - **Source**: Select **My IP address**
    - **Source port ranges**: \*
    - **Destination**: Any
    - **Service**: MS SQL (this fills in port 1433 and protocol TCP)
    - **Action**: Allow
    - **Priority**: 310
    - **Name**: `Allow-SQL-MyIP`
    - Click **Add**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image16.png)

    ![A screenshot of a computer screen Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image17.png)


1. Make sure Inbound port rule for 1433 port is created.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image18.png)


### Task 3: Run the setup script through the portal

1. On the left menu, click **Operations** and then, **Run command**, then click **RunPowerShellScript**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image19.png)

1. Paste it into the **PowerShell Script** box and click **Run**. Leave the page open. When it finishes, the **Output** box ends with **Seed complete**.

    >[!Note] Make sure to replace the **<your sqladmin password>** with your password **at the first line of the script:**
  
    ```
    $SqlPwd = '<your sqladmin password>'
    $ErrorActionPreference = 'Stop'
    New-Item -ItemType Directory -Path C:\LabSetup, C:\exports -Force | Out-Null
    icacls C:\LabSetup /grant 'Everyone:(OI)(CI)R' | Out-Null

    # 1. Enable FILESTREAM on the service, make SQL Agent automatic
    $fs = Get-CimInstance -Namespace 'root\Microsoft\SqlServer\ComputerManagement16' -ClassName FilestreamSettings |
        Where-Object InstanceName -eq 'MSSQLSERVER'
    Invoke-CimMethod -InputObject $fs -MethodName EnableFilestream -Arguments @{AccessLevel=3; ShareName='MSSQLSERVER'} | Out-Null
    Stop-Service SQLSERVERAGENT -ErrorAction SilentlyContinue
    Stop-Service MSSQLSERVER -Force

    # 1b. Make sure the sqladmin login exists and SQL authentication is on.
    #     Single-user mode gives local administrators sysadmin access, so this works
    #     even if SQL authentication was not enabled when the VM was created.
    net start MSSQLSERVER /mSQLCMD | Out-Null
    Start-Sleep -Seconds 10
    $tsql = "IF SUSER_ID('sqladmin') IS NULL CREATE LOGIN sqladmin WITH PASSWORD = N'$SqlPwd' " +
            "ELSE ALTER LOGIN sqladmin WITH PASSWORD = N'$SqlPwd'; " +
            "ALTER LOGIN sqladmin ENABLE; ALTER SERVER ROLE sysadmin ADD MEMBER sqladmin; " +
            "EXEC xp_instance_regwrite N'HKEY_LOCAL_MACHINE', N'Software\Microsoft\MSSQLServer\MSSQLServer', N'LoginMode', REG_DWORD, 2;"
    sqlcmd -S localhost -E -C -b -Q $tsql
    if ($LASTEXITCODE -ne 0) { throw 'Could not set up the sqladmin login - read the sqlcmd error above.' }
    Stop-Service MSSQLSERVER -Force
    Start-Service MSSQLSERVER
    Set-Service SQLSERVERAGENT -StartupType Automatic
    Start-Service SQLSERVERAGENT
    Start-Sleep -Seconds 15

    # 2. Tiny CLR assembly, compiled with the C# compiler built into Windows
    @'
    using System.Data.SqlTypes;
    using Microsoft.SqlServer.Server;
    public class FileUtil {
        [SqlFunction]
        public static SqlBoolean FileExists(SqlString path) { return System.IO.File.Exists(path.Value); }
    }
    '@ | Set-Content C:\LabSetup\FabrikamClr.cs
    & "$env:WINDIR\Microsoft.NET\Framework64\v4.0.30319\csc.exe" /nologo /target:library `
    /r:System.Data.dll /out:C:\LabSetup\FabrikamClr.dll C:\LabSetup\FabrikamClr.cs

    # 3. Seed databases and issues
    @'
    EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
    EXEC sp_configure 'filestream access level', 2; RECONFIGURE;
    EXEC sp_configure 'Database Mail XPs', 1; RECONFIGURE;
    EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
    EXEC sp_configure 'clr enabled', 1; RECONFIGURE;
    DBCC TRACEON (1117, -1);
    GO
    -- SalesDB: clean control case
    CREATE DATABASE SalesDB;
    GO
    ALTER DATABASE SalesDB SET COMPATIBILITY_LEVEL = 130;
    GO
    USE SalesDB;
    CREATE TABLE dbo.Orders (OrderID int IDENTITY PRIMARY KEY, CustomerID int NOT NULL,
        OrderDate datetime2 NOT NULL DEFAULT SYSDATETIME(), Amount decimal(10,2) NOT NULL);
    INSERT dbo.Orders (CustomerID, Amount)
    SELECT TOP (5000) ABS(CHECKSUM(NEWID())) % 500, ABS(CHECKSUM(NEWID())) % 1000
    FROM sys.all_objects a CROSS JOIN sys.all_objects b;
    GO
    -- InventoryDB: cross-database query, Service Broker, orphaned user
    CREATE DATABASE InventoryDB;
    GO
    ALTER DATABASE InventoryDB SET ENABLE_BROKER WITH ROLLBACK IMMEDIATE;
    GO
    USE InventoryDB;
    CREATE TABLE dbo.Stock (ProductID int PRIMARY KEY, QtyOnHand int NOT NULL);
    GO
    CREATE PROCEDURE dbo.usp_StockVsOrders AS
    SELECT s.ProductID, COUNT(o.OrderID) AS OpenOrders
    FROM dbo.Stock s JOIN SalesDB.dbo.Orders o ON o.CustomerID = s.ProductID
    GROUP BY s.ProductID;
    GO
    CREATE LOGIN inv_app WITH PASSWORD = 'Orphan#Lab2026!';
    CREATE USER inv_app FOR LOGIN inv_app;
    DROP LOGIN inv_app;
    GO
    -- DocsArchive: FILESTREAM, compat 100
    DECLARE @p nvarchar(400) = CAST(SERVERPROPERTY('InstanceDefaultDataPath') AS nvarchar(400));
    DECLARE @s nvarchar(max) = N'CREATE DATABASE DocsArchive
    ON PRIMARY (NAME = DocsArchive_data, FILENAME = ''' + @p + N'DocsArchive.mdf''),
    FILEGROUP FS CONTAINS FILESTREAM (NAME = DocsArchive_fs, FILENAME = ''' + @p + N'DocsArchive_fs'')
    LOG ON (NAME = DocsArchive_log, FILENAME = ''' + @p + N'DocsArchive.ldf'');';
    EXEC (@s);
    GO
    ALTER DATABASE DocsArchive SET COMPATIBILITY_LEVEL = 100;
    GO
    -- FinanceDB: UNSAFE CLR assembly
    CREATE DATABASE FinanceDB;
    GO
    USE FinanceDB;
    DECLARE @bin varbinary(max) =
    (SELECT BulkColumn FROM OPENROWSET(BULK 'C:\LabSetup\FabrikamClr.dll', SINGLE_BLOB) AS f);
    DECLARE @hash varbinary(64) = HASHBYTES('SHA2_512', @bin);
    EXEC sys.sp_add_trusted_assembly @hash = @hash, @description = N'FabrikamClr';
    CREATE ASSEMBLY FabrikamClr FROM @bin WITH PERMISSION_SET = UNSAFE;
    GO
    CREATE FUNCTION dbo.fn_FileExists (@path nvarchar(4000)) RETURNS bit
    AS EXTERNAL NAME FabrikamClr.FileUtil.FileExists;
    GO
    -- ReportingDB: hard-coded host name and file share in code
    CREATE DATABASE ReportingDB;
    GO
    USE ReportingDB;
    CREATE TABLE dbo.DailyFeed (SalesDate date, Total decimal(12,2));
    GO
    CREATE PROCEDURE dbo.usp_LoadDailyFeed AS
    BULK INSERT dbo.DailyFeed FROM '\\SQLVM01\exports\daily_sales.csv'
    WITH (FIELDTERMINATOR = ',', FIRSTROW = 2);
    GO
    -- LegacyHR: forgotten database, left offline
    CREATE DATABASE LegacyHR;
    GO
    ALTER DATABASE LegacyHR SET OFFLINE;
    GO
    -- Server-level: Oracle linked server (definition only), Agent CmdExec job, Database Mail
    EXEC sp_addlinkedserver @server = N'ORA_GL', @srvproduct = N'Oracle',
        @provider = N'OraOLEDB.Oracle', @datasrc = N'oragl.fabrikam.local:1521/GL';
    EXEC msdb.dbo.sp_add_job @job_name = N'Finance - nightly GL export';
    EXEC msdb.dbo.sp_add_jobstep @job_name = N'Finance - nightly GL export',
        @step_name = N'Export to file', @subsystem = N'CmdExec',
        @command = N'bcp "SELECT name FROM FinanceDB.sys.objects" queryout C:\exports\gl_export.csv -c -T -S SQLVM01';
    EXEC msdb.dbo.sp_add_jobserver @job_name = N'Finance - nightly GL export';
    EXEC msdb.dbo.sysmail_add_account_sp @account_name = 'FabrikamAlerts',
        @email_address = 'alerts@fabrikam.example', @mailserver_name = 'smtp.fabrikam.example';
    EXEC msdb.dbo.sysmail_add_profile_sp @profile_name = 'FabrikamOps';
    EXEC msdb.dbo.sysmail_add_profileaccount_sp @profile_name = 'FabrikamOps',
        @account_name = 'FabrikamAlerts', @sequence_number = 1;
    GO
    '@ | Set-Content C:\LabSetup\seed.sql
    sqlcmd -S localhost -U sqladmin -P $SqlPwd -C -b -i C:\LabSetup\seed.sql
    if ($LASTEXITCODE -ne 0) { throw 'Seeding FAILED - read the sqlcmd error above.' }
    Write-Output 'Seed complete.'
    ```
  
    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image20.png)
  
    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image21.png)


### Task 4: Connect to Server

1. Open **SQL Server Management Studio** (SSMS 22) and select login with **Work or school account**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image22.png)

1. Sign in with the credentials.

    ![A screenshot of a login box Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image23.png)

1. In the **Connect to Server** dialog:

    - **Server type**: Database Engine
    - **Server name**: the VM's **public IP address** from Setup step 2
    - **Authentication**: SQL Server Authentication
    - **Login**: `labadmin`
    - **Password**- +++P@ssword12345+++
    - Tick **Trust server certificate**
    - Click **Connect**

    ![A screenshot of a computer screen Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image24.png)


1. In **Object Explorer**, expand the server node, then **Databases**. You should see six user databases, with **LegacyHR** marked **(Offline)**. In the rest of this guide, this connection is called the **sqladmin connection**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image25.png)


## Exercise 2: Discover SQL Server instances and databases

**Goal**: build an inventory that includes what a quick look misses, both forgotten databases and the server-level objects that don't travel with a database backup.

### Task 1: Enumerate SQL components on the host

1. In the Azure portal, open your **SQLVM01** VM and click **Operations \   Run command \   RunPowerShellScript**.

1. Paste the command below and click **Run**:

    ```
    Get-Service | Where-Object { $_.Name -match '^(MSSQL|SQLAgent|SQLSERVERAGENT|MsDtsServer|ReportServer|MSOLAP|SQLBrowser)' } |
    Select-Object Name, DisplayName, Status, StartType | Format-Table -AutoSize | Out-String -Width 200
    ```
  
    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image26.png)

1. In the **Output** box, write down how many **database engine** instances there are (names starting with MSSQL). Then note every other SQL service (Agent, Browser, SSIS) that a migration would also have to account for.

    This host runs one database engine instance, MSSQLSERVER, but seven SQL services in total. That’s the lesson: a SQL Server migration is rarely just the databases. The Agent is running, so there are scheduled jobs to move. SSIS is running, so there may be ETL packages that need their own migration path. Analysis Services, full-text search and ML Services are installed too. Each is either a dependency to plan for or something to confirm is unused and can be left behind. If we inventoried only databases, we’d miss all of this.
  
    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image27.png)


### Task 2: Inventory every database, including offline ones

1. Switch to **SSMS**. In **Object Explorer**, click the server node of your **sqladmin connection** (it shows the VM's IP address) and select the **New Query**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image28.png)

1. Paste the query below and click on **Execute**:

    ```
    SELECT  d.name AS database_name,
        d.state_desc,
        d.compatibility_level,
        d.recovery_model_desc,
        d.is_broker_enabled,
        CAST(SUM(mf.size) * 8.0 / 1024 AS decimal(10,1)) AS size_mb,
        MAX(CASE WHEN mf.type_desc = 'FILESTREAM' THEN 1 ELSE 0 END) AS has_filestream
    FROM    sys.databases d
    JOIN    sys.master_files mf ON mf.database_id = d.database_id
    WHERE   d.database_id > 4
    GROUP BY d.name, d.state_desc, d.compatibility_level, d.recovery_model_desc, d.is_broker_enabled
    ORDER BY d.state_desc DESC, size_mb DESC;
    ```
  
    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image29.png)

1. In the Results grid, check these three things:

    - **state_desc:** one database shows OFFLINE. Note its name.
    - **has_filestream:** one database shows 1. That's a FILESTREAM database.
    - **compatibility_level:** find the lowest value. That database will
    need an upgrade test before any migration.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image30.png)


1. In **Object Explorer**, expand **Databases** under the server node and find **LegacyHR**. Note the grey icon and the **(Offline)** label. Your query found it because **sys.master**\_files reads from master. Tools and scripts that open each database in turn skip it silently.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image31.png)


### Task 3: Inventory server-level dependencies

These objects live in master or msdb. They don't move when a database is backed up and restored somewhere else.

1. Open a new query editor.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image32.png)

1. **Run** the query:

    ```
    SELECT 'SQL logins'             AS object_type, COUNT(*) AS cnt FROM sys.server_principals WHERE type = 'S' AND name NOT LIKE '##%'
    UNION ALL SELECT 'Windows logins/groups', COUNT(*) FROM sys.server_principals WHERE type IN ('U','G') AND name NOT LIKE 'NT %'
    UNION ALL SELECT 'Linked servers',        COUNT(*) FROM sys.servers WHERE is_linked = 1
    UNION ALL SELECT 'Agent jobs',            COUNT(*) FROM msdb.dbo.sysjobs
    UNION ALL SELECT 'Database Mail profiles',COUNT(*) FROM msdb.dbo.sysmail_profile
    UNION ALL SELECT 'Credentials',           COUNT(*) FROM sys.credentials;
    ```
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image33.png)

1. Confirm the counts in the GUI. In **Object Explorer**:

    - Expand **Server Objects \   Linked Servers** and find **ORA_GL**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image34.png)

    - Expand **SQL Server Agent \   Jobs** and find **Finance - nightly GL
    export**. syspolicy_purge_history is a built-in job, so ignore it.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image35.png)

    - Expand **Management \   Database Mail**, right-click it, and select
    **Configure Database Mail**. Choose **Manage profile security** to see **FabrikamOps**, then click **Cancel**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image36.png)
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image37.png)
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image38.png)
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image39.png)
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image40.png)


## Exercise 3: Run an Azure SQL migration assessment

**Goal**: assess every database under a least-privilege login, then prove that the assessment's coverage matches the inventory from Exercise 1.

### Task 1: Run the assessment

1. In **Object Explorer**, right-click the **sqladmin** server node and select **Migrate SQL Server**. The assessment runs as sqladmin, because it needs sysadmin to read the databases.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image41.png)

1. On the **Migrate SQL Server to Azure** page, find the **Migration readiness assessment** card (**Step 1 of 4**) and click **Run readiness assessment**. It takes about a minute. Don't use the **Provision target** or **Migrate data** cards; they belong to later labs.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image42.png)
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image43.png)

1. Review the file.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image44.png)

1. Check the report and drill into one blocker:

    - Under **Source instance**, **User databases** should read **5/6**.
    LegacyHR is offline, so it isn't assessed yet; Task 2 fixes that.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image45.png)

    - On the **Azure SQL Database** card, click **View details**. Select
    **Compatibility findings** tab and select **SQL CLR assemblies** finding and look at the **impacted objects**; it names the **FabrikamClr** assembly.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image46.png)
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image47.png)
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image48.png)

    - Go back and click **View details** on the **Azure SQL Managed
    Instance** card. One database is **Ready with warnings**. Switch to **Compatibility findings** and read the warning. Then compare: the same database is **Not ready** for Azure SQL Database. The same feature can block one target and only warn on another.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image49.png)
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image50.png)
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image51.png)
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image52.png)


### Task 2: Check coverage against your inventory

1. **View the details** for the **SQL Server on Azure VM**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image53.png)

1. Compare the databases in the report with your Exercise 1 inventory. One is missing. Observe which one, and why.

    **Answer**: LegacyHR wasn’t assessed because it’s offline, and the tool can only assess databases it can open. so, it's skipped. The report still counts it but gives it no readiness result.
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image54.png)

1. Bring it online now. Navigate to **SSMS**.

1. In **Object Explorer**, under the **sqladmin** server node, expand **Databases**, right-click **LegacyHR**, and select **Tasks \   Bring Online**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image55.png)

1. Click **Close** when it shows **Success**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image56.png)

1. Go back to the **Migration** tab and click **Run readiness assessment** again.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image57.png)

1. Then confirm that LegacyHR now appears in the report. To compare runs later, use **View assessment history** on the same card.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image58.png)
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image59.png)


## Exercise 4: Find, fix, prove compatibility issues

**Goal**: work like a migration DBA. Scan for issues, fix the ones that can be fixed before any destination is chosen, prove the fixes with a fresh assessment, and log what remains for Lab 2.

Use the **sqladmin server node** for every task. Each task ends with a **Log it** line. Add that row to the **Remediation log** at the end of this exercise as you go.

### Task 1: Scan: Run the probe pack

1. In SSMS, open a new query editor.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image60.png)

1. Run the query. Each numbered probe returns its own result grid, so scroll through the **Results** tab to see them all.

    ```
    -- P1. Cross-database references in code
    EXEC sp_MSforeachdb N'USE [?]; IF DB_ID() > 4
    SELECT ''P1'' AS probe, DB_NAME() AS db, OBJECT_NAME(referencing_id) AS obj, referenced_database_name
    FROM sys.sql_expression_dependencies WHERE referenced_database_name IS NOT NULL;';
    -- P2. User CLR assemblies
    EXEC sp_MSforeachdb N'USE [?]; IF DB_ID() > 4
    SELECT ''P2'' AS probe, DB_NAME() AS db, name, permission_set_desc FROM sys.assemblies WHERE is_user_defined = 1;';
    -- P3. Hard-coded host names or file loads in code
    EXEC sp_MSforeachdb N'USE [?]; IF DB_ID() > 4
    SELECT ''P3'' AS probe, DB_NAME() AS db, OBJECT_NAME(object_id) AS obj,
        CASE WHEN definition LIKE ''%SQLVM01%'' THEN ''host name'' ELSE ''file load'' END AS hit
    FROM sys.sql_modules WHERE definition LIKE ''%SQLVM01%'' OR definition LIKE ''%BULK INSERT%'';';
    -- P4. Orphaned SQL users
    EXEC sp_MSforeachdb N'USE [?]; IF DB_ID() > 4
    SELECT ''P4'' AS probe, DB_NAME() AS db, dp.name FROM sys.database_principals dp
    LEFT JOIN sys.server_principals sp ON sp.sid = dp.sid
    WHERE dp.type = ''S'' AND sp.sid IS NULL AND dp.name NOT IN (''dbo'',''guest'',''INFORMATION_SCHEMA'',''sys'');';
    -- P5. Agent job steps that run outside the engine
    SELECT 'P5' AS probe, j.name, s.step_name, s.subsystem FROM msdb.dbo.sysjobs j
    JOIN msdb.dbo.sysjobsteps s ON s.job_id = j.job_id WHERE s.subsystem IN ('CmdExec','PowerShell','SSIS');
    -- P6. Risky instance settings and trace flags
    SELECT 'P6' AS probe, name, value_in_use FROM sys.configurations WHERE name IN ('xp_cmdshell','clr enabled');
    DBCC TRACESTATUS(-1);	
    ```
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image61.png)

1. Scroll through the **Results** tab. Most grids are empty, because each probe returns one grid per database and only some databases have hits. Look for the rows labelled P1 to P6. Then **leave this tab open**.


### Task 2: Fix the orphaned user

1. ***Investigate --*** In **Object Explorer**, click the + next to **InventoryDB** to expand it. Under InventoryDB, click + next to its own **Security** folder, then + next to **Users**. This is the database's **Security** folder, not the server-level Security folder lower down.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image62.png)

1. Right-click **inv_app** and select **Properties**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image63.png)

1. On the **General** page, **User type** shows **SQL user without login**, and there's no Login name field. That confirms the user is orphaned: it was created for a login that no longer exists. Click **Cancel**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image64.png)

1. ***Fix --*** Run the query below, which creates the missing login and remaps the user:

    ```
    USE master;
    CREATE LOGIN inv_app WITH PASSWORD = 'Orphan#Lab2026!';
    USE InventoryDB;
    ALTER USER inv_app WITH LOGIN = inv_app;
    ```
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image65.png)
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image66.png)

1. ***Verify --*** Go back to the probe tab, select the **P4** lines, and press **Execute**. It should return no rows.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image67.png)

1. ***Fixed --*** In **Object Explorer**, click **inv_app**, select **Properties**, and check that **User** type now shows **SQL user with login** and **Login name** shows **inv_app**. Click **Cancel**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image68.png)

1. **Log it:** R01, orphaned user, status Fixed. In a real migration, you'd recreate the login with its original SID from the source server, so users in every database match automatically.


### Task 3: Retire the unused CLR assembly

1. ***Investigate --*** Expand **Databases \   FinanceDB \ Programmability \   Functions \   Scalar-valued Functions**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image69.png)

1. Right-click **dbo.fn_FileExists** and select **View Dependencies**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image70.png)

1. With **Objects that depend on \[fn_FileExists\]** selected, the list is empty. Click **OK**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image71.png)

1. Confirm nothing outside the database calls it either. *Run the query*:

    ```
    SELECT 'job step' AS found_in, step_name FROM msdb.dbo.sysjobsteps WHERE command LIKE '%fn_FileExists%'
    UNION ALL
    SELECT 'module', OBJECT_NAME(object_id) FROM FinanceDB.sys.sql_modules WHERE definition LIKE '%fn_FileExists%';
    ```
  
    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image72.png)

1. ***Fix --*** No rows means the assembly is dead code. *Run the query* to remove it and turn off CLR:

    ```
    USE FinanceDB;
    DROP FUNCTION dbo.fn_FileExists;
    DROP ASSEMBLY FabrikamClr;
    DECLARE @hash varbinary(64) = (SELECT hash FROM sys.trusted_assemblies WHERE description = N'FabrikamClr');
    EXEC sys.sp_drop_trusted_assembly @hash = @hash;
    EXEC sp_configure 'clr enabled', 0; RECONFIGURE;
    ```
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image73.png)

1. ***Verify --*** Expand **FinanceDB \   Programmability \ Assemblies** and Refresh. **FabrikamClr** is gone. **Microsoft.SqlServer.Types** stays; it's a built-in system assembly in every database, not user code.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image74.png)

1. **Rerun** probe **P2**; it returns no rows.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image75.png)

1. **Log it:** status *Fixed (removed)*.


### Task 4: Remove the hard-coded file share

1. ***Investigate --*** Expand **Databases \   ReportingDB \ Programmability \   Stored Procedures**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image76.png)

1. Right-click **dbo.usp_LoadDailyFeed** and select **Modify**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image77.png)

1. The script shows the server name `SQLVM01` written into the file path. That breaks the day the server’s name changes.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image78.png)

1. ***Fix --*** In that script window, press **Ctrl+A**, and paste the version below, and press **Execute**. The path becomes a parameter, so the caller supplies it.

    ```
    ALTER PROCEDURE dbo.usp_LoadDailyFeed
    @FilePath nvarchar(400)
    AS
    DECLARE @sql nvarchar(max) = N'BULK INSERT dbo.DailyFeed FROM '''
        + REPLACE(@FilePath, '''', '''''')
        + N''' WITH (FIELDTERMINATOR = '','', FIRSTROW = 2);';
    EXEC sys.sp_executesql @sql;
    ```
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image79.png)

1. ***Verify --*** Rerun probe **P3**. ReportingDB now shows *file load* instead of *host name*. The host dependency is gone, but the procedure still loads from a file. Where that file can live depends on the destination.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image80.png)


### Task 5: Inspect the Agent job, but don’t fix it

1. Expand **SQL Server Agent \   Jobs**. Right-click **Finance - nightly GL export**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image81.png)

1. Select **Properties**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image82.png)

1. In the left pane, click **Steps**. Select the step and click **Edit**. Note that **Type** is **Operating system (CmdExec)**, and that the command writes to C:\exports.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%201/media/image83.png)

1. Click **Cancel** twice. Don't change anything: the export feeds another team's process, so it needs that team's sign-off.


## Conclusion

In this lab, you learned that migration readiness starts with discovery: the host's services, offline databases and server-level objects that don't travel with a backup. You ran an SSMS readiness assessment, read its results across Azure targets, and saw its limits. You then confirmed findings with your own queries, fixed the issues that are safe to fix before choosing a destination, proved each fix, and logged the rest as inputs.
