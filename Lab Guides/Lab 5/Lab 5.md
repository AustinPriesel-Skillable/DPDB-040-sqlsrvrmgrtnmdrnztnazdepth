# Lab 5: Deploy and Validate Azure SQL Migration Targets

**Duration:** about 40 minutes

### Lab overview

In Lab 2 you finished Fabrikam Retail’s destination recommendation for the databases on SQLVM01. Leadership approved it, and the migration team has grouped the work into three waves:
- **Wave 1: ReportingDB to Azure SQL Database.** No blockers, low
  business risk. It is the pilot that proves the process.

- **Wave 2: SalesDB and InventoryDB to Azure SQL Managed Instance.**
  Business critical. They must move together because InventoryDB queries SalesDB across databases.

- **Wave 3: FinanceDB to Azure SQL Database, and DocsArchive to SQL
  Server on Azure VM.** FinanceDB waits for Finance to sign off the rebuilt GL export. DocsArchive uses FILESTREAM, which only a VM supports, and needs an upgrade test from compatibility level 100.

- **Retire: LegacyHR.** It gets no target.


Leadership also wants every target ready for the AI features planned for later in the year, so each destination must support the native vector data type.

In this lab you build the empty targets for all three waves. Migrating the data comes later.

### Objectives

After completing this lab, you will be able to:
- Deploy SQL Server on Azure VM with storage best practices, and prove
  it supports a feature the PaaS targets don’t.

- Deploy Azure SQL Database at the size and tier the selection rules
  call for, and observe its database-scoped boundary.

- Deploy Azure SQL Managed Instance at the size Azure Arc recommended,
  and prove it supports the instance-scoped features that decided its selection.

- Tag every target with its migration wave and workload and review the
  targets wave by wave.

- Check native vector support on each destination and set the Managed
  Instance update policy that enables it.


### Getting Started: Build the Network and Start the Managed Instance

**Duration:** 8 minutes

A new managed instance takes about 30 minutes to deploy, so you start it first and build the other two targets while it deploys.

**Work in this order:** Getting Started, Exercise 1 (Task 1), Exercise 2, Exercise 1 (Tasks 2 and 3), then Exercise 3.

### Task 1: Create the virtual network

The managed instance and the SQL VM share one virtual network so the VM can reach the instance privately. SQL Managed Instance needs a subnet of its own, so the network has two subnets.

1. Open +++https://portal.azure.com/+++ and sign in with your lab account.

    ![A screenshot of a login screen Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image1.png)

1. In the top search bar, type **Virtual networks** and select **Virtual networks** under **Services**. Click **+ Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image2.png)

1. On the **Basics** tab, set **Resource group** to **ResourceGroup1**, **Virtual network name** to **vnet-fabrikam** and **Region** to **Central US**.

    Click **Next** until you reach the **Address Space** tab.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image3.png)

1. Leave the address space at **10.0.0.0/16**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image4.png)

1. Rename the existing subnet in the default subnet row, click the default link or the **pencil (edit)** icon.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image5.png)

1. Set Name to `snet-sqlmi`, leave Starting address at 10.0.0.0 and Size at /24.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image6.png)

1. Scroll down to **Private subnet** and **clear** the **Enable private subnet (no default outbound access)** check box; the portal selects it by default. Leave **NAT gateway**, **Network security group**, **Route table** and **Subnet delegation** at **None**. Click **Save**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image7.png)

1. Click **+ Add a subnet**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image8.png)

1. Set **Name** to **snet-sqlvm**, **Starting address** to **10.0.1.0** and **Size** to **/24**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image9.png)

1. Scroll down and **clear** **Enable private subnet (no default outbound access)** here too, leaving everything else at **None**. Click **Add**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image10.png)

1. Click **Review + create**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image11.png)

1. Click on **Create** to proceed with the deployment.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image12.png)

    **Expected result:** **Your deployment is complete** within a minute.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image13.png)


### Task 2: Start the Managed Instance deployment (Wave 2)

SalesDB and InventoryDB go to Managed Instance because of their instance-scoped features: a cross-database query and Service Broker. You use the size Azure Arc recommended in Lab 2: Next-gen General Purpose, Standard-series (Gen5), 4 vCores and 32 GB.

1. In the top search bar on **Azure Portal**, type **SQL managed instances** and select **SQL managed instances** under **Services**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image14.png)

1. Click **+ Create** and then, select **SQL managed instance.**

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image15.png)

1. On the **Basics** tab, set **Resource group** to **ResourceGroup1**, **Managed Instance name** to **sqlmi-fabrikam-jd427** and **Region** to **Central US**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image16.png)

1. On the Basics tab under **Managed Instance details**, click **Configure Managed Instance** in **Compute+Storage**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image17.png)

1. Set the following settings:

    - **Service tier:** General Purpose, with **Next-gen General Purpose**
    selected (it may appear as a check box or option under the tier)

    - **Hardware generation:** Standard-series (Gen5)
    - **vCores:** 4
    - **Storage in GB:** 32
    - **Backup storage redundancy:** Locally-redundant backup storage

    Click **Apply**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image18.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image19.png)

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image20.png)


1. Under **Authentication**, set **Authentication method** to **Use SQL authentication**, Managed Instance **admin** **login** to **sqladminuser**, and **Password** and **Confirm password** to **Fabrikam!Lab2026**.

    Click **Next: Networking**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image21.png)

1. Set **Virtual network / subnet** to **vnet-fabrikam/snet-sqlmi** and **Public endpoint** to **Disable**. If the portal says the subnet will be configured for SQL Managed Instance, leave that as it is.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image22.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image23.png)

1. Click **Next: Security**, then **Next: Additional settings**. Leave **Collation**, **Time zone** and **Maintenance window** at their defaults. Under **Update policy**, select **SQL Server 2025**.

    **AI-ready consideration:** the update policy decides which engine features the instance gets. The **SQL Server 2022** policy has no vector data type. The **SQL Server 2025** policy adds the vector data type and functions, and backups can still be restored to SQL Server 2025. **Always-up-to-date** gets every new feature first, but you can never switch back, and backups can’t be restored to any SQL Server version.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image24.png)

1. Click **Next** until you reach the **Tags** tab.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image25.png)

1. Add two tags:

    1. **Name:** MigrationWave, **Value:** 2

    1. **Name:** Workload, **Value:** SalesDB and InventoryDB

    Click **Review + create**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image26.png)

1. Then, click on **Create**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image27.png)

    **Expected result:** **Deployment is in progress**. Don’t wait. Go straight to Exercise 1.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image28.png)

    **If the deployment fails with a quota or capacity error,** tell your instructor. Lab subscriptions sometimes have no Managed Instance quota in a region.


## Exercise 1: Deploy SQL Server on Azure Virtual Machine

**Duration:** 12 minutes

DocsArchive goes to a VM because it uses FILESTREAM, which neither PaaS target supports. Lab 1 built SQLVM01 as a default source server. This VM is a **target**, so you build it the way Module 3 recommends for production: a SQL Server 2025 image for AI readiness, private SQL connectivity, separate data and log disks, and tempdb on the local SSD. The VM size, D2ads_v5, is the size the SKU recommendation produced in Lab 4.

### Task 1: Create the SQL virtual machine (Wave 3)

1. In the top search bar, type **Azure SQL** and select **Azure SQL** under **Services**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image29.png)

1. In the left menu, expand **SQL Server** and click **SQL Server on Azure VMs**. Click **+ Create**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image30.png)

1. In the **Image** drop-down, select **Free SQL Server License: SQL Server 2025 Enterprise Developer on Windows Server 2025**, then click **Create** SQL Virtual Machine.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image31.png)

1. On the **Basics** tab, set:

    - **Resource group:@lab.CloudResourceGroup(ResourceGroup1).Name
    - **Virtual machine name:** sqlvm-docs
    - **Region:@lab.CloudResourceGroup(ResourceGroup1).Location
    - **Availability options:** No infrastructure redundancy required
    - **Size:** click **See all sizes**, search for `D2ads_v5`, select
    **Standard_D2ads_v5** and click **Select**

    - **Username:** sqladminuser
    - **Password** and **Confirm password:** Fabrikam!Lab2026
    - **Public inbound ports:** Allow selected ports, with **RDP (3389)**
    selected

    **Why D2ads_v5?** The **d** in the name means the size has a local temporary SSD, which is where tempdb goes. A size without the **d**, such as D2s_v5, has no local disk to put tempdb on.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image32.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image33.png)

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image34.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image35.png)


1. Click **Next: Disks**. Leave the defaults. Click **Next: Networking**, and set **Virtual network** to **vnet-fabrikam** and **Subnet** to **snet-sqlvm**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image36.png)

1. Click the **SQL Server settings** tab and set:

    1. **SQL connectivity:** Private (within Virtual Network)

    1. **SQL Authentication:** Enable, with **Login name** sqladminuser and **Password** Fabrikam!Lab2026

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image37.png)

1. On the same tab, under **Storage configuration**, click **Change configuration**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image38.png)

1. The Configure storage pane opens with a recommendation for **D2ads_v5**. Check the settings, without changing any values:

    - **Data storage**: Data drive location is **F:\data**, Disk type is
    **Premium SSD**, and there is 1 disk of 1024 GiB. Leave Set system databases (except tempdb) location to newly created data storage cleared.

    - **Log storage**: click it to expand. It shows 1 disk of 1024 GiB,
    Premium SSD, on the G: drive.

    - **tempdb storage**: it shows Using local SSD drive for tempdb.

    Click **Ok**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image39.png)


1. Navigate to **Tags** tab and add two tags:

    1. **Name:** MigrationWave, **Value:** 3

    1. **Name:** Workload, **Value:** DocsArchive

    Click **Review + create**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image40.png)

1. Then **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image41.png)

    **Expected result:** deployment starts and takes 8 to 12 minutes. Don’t wait. Go straight to Exercise 2, then return here for Task 2.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image42.png)


### Task 2: Check the storage layout

1. When the deployment completes, click **Go to resource**. In the virtual machine’s left menu, click **Operations \> Run command**, then click **RunPowerShellScript**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image43.png)

1. Paste this script into the **Run Command Script** box and click **Run**. It saves a small helper, **Invoke-LabSql**, that runs T-SQL with the .NET client built into Windows, then checks the SQL Server version and storage layout: **PowerShell (Run command on sqlvm-docs)**

    ```
    New-Item -ItemType Directory -Path C:\Lab -Force | Out-Null
    function Invoke-LabSql {
    param([string]$Server, [string]$Query, [string]$Database =
    'master')
    $cs = "Server=$Server;Database=$Database;User
    ID=sqladminuser;Password=Fabrikam!Lab2026;Encrypt=True;TrustServerCertificate=True;Connect
    Timeout=30"
    $conn = New-Object System.Data.SqlClient.SqlConnection $cs
    $adapter = New-Object System.Data.SqlClient.SqlDataAdapter $Query,
    $conn
    $results = New-Object System.Data.DataSet
    [void]$adapter.Fill($results)
    foreach ($t in $results.Tables) { $t | Format-List | Out-String -Width
    200 }
    }
    # Save the helper so later scripts can load it
    "function Invoke-LabSql {" + ${function:Invoke-LabSql} + "}" |
    Set-Content C:\Lab\Invoke-LabSql.ps1
    Invoke-LabSql -Server localhost -Query "SELECT
    SERVERPROPERTY('ProductMajorVersion') AS major_version,
    SERVERPROPERTY('Edition') AS edition,
    SERVERPROPERTY('InstanceDefaultDataPath') AS data_path,
    SERVERPROPERTY('InstanceDefaultLogPath') AS log_path, (SELECT TOP (1)
    physical_name FROM sys.master_files WHERE database_id = 2) AS
    tempdb_file;"
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image44.png)

    **Expected result:** after up to a minute, the Output pane shows major_version 17 (SQL Server 2025), an Enterprise Developer edition, data_path on the F: drive, log_path on the G: drive and tempdb_file on the D: drive (the local SSD). Data, log and tempdb each have their own drive.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image45.png)


### Task 3: Prove the selection criteria and AI readiness

In this task, you prove that the VM meets the two requirements for DocsArchive. First, it supports FILESTREAM, which Azure SQL Database and Azure SQL Managed Instance don’t; this is why DocsArchive goes to a VM. Second, it supports the native vector data type that Fabrikam’s planned AI features need.

1. In the **Run Command Script** box, replace the script with this one and click **Run**. It enables FILESTREAM on the SQL Server service and restarts SQL Server: **PowerShell (Run command on sqlvm-docs)**

    ```
    $ns = Get-CimInstance -Namespace root\Microsoft\SqlServer -ClassName
    __NAMESPACE |
    Where-Object Name -like 'ComputerManagement*' | Sort-Object Name
    -Descending | Select-Object -First 1
    $fs = Get-CimInstance -Namespace
    "root\Microsoft\SqlServer\($ns.Name)" -ClassName FilestreamSettings
    -Filter "InstanceName='MSSQLSERVER'"
    Invoke-CimMethod -InputObject $fs -MethodName EnableFilestream
    -Arguments @{ AccessLevel = 1; ShareName = 'MSSQLSERVER' } | Out-Null
    Restart-Service MSSQLSERVER -Force
    Start-Service SQLSERVERAGENT -ErrorAction SilentlyContinue
    Get-Service MSSQLSERVER | Select-Object Name, Status | Out-String
    ```

    **Expected result**: MSSQLSERVER with status Running.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image46.png)

1. Replace the script with this one and click **Run**. It sets the FILESTREAM access level, creates a test database with a FILESTREAM filegroup on the data drive, and declares a vector:

    . C:\Lab\Invoke-LabSql.ps1

    ```
    Invoke-LabSql -Server localhost -Query "EXEC sp_configure 'filestream
    access level', 1; RECONFIGURE;"
    Invoke-LabSql -Server localhost -Query "
    DECLARE @p nvarchar(400) =
    CAST(SERVERPROPERTY('InstanceDefaultDataPath') AS nvarchar(400));
    DECLARE @s nvarchar(max) = N'CREATE DATABASE DocsArchive_Target
    ON PRIMARY (NAME = DocsArchive_data, FILENAME = ''' + @p +
    N'DocsArchive_Target.mdf''),
    FILEGROUP FS CONTAINS FILESTREAM (NAME = DocsArchive_fs, FILENAME =
    ''' + @p + N'DocsArchive_Target_fs'')';
    IF DB_ID('DocsArchive_Target') IS NULL EXEC (@s);
    SELECT name, type_desc, physical_name FROM sys.master_files WHERE
    database_id = DB_ID('DocsArchive_Target');
    "
    Invoke-LabSql -Server localhost -Query "DECLARE @v VECTOR(3) = '[0.1,
    2, 30]'; SELECT CAST(@v AS nvarchar(100)) AS vector_value;"
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image47.png)

    **Expected result**: three files for DocsArchive_Target, one of them with type_desc FILESTREAM, followed by a vector_value of the three numbers (for example \[1.0000000e-001,2.0000000e+000,3.0000000e+001\]). FILESTREAM works because this is a full SQL Server instance. The vector works because the image is SQL Server 2025; a SQL Server 2022 image has no vector data type.

    >[!Note] If you see an error about FILESTREAM being disabled, step 1 didn’t finish. Run step 1 again, wait for Running, then run this step again.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image48.png)


## Exercise 2: Deploy Azure SQL Database

**Duration:** 10 minutes

ReportingDB and FinanceDB go to Azure SQL Database because they have no instance-scoped features, so they get the lowest management effort. In Lab 2’s decision flow, a database over 4 TB would go to **Hyperscale**; both databases are a few MB, so they use **General Purpose serverless**, which pauses when idle.

### Task 1: Create the logical server and ReportingDB (Wave 1)

1. In the portal search bar, type **SQL database** in the portal search bar and select **Azure** **SQL database** under **Services**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image49.png)

1. Click **+ Create** and select **SQL database**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image50.png)

1. On the **Basics** tab, set **Resource group** to **ResourceGroup1** and **Database name** to **ReportingDB**. Under **Server**, click **Create new**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image51.png)

1. In the **Create SQL Database Server** pane, set **Server name** to **sql-fabrikam-jd427**, **Location** to **Central US**, **Authentication method** to **Use SQL authentication**, **Server admin login** to **sqladminuser**, and **Password** and **Confirm password** to **Fabrikam!Lab2026**. Click **OK**.

    ![A screenshot of a computer screen Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image52.png)

1. Set **Want to use SQL elastic pool?** to **No** and **Workload environment** to **Development**.

    ![A screenshot of a computer screen Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image53.png)

1. Under **Compute + storage**, click **Configure database**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image54.png)

1. Set **Service tier** to **General Purpose (Scalable compute and storage options)**, **Compute tier** to **Serverless**, **Max vCores** to **2** and **Auto-pause delay** to **1 hour**. Click **Apply**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image55.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image56.png)

    **Hyperscale check:** open the **Service tier** drop-down before you click **Apply** and note **Hyperscale (On-demand scalable storage)**. That is where a database over 4 TB, or one that needs fast scale-out, would go. Leave **General Purpose** selected.

1. Set **Backup storage redundancy** to **Locally-redundant backup storage**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image57.png)

1. Click **Next: Networking**. Set **Connectivity method** to **Public endpoint**, **Allow Azure services and resources to access this server** to **No**, and **Add current client IP address** to **Yes**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image58.png)

1. Click **Next** until you reach the **Tags** tab. Add **MigrationWave** with value **1**, and **Workload** with value **ReportingDB**.

    Click **Review + create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image59.png)

1. Then, click on **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image60.png)

1. **Expected result:** **Your deployment is complete** after 3 to 5 minutes. Click **Go to resource**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image61.png)


### Task 2: Add FinanceDB to the same server (Wave 3)

1. On the ReportingDB **Overview** page, click the **Server name** link (sql-fabrikam-jd427).

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image62.png)

1. On the server’s toolbar, click **+ Create database**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image63.png)

1. Set **Database name** to **FinanceDB**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image64.png)

1. Under **Compute + storage**, click **Configure database.**

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image65.png)

1. Choose the same settings as ReportingDB: **General Purpose**, **Serverless**, **Max vCores 2**, **Auto-pause delay 1 hour**. Click **Apply**. Set **Backup storage redundancy** to **Locally-redundant backup storage**.

    ![A screenshot of a computer screen Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image66.png)

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image67.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image68.png)

1. Click **Next** until you reach the **Tags** tab. Add **MigrationWave** with value **3**, and **Workload** with value **FinanceDB**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image69.png)

1. Click **Review + create**, then **Create**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image70.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image71.png)

1. **Expected result:** the deployment completes in 1 to 3 minutes. While you wait, return to Exercise 1, Task 2 if the VM has finished deploying.


### Task 3: Prove the selection criteria and AI readiness

1. Open **FinanceDB** and, and in the left menu, click **Query editor**.

1. On the **SQL server authentication** tab, enter `sqladminuser` and **Fabrikam!Lab2026**, and click **OK**. If you see a firewall error, click **Allowlist IP** in the message, then click **OK** again.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image72.png)

1. Open a **New Query**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image73.png)

1. Run this query. It tries to read a table in ReportingDB from FinanceDB, the kind of query InventoryDB runs against SalesDB:


### T-SQL

    SELECT TOP (1) name FROM ReportingDB.sys.objects;

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image74.png)

    **Expected result:** the query fails on purpose. The Messages pane shows “Reference to database and/or server name in ‘ReportingDB.sys.objects’ is not supported in this version of SQL Server” (error 40515), followed by “Failed to execute the query”. Each Azure SQL database is its own boundary, so cross-database queries fail. That is exactly why SalesDB and InventoryDB go to Managed Instance instead. Continue with the next step.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image75.png)

1. Clear the editor and run:

    **T-SQL**

    ```
    DECLARE @v VECTOR(3) = '[0.1, 2, 30]';
    SELECT CAST(@v AS nvarchar(100)) AS vector_value,
    DATABASEPROPERTYEX(DB_NAME(), 'ServiceObjective') AS
    service_objective;
    ```

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image76.png)

    **Expected result:** a vector_value of the three numbers and service_objective **GP_S_Gen5_2**. Azure SQL Database supports the vector data type at every compatibility level, with no setting to change.


## Exercise 3: Deploy Azure SQL Managed Instance

**Duration:** 10 minutes (plus any remaining wait for the deployment you started in Getting Started)

The managed instance has no public endpoint, so you test it from sqlvm-docs, which is in the same virtual network.

### Task 1: Confirm the deployment

1. In the portal search bar, type **SQL managed instances** and, under **Services**, select **Azure SQL Managed Instance**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image77.png)

1. Click **sqlmi-fabrikam-jd427**. If it isn’t listed yet, click the **Notifications** (bell) icon and click **Deployment in progress** to watch it.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image78.png)

1. On the **Overview** page, wait until **Status** shows **Ready**, clicking **Refresh** every few minutes.

1. Check that **Pricing tier** shows **Next-gen General Purpose**, **Standard-series (Gen5)**, **4 vCores** and **32 GB**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image79.png)

1. Copy the **Host** value,

    **for example** sqlmi-fabrikam-jd427.abc123def456.database.windows.net.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image80.png)


### Task 2: Prove the selection criteria

1. Open **sqlvm-docs**: in the portal search bar, type **Virtual machines**, select **Virtual machines**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image81.png)

1. Click **sqlvm-docs**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image82.png)

1. In the left menu, click **Operations \> Run command**, then click **RunPowerShellScript**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image83.png)

1. Paste this script, replace ***{host}*** with the Host value you copied, and click **Run**. It connects from the VM to the managed instance over the virtual network, creates the two Wave 2 databases, runs the kind of cross-database query that failed on Azure SQL Database, and checks SQL Agent: **PowerShell (Run command on sqlvm-docs)**

    . C:\Lab\Invoke-LabSql.ps1

    ```
    $mi = '*{host}*'
    Invoke-LabSql -Server $mi -Query "SELECT @@SERVERNAME AS server_name,
    SERVERPROPERTY('EngineEdition') AS engine_edition;"
    Invoke-LabSql -Server $mi -Query "IF DB_ID('SalesDB') IS NULL CREATE
    DATABASE SalesDB;"
    Invoke-LabSql -Server $mi -Query "IF DB_ID('InventoryDB') IS NULL
    CREATE DATABASE InventoryDB;"
    Invoke-LabSql -Server $mi -Database SalesDB -Query "IF
    OBJECT_ID('dbo.Orders') IS NULL BEGIN CREATE TABLE dbo.Orders (OrderID
    int PRIMARY KEY, CustomerID int NOT NULL); INSERT dbo.Orders VALUES
    (1, 10), (2, 20); END"
    Invoke-LabSql -Server $mi -Database InventoryDB -Query "SELECT
    COUNT(*) AS orders_seen_from_inventorydb FROM SalesDB.dbo.Orders;"
    Invoke-LabSql -Server $mi -Query "SELECT COUNT(*) AS agent_jobs FROM
    msdb.dbo.sysjobs;"
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image84.png)


### Expected result:
    - server_name is your instance name and engine_edition is 8, which
  identifies Azure SQL Managed Instance.

    - orders_seen_from_inventorydb is 2. InventoryDB reads SalesDB with a
  three-part name, the kind of query that returned error 40515 on Azure SQL Database.

    - agent_jobs is 0 or higher, with no error. Azure SQL Database has no
  msdb and no SQL Agent.


>[!Note] If the output shows a timeout or “network-related” error, check that the instance Status is Ready, that *{host}* was replaced with the full Host value, and that sqlvm-docs is in subnet snet-sqlvm of vnet-fabrikam (the VM’s Networking page).

![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image85.png)

### Task 3: Check AI readiness

1. Replace the script with this one, replace ***{host}*** again, and click **Run**: **PowerShell (Run command on sqlvm-docs)**

    . C:\Lab\Invoke-LabSql.ps1

    ```
    $mi = '*{host}*'
    Invoke-LabSql -Server $mi -Database SalesDB -Query "DECLARE @v
    VECTOR(3) = '[0.1, 2, 30]'; SELECT CAST(@v AS nvarchar(100)) AS
    vector_value;"
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image86.png)

    **Expected result**: a vector_value of the three numbers. The vector type works because you chose the SQL Server 2025 update policy.

    >[!Note] If you see “Cannot find data type VECTOR”, the instance has the SQL Server 2022 update policy. On the managed instance’s left menu, click Settings \> Compute + storage, set Update policy to SQL Server 2025, click Apply, wait for the update to finish, and run the script again.


### Task 4: Review the targets by migration wave

1. In the portal search bar, type **All resources** and select **All resources**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image87.png)

1. Set the **Resource group** filter to **ResourceGroup1**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image88.png)

1. Click **Add filter**, select **Tags**, choose **MigrationWave** and the value **1**, and click **Apply**.

    **Expected result:** only **ReportingDB** is listed: the Wave 1 pilot.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image89.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image90.png)

1. Change the filter value to **2**.

    **Expected result:** only **sqlmi-fabrikam-jd427** is listed. SalesDB and InventoryDB share one instance, so they move as one wave.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image91.png)

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image92.png)

1. Change the filter value to **3**.

    **Expected result:** **sqlvm-docs**: the targets that wait for a sign-off or an upgrade test.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image93.png)

1. Remove the tag filter: click the X on the **MigrationWave** filter chip above the list. Expected result: all resources in ResourceGroup1 are listed again. **vnet-fabrikam** has no **MigrationWave** tag: it is shared infrastructure, not a migration target.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%205/media/image94.png)


## Conclusion

In this lab, you deployed and validated the three Azure SQL migration targets for Fabrikam Retail’s approved migration plan. You built SQL Server 2025 on an Azure VM for DocsArchive, Azure SQL Database for ReportingDB and FinanceDB, and Azure SQL Managed Instance for SalesDB and InventoryDB. You then proved each choice against its selection criteria: FILESTREAM on the VM, the cross-database query boundary on Azure SQL Database, and cross-database queries and SQL Agent on Managed Instance. You tagged each target with its migration wave and confirmed that every destination supports the native vector data type. The targets are now ready for the migration and business continuity work in the labs that follow.
