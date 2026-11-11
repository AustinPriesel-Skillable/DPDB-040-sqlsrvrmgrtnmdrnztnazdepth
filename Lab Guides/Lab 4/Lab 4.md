# Lab 4: Estimate Azure Migration Costs

## Lab overview

Zava Retail, a home and lifestyle retailer with 140 stores and an online store, is moving its inventory database, **ZavaInventory**, off an 8-core SQL Server 2019 Enterprise server before its datacenter contract ends. Like most retailers, Zava must size for peak trading periods such as the festive season, not the average day, and justify every running cost to finance.

A consultant has priced a like-for-like 8-vCore Azure SQL Database. Your job is to replace that guess with evidence: measure the workload, size it, and find the lowest cost you can defend. ZavaInventory has no instance-level features, so all three Azure SQL targets are compatible, and this lab focuses on **size and cost**.

### Objectives

By the end of the lab you will be able to:
- Collect a performance baseline from a live SQL Server workload with
  the Az.DataMigration PowerShell module

- Generate SKU recommendations for Azure SQL Database, Azure SQL Managed
  Instance and SQL Server on Azure VM, and explain each recommendation reason

- Price the recommended SKUs in the Azure Pricing Calculator and
  identify the compute, license and storage parts of each bill

- Compare the monthly and 3-year cost of SQL Server on Azure VM, Managed
  Instance and Hyperscale

- Show how the target percentile and scaling factor change the
  recommended size

- Apply Azure Hybrid Benefit, reserved capacity and serverless compute
  correctly, including the license math


**Verify the tools (1 minute).** Open **Windows PowerShell as Administrator** and run:

  ```
  $PSVersionTable.PSVersion  
  sqlcmd -? | Select-Object -First 2  
  sqlcmd -S localhost -E -C -Q "SELECT @@VERSION;"
  ```

**Expected result:** PowerShell 5.1, a sqlcmd version line, and the SQL Server version string. If the last command fails with a certificate-trust error, keep -C on every sqlcmd command (already included in this guide).

### Lab conventions

- Prices change often, so this guide never prints a price. Each time you
  read a value from a command's output or from the calculator, you save it with the Save-Result command you create in Exercise 0, Task 1. Later scripts read those values back, so you never write anything in this guide.

- You use **two PowerShell windows**. **Window A** is your main window:
  keep it open for the whole lab, because later steps reuse its variables and helper commands. **Window B** runs the performance collector in Exercise 0, Task 5.

- Your results will differ slightly from other learners', because each
  VM measures its own workload. Check your work against the pattern in each **Expected result**, not against exact numbers.


## Exercise 0: Set up the source and start measuring

**Goal:** create ZavaInventory with about 200 MB of stock-movement history across 140 stores, put it under a steady workload of till sales and replenishment reports, and start a 6-minute performance collection that keeps running while you move on to Exercise 1.

### Task 1: Set lab variables

1. In **Window A** (Windows PowerShell as Administrator), run:

    ```
    New-Item -ItemType Directory -Path C:\Lab4\PerfData -Force | Out-Null
    Set-Location C:\Lab4
    $Region   = "centralindia"
    $Currency = "USD"
    $Hours    = 730
    $PerfDir  = "C:\Lab4\PerfData"
    $Results  = "C:\Lab4\results.csv"

    # Helper commands that keep your lab results in C:\Lab4\results.csv
    function Save-Result {
      param([Parameter(Mandatory)][string]$Name, [Parameter(Mandatory)]$Value)
      $rows = @()
      if (Test-Path $Results) { $rows = @(Import-Csv $Results | Where-Object { $_.Name -ne $Name }) }
      $rows += [pscustomobject]@{ Name = $Name; Value = "$Value"; SavedAt = (Get-Date -Format 'HH:mm:ss') }
      $rows | Export-Csv $Results -NoTypeInformation
      Write-Host "Saved $Name = $Value" -ForegroundColor Green
    }
    function Get-Result {
      param([Parameter(Mandatory)][string]$Name)
      $row = Import-Csv $Results | Where-Object { $_.Name -eq $Name }
      if (-not $row) { throw "No result named '$Name' yet. Save it with Save-Result first." }
      $row.Value
    }
    function Show-Results { Import-Csv $Results | Format-Table Name, Value, SavedAt -AutoSize }

    Save-Result LabStarted (Get-Date -Format 'yyyy-MM-dd HH:mm')
    ```

    **Expected result:** the prompt shows C:\Lab4, and a green line reads **Saved LabStarted = ...**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image1.png)


### Task 2: Install the Az.DataMigration module

1. In Window A, run the block below. It takes 1 to 2 minutes.

    ```
      [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
    Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Scope CurrentUser -Force | Out-Null
    Set-PSRepository -Name PSGallery -InstallationPolicy Trusted
    Install-Module -Name Az.DataMigration -RequiredVersion 1.1.1 -Scope CurrentUser -Force -AllowClobber
    Import-Module Az.DataMigration -RequiredVersion 1.1.1
    Get-Command -Module Az.DataMigration -Name Get-AzDataMigrationPerformanceDataCollection, Get-AzDataMigrationSkuRecommendation
    ```

    **Expected result:** two cmdlets are listed. Az.DataMigration also installs Az.Accounts as a dependency; you do not need to sign in to Azure for this lab.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image2.png)

    These two cmdlets wrap the same assessment engine that the Azure SQL migration tooling uses. The first collects performance counters from SQL Server; the second turns them into SKU recommendations. Both run locally, so you can use them on a customer's server before anything is connected to Azure.


### Task 3: Build the source database

1. In Window A, create the setup script and run it:

    ```
    @'
    CREATE DATABASE ZavaInventory;
    GO
    ALTER DATABASE ZavaInventory SET RECOVERY FULL;
    GO
    USE ZavaInventory;
    GO
    CREATE TABLE dbo.Products(
      ProductID   INT IDENTITY(1,1) PRIMARY KEY,
      ProductName NVARCHAR(100) NOT NULL,
      Category    NVARCHAR(30)  NOT NULL,
      UnitCost    DECIMAL(10,2) NOT NULL);
    CREATE TABLE dbo.StockMovements(
      MovementID   BIGINT IDENTITY(1,1) PRIMARY KEY,
      ProductID    INT NOT NULL REFERENCES dbo.Products(ProductID),
      StoreID      INT NOT NULL,                -- 1 to 140
      Quantity     INT NOT NULL,                -- positive = delivery, negative = sale
      MovementDate DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
      Notes        CHAR(1000) NOT NULL DEFAULT 'x');
    GO
    WITH n AS (SELECT TOP (500) ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS i FROM sys.all_objects)
    INSERT dbo.Products(ProductName, Category, UnitCost)
    SELECT CONCAT(N'Item ', i), CHOOSE(i % 4 + 1, N'Furniture', N'Kitchen', N'Decor', N'Bedding'), (i % 90) + 4.50 FROM n;
    GO
    WITH n AS (SELECT TOP (200000) ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS i
              FROM sys.all_objects a CROSS JOIN sys.all_objects b)
    INSERT dbo.StockMovements(ProductID, StoreID, Quantity, MovementDate)
    SELECT (i % 500) + 1, (i % 140) + 1,
          CASE WHEN i % 5 = 0 THEN 20 ELSE -((i % 3) + 1) END,
          DATEADD(MINUTE, -i, SYSUTCDATETIME())
    FROM n;
    GO
    CREATE OR ALTER PROCEDURE dbo.usp_SimulateLoad AS
    BEGIN
      SET NOCOUNT ON;
      -- Writes: 50 till sales from random stores
      INSERT dbo.StockMovements(ProductID, StoreID, Quantity)
      SELECT TOP (50) ProductID, (ABS(CHECKSUM(NEWID())) % 140) + 1, -1
      FROM dbo.Products ORDER BY NEWID();
      -- Reads: a 30-day replenishment report that scans and aggregates stock movements
      SELECT p.Category, COUNT_BIG(*) AS Movements,
            SUM(m.Quantity) AS NetUnits, SUM(m.Quantity * p.UnitCost) AS NetStockValue
      FROM dbo.StockMovements AS m
      JOIN dbo.Products AS p ON p.ProductID = m.ProductID
      WHERE m.MovementDate > DATEADD(DAY, -30, SYSUTCDATETIME())
      GROUP BY p.Category
      OPTION (MAXDOP 1);
    END;
    GO
    '@ | Set-Content C:\Lab4\01_source.sql

    sqlcmd -S localhost -E -C -b -i C:\Lab4\01_source.sql
    ```

    **Result**: The 500 rows are the products and the 200,000 rows are the stock movements. Creating the database, tables and procedure prints nothing. The 200,000-row insert can take up to a minute.

    ![A screenshot of a computer program Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image3.png)

1. Confirm the row count and size:

    `sqlcmd -S localhost -d ZavaInventory -E -C -W -Q "SELECT COUNT(*) AS Movements FROM dbo.StockMovements; SELECT CAST(SUM(size) * 8 / 1024.0 AS int) AS AllocatedMB FROM sys.database_files;"`

    **Expected result:** Movements = 200000, and AllocatedMB is roughly 500 to 700 MB (the test run showed 656). About a third is data; the rest is transaction log, which grows during the 200,000-row insert because the database uses the full recovery model. The Notes column is padding: it gives each stock movement a realistic row size so the storage and IO numbers in the recommendation are meaningful.

    ![A screen shot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image4.png)


### Task 4: Start the workload

1. In Window A, start two background sessions that call the procedure in a loop for 15 minutes:

    ```
    1..2 | ForEach-Object {
    Start-Job -Name "ZavaLoad$_" -ScriptBlock {
      $end = (Get-Date).AddMinutes(15)
      while ((Get-Date) -lt $end) {
        sqlcmd -S localhost -d ZavaInventory -E -C -Q "EXEC dbo.usp_SimulateLoad;" | Out-Null
      }
      }
    } | Out-Null
    Get-Job -Name ZavaLoad* | Format-Table Name, State -AutoSize
    ```

    ![A computer screen with text on it Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image5.png)

    The jobs stop by themselves after 15 minutes, so move on to step 2 and Task 5 without waiting.

1. Check that the server is busy:

    ```
    Get-Counter '\Processor(_Total)\% Processor Time' -SampleInterval 2 -MaxSamples 3 |
    ForEach-Object { [math]::Round($_.CounterSamples[0].CookedValue, 1) }
    ```

    **Expected result:** three CPU readings, one every 2 seconds, well above idle. On the test VM they were around 80% (80.4, 83, 77.9); your values depend on the VM size. Part of this load comes from starting a new sqlcmd process for every call, which is fine: the collector still measures how hard SQL Server works.

    If all three readings are below 30%, your VM has more cores than the test VM. Start a third job with the Task 4 step 1 command, changing 1..2 to 3..3, so the recommendation has a meaningful load to size.

    ![A screen shot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image6.png)


### Task 5: Start the performance collection (Window B)

1. Open a **second Windows PowerShell window as Administrator**. This is **Window B**.

1. Run the collector. It stops by itself after 6 minutes (-Time 360):

    ```
    Import-Module Az.DataMigration
    Get-AzDataMigrationPerformanceDataCollection `
    -SqlConnectionStrings "Data Source=localhost;Initial Catalog=master;Integrated Security=True;Encrypt=True;TrustServerCertificate=True" `
    -OutputFolder "C:\Lab4\PerfData" `
    -PerfQueryInterval 30 `
    -StaticQueryInterval 60 `
    -NumberOfIterations 2 `
    -Time 360
    ```

    **Result:** The collector samples every 30 seconds and writes an **Aggregated ...** line every 2 samples, so a new pair of iteration lines and an aggregation line appear about once a minute until the 6 minutes are up. The **Collected static configuration data** line (shown in green) repeats every 60 seconds.

    ![A screen shot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image7.png)

1. Leave **Window B** running and go straight to Exercise 1 in Window A. Do not press Enter in Window B; that stops the collection early.

    **Why these settings?** -PerfQueryInterval 30 samples every 30 seconds, and -NumberOfIterations 2 (the minimum) saves the data to disk every minute, so you get usable data inside a lab. On a real customer server you would keep the defaults and collect across a full business cycle, including month-end and seasonal peaks, because the recommendation can only reflect the load it saw.


## Exercise 1: Calculate Azure SQL deployment costs

**Goal:** turn the measured workload into SKU recommendations for all three Azure SQL targets, then price them against the consultant's like-for-like estimate.

### Task 1: Prepare the estimate while the collector runs

1. Open the [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) and log in with the given credentials.

    ![A screenshot of a login page Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image8.png)

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image9.png)

1. Find the **Azure SQL Database** tile (under **Popular**, or type it in **Search products**) and select **Add to estimate**. Scroll down to the estimate area, where the new item appears with its settings.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image10.png)

1. Click in the large name box that reads **Azure SQL Database** (just above the settings), clear it and type +++Like-for-like-8vCore+++. The item's header row updates to the new name.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image11.png)

1. **Set these values.** The item opens with defaults such as East US and Hyperscale, so change every field:

    - **Region:** Central India
    - **Type:** Single Database
    - **Purchase Model:** vCore
    - **Service Tier:** General Purpose
    - **Compute Tier:** Provisioned
    - **Hardware Type:** Standard-series (Gen 5)
    - **Instance:** 8 vCore
    - **Disaster Recovery:** Primary or Geo replica (the default)
    - **Compute \> Redundancy:** Locally Redundant (the default), with 1
    database × 730 hours

    - **Savings Options \> Compute:** Pay as you go. Leave **Savings plan
    for databases** and **Reservations** unselected.

    - **Savings Options \> SQL License:** Pay as you go. This is the
    license-included price; the other option, **Bring Your Own License (Azure Hybrid Benefit)**, is what you use in Exercise 3.

    - **Storage \> Data:** 32 GB. The **Log** size is calculated for you.
    - **Backup Storage:** leave the defaults (LRS redundancy, 0 GB
    point-in-time restore, long-term retention switched off).

    ![A screenshot of a web page Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image12.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image13.png)

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image14.png)


1. **Check the currency:** scroll to the bottom of the estimate. Next to the estimated monthly cost, check that Currency shows **United States – Dollar ($) USD**, and change it if not. The currency drop-down only appears once the estimate has at least one item.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image15.png)

    **Expected result:** the item shows separate monthly figures for compute and for the SQL license under Savings Options, plus storage lines, and the Monthly cost at the bottom of the item adds them up. On the test run in Central India the monthly cost was about $1,567 (compute about $978, SQL license about $584, storage about $5). Prices change, so your figure may differ slightly. Notice that the license is a separate charge: Exercise 2 shows why that matters.

1. **Save it.** Find the item's estimated monthly cost on the right-hand side of the calculator item, for example $1567.10.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image16.png)

1. Switch to Window A and save it as a plain number, without the $ sign or commas, for example **1567.10**. This is the number you are trying to beat with evidence:

    >[!Note] Replace 0 with the monthly cost.

    Save-Result CostLikeForLike 0 **\# replace 0 with the monthly total**

    ![A screen shot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image17.png)

1. Look at what the collector has written so far: In **Window A**, look at what the collector has written so far (the command uses the **$PerfDir** variable, which only exists in Window A):

    ```
    Get-ChildItem $PerfDir -Recurse -File | Sort-Object LastWriteTime -Descending |
    Select-Object Name, @{n='SizeKB';e={[int]($_.Length/1KB)}}, LastWriteTime | Format-Table -AutoSize
    ```

    **Expected result:** files appear and grow each minute. File names vary by engine version; you will see static configuration data (cores, memory, edition, database files) and aggregated performance counters. These files are the evidence the recommendation will use in **Task 2**. three CSV files, each named after your server and instance (*{server}*,MSSQLSERVER\_...)


### Task 2: Generate SKU recommendations

1. Wait until Window B returns to the prompt. This happens about 6 minutes after you started it.

    **Expected result:** Window B shows the last **Aggregated 12 raw data points to 4 performance counters** line, then a line giving the engine's log folder. The log folder line is normal and is not an error; if a later step fails, the engine's logs are in that folder. You can close Window B now: the rest of the lab runs in Window A.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image18.png)

1. In **Window A**, generate one recommendation per target:

    ```
      foreach ($target in 'AzureSqlDatabase', 'AzureSqlManagedInstance', 'AzureSqlVirtualMachine') {
      Write-Host "`n===== $target =====" -ForegroundColor Cyan
      Get-AzDataMigrationSkuRecommendation -OutputFolder $PerfDir -TargetPlatform $target `
        -DatabaseAllowList "ZavaInventory" -DisplayResult -Overwrite
    }
    ```

    **Expected result:** for each target, the output shows **Starting SKU recommendation...**, **Performing aggregation for instance ...**, then a **SKU recommendation** block followed by **Recommensdation reasons**, and ends with **SKU recommendation report saved to folder C:\Lab4\PerfData**. Scroll up in Window A to see all three blocks. The first reason line in each block is the CPU evidence: the measured utilization percentage of the source server's cores, and the vCores that covers it.

1. **Save it.** Read the values from the output and save them.

    **Important Note: Replace each 0 with your own values. Refer the table given below to find the values in the output of the step 2 command.**

    ```
    Save-Result CpuUtilPct     0          # Managed Instance block: the N in "Considering the N% utilization"
    Save-Result SourceCores    0          # same line: the X in "of the X cores"
    Save-Result SqlDbTier  'tier'         # SQL Database block, Compute line, e.g. HyperScale
    Save-Result SqlDbVcores    0          # SQL Database block, Compute line: cores
    Save-Result SqlDbStorageGB 0          # SQL Database block, Storage line
    Save-Result MiTier     'tier'         # Managed Instance block, Compute line, e.g. NextGenGeneralPurpose
    Save-Result MiVcores       0          # Managed Instance block, Compute line: cores
    Save-Result MiStorageGB    0          # Managed Instance block, Storage line
    Save-Result VmSize     'VM-size'      # VM block, Compute line, e.g. D2ads_v5
    Save-Result VmDisks    'disks'        # VM block, Data and Log storage, e.g. 2 x PremiumSSDV2 1 GB
    Show-Results

    "Measured CPU need: {0:N2} vCores ({1}% of {2} cores)" -f ([decimal](Get-Result CpuUtilPct) / 100 * [decimal](Get-Result SourceCores)), (Get-Result CpuUtilPct), (Get-Result SourceCores)
    ```


### Find the values as per this table and update in the above command and run it in the PowerShell:

    | # | Name | Block in the output | Line to find (value in bold) | Test VM value |
    |---|------|---------------------|------------------------------|---------------|
    | 1 | CpuUtilPct | AzureSqlManagedInstance | Considering the **29**% utilization of the 4 cores ... (first line under Recommendation reasons) | `29` |
    | 2 | SourceCores | AzureSqlManagedInstance | Considering the 29% utilization of the **4** cores ... (same line) | `4` |
    | 3 | SqlDbTier | AzureSqlDatabase | Compute: **HyperScale** - 2 cores | `'HyperScale'` |
    | 4 | SqlDbVcores | AzureSqlDatabase | Compute: HyperScale - **2** cores | `2` |
    | 5 | SqlDbStorageGB | AzureSqlDatabase | Storage: **10** GB (not the 102400 GB in the reasons) | `10` |
    | 6 | MiTier | AzureSqlManagedInstance | Compute: Gen5 - **NextGenGeneralPurpose** - 4 cores | `'NextGenGeneralPurpose'` |
    | 7 | MiVcores | AzureSqlManagedInstance | Compute: Gen5 - NextGenGeneralPurpose - **4** cores | `4` |
    | 8 | MiStorageGB | AzureSqlManagedInstance | Storage: **32** GB | `32` |
    | 9 | VmSize | AzureSqlVirtualMachine | Compute: standardDADSv5Family - **D2ads_v5** - 2 Available vCPU | `'D2ads_v5'` |
    | 10 | VmDisks | AzureSqlVirtualMachine | Data storage: Type: **PremiumSSDV2**, Size: **1 GB** ... plus Log storage: Type: **PremiumSSDV2**, Size: **1 GB** ... (1 data + 1 log disk) | `'2 x PremiumSSDV2 1 GB'` |

    **Expected result:** a green **Saved** line for each of the ten values. Then Show-Results prints a table of 12 rows with their save times: the ten new values plus **LabStarted** and **CostLikeForLike** from earlier. The last line shows your measured CPU need, for example **Measured CPU need: 1.16 vCores (29% of 4 cores)** on the test VM.

    If the recommendation output says there is not enough performance data, the collector stopped early. Rerun Exercise 0, Task 5 with -Time 240, then repeat this task.

    **Read the reasons, not just the SKU.** Each reason line explains one sizing constraint: CPU, memory, storage or IO. The recommended SKU is the smallest one that satisfies all of them at once, so whichever line is closest to the SKU's limit is the constraint that decided the size.

    **Checkpoint question:** the measured CPU need (about 1.2 vCores on the test VM) is far below the 8 cores of Zava's server. Why? \[The engine sizes from measured utilization at the 95th percentile, not from allocated hardware. A like-for-like estimate pays for capacity the workload never uses.\]

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image19.png)

### Task 3: Price the recommendations

1. Run this command to display the recommended sizes so you have them in front of you:

    `"SQL Database: {0} vCores, {1} GB | Managed Instance: {2} vCores, {3} GB" -f (Get-Result SqlDbVcores), (Get-Result SqlDbStorageGB), (Get-Result MiVcores), (Get-Result MiStorageGB)`

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image20.png)

1. Create the General Purpose item at the recommended size by cloning the like-for-like item:

    - **Find the item:** in the calculator, scroll down to the estimate
    area to the Like-for-like-8vCore item. Its header row shows the name, a summary of the settings, two small icons, and the **Upfront**, **Monthly** and **Annual** cost.

    - **Clone it:** select the **Clone** icon (two overlapping squares)
    in that header row.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image21.png)

    - Scroll down and observe that a copy of the item appears below it with
    the same settings.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image22.png)

    - **Rename it:** in the copy's large name box, clear the text and type
    **SQLDB-GP-Recommended.**

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image23.png)

    - **Change only these two values** (use the numbers from step 1):
    - **Instance:** the SqlDbVcores value, for example 2 vCore on the test
    run.

    - **Storage \> Data:** the SqlDbStorageGB value, for example 10 GB on
    the test run

    ![A screenshot of a computer screen Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image24.png)

    - Update both the values in **the SQLDB-GP-Recommended** item:
    - Update the **Instance** field as **2 vCore**
    - Under the **Storage \> Data**, the first box shows **32** with GB
    under it; select it, clear it and type the +++10+++ number. The Log line below updates by itself.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image25.png)
  
    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image26.png)

    - **Leave everything else as it is**, including **Service Tier: General
    Purpose**. Even if the engine recommended Hyperscale (as on the test run), you price General Purpose here at the recommended size and add Hyperscale in Exercise 2, so you can compare the two.

    - **Read the price:** the copy's header row shows its **Monthly** cost.
    You save it in step 4.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image27.png)

    The original **Like-for-like-8vCore** item must stay unchanged, so check that you edited the copy: its header row should show the new name.


1. Add the **Managed Instance** item at the recommended size:

    - **Add it:** scroll up to **Search products**, type Azure SQL Managed
    Instance, and select **Add to estimate** on its tile.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image28.png)

    - Scroll down to the estimate area, where the new item appears below
    your SQL Database items.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image29.png)

    - **Rename it:** in the item's large name box, clear the text and type

    `SQLMI-GP-Recommended`

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image30.png)

    - **Set these values:**
    - **Region**: Central India
    - **Type** (or Deployment option): Single instance
    - **Service** **Tier**: Next-gen General Purpose, to match MiTier
    (NextGenGeneralPurpose on the test run). If the calculator doesn't list it, choose General Purpose.

    - **Hardware** **Type**: Standard-series (Gen 5)
    - **Instance** (**vCores**): the MiVcores value from step 1, for example
    4 vCore on the test run

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image31.png)

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image32.png)

    - **Savings Options \> Compute:** Pay as you go. Leave any savings plan
    or reservation unselected.

    - **Savings Options \> SQL License:** Pay as you go (license included)

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image33.png)

    - **Storage:** the MiStorageGB value from step 1, for example **32 GB**
    on the test run

    - **Backup storage:** leave the defaults

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image34.png)

    - **Read the price:** the item's header row shows its **Monthly** cost,
    at the top right next to Upfront and Annual. You save it in step 4.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image35.png)


1. **Save** it, then compare with the consultant's estimate:

    ```
    Save-Result CostSqlDbGp 0    # monthly total of SQLDB-GP-Recommended
    Save-Result CostSqlMi   0    # monthly total of SQLMI-GP-Recommended
    $pct = [decimal](Get-Result CostSqlDbGp) / [decimal](Get-Result CostLikeForLike) * 100
    "Evidence-based SQL Database cost is {0:N0}% of the like-for-like estimate" -f $pct
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image36.png)

    **Expected result:** the percentage is well below 100. On this small workload, the Managed Instance cost is usually higher than the SQL Database cost, because Managed Instance has a higher minimum size per instance. That is a cost floor you will meet again in Exercise 3.


## Exercise 2: Compare SQL VM, Managed Instance and Hyperscale costs

**Goal:** see what each target's bill is actually made of, price all three targets at the measured requirement, and compare them over the 3-year horizon the CFO cares about, including the work that the price does not cover.

### Task 1: See what each bill is made of with the Retail Prices API

The [Azure Retail Prices API](https://learn.microsoft.com/rest/api/cost-management/retail-prices/azure-retail-prices) is a public REST endpoint with no sign-in. It returns the meters that Azure bills against, which shows you how each price is built.

1. In Window A, load the helper function:

    ```
    function Get-RetailPrices {
    param([Parameter(Mandatory)][string]$Filter)
    $uri  = "https://prices.azure.com/api/retail/prices?currencyCode='$Currency'&`$filter=" + [uri]::EscapeDataString($Filter)
    $rows = New-Object System.Collections.Generic.List[object]
    while ($uri) {                       # the API pages its results
      $page = Invoke-RestMethod -Uri $uri -Method Get
      if ($page.Items) { $rows.AddRange([object[]]$page.Items) }
      $uri = $page.NextPageLink
    }
    $rows
    }
    ```

    ![A computer screen with text on it Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image37.png)

1. Pull the regional meters for SQL Database and Managed Instance, plus the license meters, which are listed under the Global region. This takes about a minute:

    ```
    $meters  = @()
    $meters += Get-RetailPrices "serviceName eq 'SQL Database' and armRegionName eq '$Region' and priceType eq 'Consumption'"
    $meters += Get-RetailPrices "serviceName eq 'SQL Managed Instance' and armRegionName eq '$Region' and priceType eq 'Consumption'"
    $meters += Get-RetailPrices "serviceName eq 'SQL Database' and armRegionName eq 'Global' and priceType eq 'Consumption'"
    $meters += Get-RetailPrices "serviceName eq 'SQL Managed Instance' and armRegionName eq 'Global' and priceType eq 'Consumption'"
    "{0} meters downloaded" -f $meters.Count
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image38.png)

1. Group the meters for the three tiers you are comparing:

    ```
    $meters | Where-Object { $_.productName -match 'General Purpose|Hyperscale' } |
    Group-Object productName | Sort-Object Name |
    Select-Object @{n='Product';e={$_.Name}}, @{n='Meters';e={$_.Count}},
                  @{n='Region';e={($_.Group.armRegionName | Sort-Object -Unique) -join ', '}} |
    Format-Table -AutoSize -Wrap
    ```

    **Expected result:** a table of about 25 products with a meter count and region for each. The pattern is the point of this step:
    - **General Purpose** (SQL Database and Managed Instance) has a
    **Compute** product in centralindia and a separate **SQL License** product in Global. The license-included price you saw in the calculator is these two added together.

    - **Hyperscale** has a Compute product too, but its **SQL License**
    product only applies to Hyperscale databases created before December 2023. New Hyperscale databases, like the one the engine recommended, have no license charge.

    - The other rows (Serverless, Storage, Gen4, Premium Series, DC-Series)
    are other options and storage meters that this lab doesn't use.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image39.png)


1. Look at the individual Managed Instance General Purpose meters:

    ```
    $v   = [int](Get-Result MiVcores)          # 4 on the test run
    $sel = $meters | Where-Object {
      ($_.productName -like '*Managed Instance General Purpose - Compute Gen5' -and $_.skuName -eq "$v vCore") -or
      ($_.productName -like '*Managed Instance General Purpose - SQL License')
    }
    $sel | Format-Table productName, skuName, retailPrice, unitOfMeasure, armRegionName -AutoSize
    ```

    **Expected result:** two rows, each priced per hour. The compute meter is priced for the whole 4-vCore size, but the license meter's skuName is just **vCore**: it is priced **per vCore per hour**, so the license cost grows with every vCore you add.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image40.png)

    **Checkpoint question:** which meter does Azure Hybrid Benefit remove, and what does that mean for new Hyperscale databases? \[The SQL License meter. New Hyperscale databases have no license meter, so Azure Hybrid Benefit has nothing to remove and does not apply.\]

1. Turn the two hourly rates into monthly costs:

    ```
    $compute = ($sel | Where-Object { $_.productName -like '*Compute*' }).retailPrice
    $license = ($sel | Where-Object { $_.productName -like '*SQL License*' }).retailPrice
    "Compute:     {0,8:N2} USD/month" -f ($compute * $Hours)
    "SQL license: {0,8:N2} USD/month ({1} vCores x {2} per vCore-hour)" -f ($license * $v * $Hours), $v, $license
    "Total:       {0,8:N2} USD/month, before storage" -f (($compute + $license * $v) * $Hours)
    ```

    ![A screen shot of a computer program Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image41.png)

    **Expected result:** on the test run, **Compute: 488.92**, **SQL license: 291.90** and **Total: 780.82** USD per month. About 37% of the license-included compute price is the SQL license, which is exactly what Azure Hybrid Benefit removes. If the Managed Instance item in your calculator estimate uses Next-gen General Purpose, its price can differ from these General Purpose meters; the split between compute and license is the lesson here, not an exact match.


### Task 2: Price SQL Server on Azure VM and Hyperscale

1. Display the sizes you need:

    `"VM: {0}, disks: {1} | Hyperscale: {2} vCores, {3} GB" -f (Get-Result VmSize), (Get-Result VmDisks), (Get-Result SqlDbVcores), (Get-Result SqlDbStorageGB)`

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image42.png)

1. **Add it:** scroll up to Search products, type Virtual Machines, and select Add to estimate on its tile. Scroll down to the estimate area, where the new item appears.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image43.png)

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image44.png)

1. **Rename it:** in the item's large name box, clear the text and type

    `SQLVM-Recommended`

1. **Set these values.** The item opens with defaults such as East US and Linux, so check every field:

    - **Region:** Central India
    - **Operating system:** Windows
    - **Type:** SQL Server (Zava's current edition)
    - **Tier:** Standard
    - **Instance:** the VmSize value from step 1, for example **D2ads v5**
    on the test run. If it isn't listed, pick the nearest size with the same vCPU count and memory (2 vCPU, 8 GB).

    - **Savings Options \> Compute:** Pay as you go. Leave any savings plan
    or reservation unselected.

    - **Licensing (OS and SQL Server):** keep the license-included options.
    Do not select Azure Hybrid Benefit.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image45.png)


1. **Add the disks from VmDisks:**

    - Scroll down inside SQLVM-Recommended to its **Managed Disks** section.
    Select **Premium SSD v2** in the Tier field.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image46.png)

    - **Set these values** (on the test run, VmDisks was 2 x PremiumSSDV2 1
    GB):
    - **Tier** (or **Disk type**): Premium SSD v2
    - **Storage Costs:** this line multiplies disk size × number of disks
    × hours × the price per GiB/hour. Set:
    - **Disk Size (GiB):** 1
    - **Number of disks:** 2 (one for data, one for log). The default is
    1, so change it.

    - **Hours:** leave 730
    - **IOPS:** leave **Number of IOPS** at 3000. The note above it
    confirms that up to 3,000 IOPS are included at no additional cost for a 1 GiB disk.

    - **Throughput** (further down, if shown): leave the default of 125
    MB/s, which is also included.

    - **TempDB** needs no disk: it uses the VM's local SSD, which is
    included in the VM price.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image47.png)


1. **Add the Hyperscale item:**

    - **Add it:** search for **Azure SQL Database** and select **Add to
    estimate**. Scroll down to the new item.

    ![A screenshot of a website Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image48.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image49.png)

    - **Rename it:** in the item's large name box, type

    `SQLDB-Hyperscale-Recommended`

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image50.png)


1. **Set these values:**

    - **Region**: Central India
    - **Type**: Single Database
    - **Purchase** **Model**: vCore
    - **Service** **Tier**: Hyperscale
    - **Compute** **Tier**: Provisioned
    - **Hardware** **Type**: Standard-series (Gen 5)
    - **Instance**: the SqlDbVcores value from step 1, for example 2 vCore
    on the test run

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image51.png)

    - **Savings** **Options** **\>** **Compute**: Pay as you go

    ![A white background with black text Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image52.png)

    - **High-availability replicas: 0**

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image53.png)

    - **Storage**: the SqlDbStorageGB value from step 1, for example 10 GB
    on the test run

    - Notice that Savings Options has no SQL License choice for
    Hyperscale, unlike the General Purpose item: there is no license charge to pay or to remove.

    ![A screenshot of a website Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image54.png)


1. Note the monthly total of **SQLVM-Recommended** and **SQLDB-Hyperscale-Recommended**

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image55.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image56.png)

1. **Save it. Replace 0 with the above noted monthly values.**

    ```
    Save-Result CostSqlVm      0    # monthly total of SQLVM-Recommended, including the disk
    Save-Result CostHyperscale 0    # monthly total of SQLDB-Hyperscale-Recommended
    ```

    ![A screen shot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image57.png)

    **Expected result:** two green **Saved** lines. The VM price includes the Windows and SQL Server Enterprise licenses; the Hyperscale price has no SQL license component at all.


### Task 3: Build the 3-year comparison

1. Run the comparison. It reads the costs you saved, so there is nothing to type in:

    ```
      $targets = [ordered]@{
      'SQL Server on Azure VM'       = 'CostSqlVm'
      'SQL Managed Instance (GP)'    = 'CostSqlMi'
      'SQL Database Hyperscale'      = 'CostHyperscale'
      'SQL Database General Purpose' = 'CostSqlDbGp'
    }
    $targets.GetEnumerator() | ForEach-Object {
      $m = [decimal](Get-Result $_.Value)
      [pscustomobject]@{
        Target       = $_.Key
        MonthlyUSD   = [math]::Round($m, 2)
        ThreeYearUSD = [math]::Round($m * 36, 2)
      }
    } | Sort-Object ThreeYearUSD | Format-Table -AutoSize
    ```

    **Expected result:** a table of the four targets sorted from cheapest to most expensive over 3 years.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image58.png)

1. Compare that ranking against what each price includes:

    | What the price covers | SQL Server on Azure VM | SQL Managed Instance (GP) | SQL Database Hyperscale |
    |-----------------------|------------------------|---------------------------|-------------------------|
    | High availability | Not included: you build and pay for a second VM with an availability group or failover cluster | Built in | Built in; add HA replicas for faster failover |
    | Backups | You configure them (for example, automated backup or Azure Backup) | Automated | Automated, snapshot-based |
    | OS and SQL Server patching | Your team | Microsoft | Microsoft |
    | Storage growth | Add or resize disks yourself | Up to 16 TB per instance | Up to 128 TB, grows automatically |
    | SQL Server license | In the price (or bring your own) | In the price (or bring your own) | None |

    **Checkpoint question:** on the test run the VM was already the most expensive option. Why would the real gap be even larger for Zava's two-person DBA team? \[The VM price includes no high availability, and the team still owns patching and backups. Matching PaaS availability needs a second VM with its own Enterprise license, which roughly doubles the cost, plus ongoing staff time. The PaaS prices already include all of that.\]

    **Expected outcome:** ZavaInventory has no instance-scoped features, so the choice comes down to Azure SQL Database (General Purpose or Hyperscale) for the lowest management effort. Exercise 3 decides between them once licensing and reservations are applied.


## Exercise 3: Optimize deployment sizing and licensing

**Goal:** test how sensitive the size is to your sizing assumptions, then cut the cost with the licensing and commitment options Zava is entitled to, and finish with a recommendation the CFO can approve.

### Task 1: Test the sizing assumptions

The recommendation in Exercise 1 used two defaults: the **95th percentile** of the measured data, and a **scaling factor of 100%** (no headroom). Here you rerun it on the same data with stricter assumptions.

1. In Window A, run:

    ```
    $runs = @(
    @{ Label = 'P95, 100% (baseline)'; Pct = '95'; Scale = '100' },
    @{ Label = 'P99, 100%';            Pct = '99'; Scale = '100' },
    @{ Label = 'P95, 130% headroom';   Pct = '95'; Scale = '130' }
    )
    foreach ($r in $runs) {
      Write-Host "`n===== $($r.Label) =====" -ForegroundColor Cyan
      Get-AzDataMigrationSkuRecommendation -OutputFolder $PerfDir -TargetPlatform AzureSqlManagedInstance `
        -DatabaseAllowList "ZavaInventory" -TargetPercentile $r.Pct -ScalingFactor $r.Scale -DisplayResult -Overwrite
    }
    ```

    ![A screenshot of a computer program Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image59.png)

1. **Save it.** Each run prints a block headed ===== P95, 100% (baseline) =====, ===== P99, 100% ===== or ===== P95, 130% headroom =====. Read these values in order, then save and compare:

    | # | Name | Run block | Line to find (value in bold) | Test VM value |
    |---|------|-----------|------------------------------|---------------|
    | 1 | CpuUtilP99 | P99, 100% | Considering the **29**% utilization of the 4 cores ... | `29` |
    | 2 | VcoresP99 | P99, 100% | Compute: Gen5 - NextGenGeneralPurpose - **4** cores | `4` |
    | 3 | IopsBaseline | P95, 100% (baseline) | Data and log files require **15.656** IOPS ... | `15.656` |
    | 4 | IopsScaled | P95, 130% headroom | Data and log files require **20.352** IOPS ... | `20.352` |
    | 5 | VcoresScaled | P95, 130% headroom | Compute: Gen5 - NextGenGeneralPurpose - **4** cores | `4` |

    ```
    Save-Result CpuUtilP99   0    # P99 run: the N in "Considering the N% utilization"
    Save-Result VcoresP99    0    # P99 run: cores on the Compute line
    Save-Result IopsBaseline 0    # P95 baseline run: the N in "Data and log files require N IOPS"
    Save-Result IopsScaled   0    # P95 + 130% run: the same IOPS line
    Save-Result VcoresScaled 0    # P95 + 130% run: cores on the Compute line
    "P95: {0}% -> {1} vCores | P99: {2}% -> {3} vCores | P95 + 130%: {4} vCores" -f `
      (Get-Result CpuUtilPct), (Get-Result MiVcores), (Get-Result CpuUtilP99), (Get-Result VcoresP99), (Get-Result VcoresScaled)
    "IOPS requirement: {0} -> {1} with 130% headroom ({2:N2}x)" -f (Get-Result IopsBaseline), (Get-Result IopsScaled), ([decimal](Get-Result IopsScaled) / [decimal](Get-Result IopsBaseline))
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image60.png)

    **What this tells you.** P99 sizes for nearly every peak and suits spiky, customer-facing systems. A scaling factor adds headroom for growth or for peaks your collection window didn't capture. Both raise cost, so state the setting you used next to every estimate you hand to a customer. Optional: add -ElasticStrategy to the command to see the engine's elastic sizing model, which profiles usage patterns instead of using a single percentile.


### Task 2: Apply Azure Hybrid Benefit and reserved capacity

Zava already owns SQL Server licenses and can commit for 3 years. You apply both levers to the same calculator items, one after the other, and save the price after each so you can see what each one is worth.

1. In your estimate, open the **SQLDB-GP-Recommended** item. Under **Savings Options \> SQL License**, select **Bring Your Own License (Azure Hybrid Benefit)**.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image61.png)

1. Note the monthly total of SQLDB-GP-Recommended and replace 0 with this value in the next step.

    ![A screenshot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image62.png)

1. **Save it:**

    ```
    Save-Result CostSqlDbGpAhb 0 \# monthly total of SQLDB-GP-Recommended
    with Azure Hybrid Benefit
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image63.png)

1. Work out the licenses this consumes. For General Purpose, 1 Standard core license covers 1 vCore, and 1 Enterprise core license covers 4 vCores ([licensing ratios](https://www.microsoft.com/licensing/news/expanded-ahb-rights-for-microsoft-sql-server)). Run the license ledger:

    ```
    $vcores = [int](Get-Result SqlDbVcores)
    if ($vcores -le 4) { $std = $vcores; $ent = 0 } else { $std = 0; $ent = [math]::Ceiling($vcores / 4) }
    @(
      [pscustomobject]@{ Pool = 'SQL Server Standard core + SA';   Owned = 4; Used = $std; Remaining = 4 - $std },
      [pscustomobject]@{ Pool = 'SQL Server Enterprise core + SA'; Owned = 8; Used = $ent; Remaining = 8 - $ent }
    ) | Format-Table -AutoSize
    ```

    ![A screen shot of a computer program Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image64.png)

    **Expected result:** The database uses 2 of Zava's 4 spare Standard licenses (1 core license per vCore), leaving all 8 Enterprise licenses free for other workloads, or to retire when the on-premises server is decommissioned. No pool goes below zero.

    Azure Hybrid Benefit gives you a 180-day dual-use right for migration, so the on-premises server can keep running on the same licenses during cutover. After that, a license must not be used in both places.

    **Now add the 3-year reservation** on top of Azure Hybrid Benefit, and to the Hyperscale item for comparison:

1. On **SQLDB-GP-Recommended**, under **Savings Options \> Compute**, select **Reservations \> 3 year reserved**. Keep **SQL License** on **Bring Your Own License (Azure Hybrid Benefit)** from Task 2.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image65.png)

1. On **SQLDB-Hyperscale-Recommended**, under **Savings Options \> Compute**, select **Reservations \> 3 year reserved**.

    ![A white background with black lines Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image66.png)

1. **Save it** and compare the two optimized options:

    ```
    Save-Result CostSqlDbGpOptimized   0    # SQLDB-GP-Recommended: AHB + 3-year reserved
    Save-Result CostHyperscaleReserved 0    # SQLDB-Hyperscale-Recommended: 3-year reserved

    "General Purpose (AHB + reserved): {0} | Hyperscale (reserved): {1} USD/month" -f (Get-Result CostSqlDbGpOptimized), (Get-Result CostHyperscaleReserved)
    ```

    ![A screen shot of a computer Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image67.png)

    **Expected result:** At list price Hyperscale was cheapest, because it carries no license fee. Once Azure Hybrid Benefit removes General Purpose's license charge, that advantage disappears, because Zava already owns the license, and the reservation then makes General Purpose the cheaper option. Hyperscale still earns its place when a database needs to grow past the General Purpose storage limit or scale out reads, but ZavaInventory needs neither.

    Reservations discount compute only. Storage is still billed at pay-as-you-go rates.


### Task 3: Produce the CFO summary

1. Run the summary. It picks the cheaper optimized production option, adds the test copy, and saves the result to a file:

    ```
    $gp   = [decimal](Get-Result CostSqlDbGpOptimized)
    $hs   = [decimal](Get-Result CostHyperscaleReserved)
    $like = [decimal](Get-Result CostLikeForLike)
    if ($gp -le $hs) { $prod = $gp; $prodName = 'SQL Database General Purpose, Azure Hybrid Benefit, 3-year reserved' }
    else             { $prod = $hs; $prodName = 'SQL Database Hyperscale, 3-year reserved' }

    $summary = @(
      [pscustomobject]@{ Line = "Consultant's like-for-like estimate"; Configuration = 'SQL Database GP, 8 vCores, license included, pay as you go'; MonthlyUSD = [math]::Round($like, 2) },
      [pscustomobject]@{ Line = 'Recommended, optimized'; Configuration = "$prodName, $(Get-Result SqlDbVcores) vCores"; MonthlyUSD = [math]::Round($prod, 2) },
      [pscustomobject]@{ Line = 'Monthly saving'; Configuration = ('{0:N0}% below the like-for-like estimate' -f (($like - $prod) / $like * 100)); MonthlyUSD = [math]::Round($like - $prod, 2) },
      [pscustomobject]@{ Line = '3-year saving'; Configuration = 'Monthly saving x 36'; MonthlyUSD = [math]::Round(($like - $prod) * 36, 2) }
    )
    $summary | Format-Table -AutoSize -Wrap
    $summary | Export-Csv C:\Lab4\cfo-summary.csv -NoTypeInformation
    ```

    ![A screen shot of a computer program Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image68.png)

    The evidence-based, optimized recommendation costs about 7% of the consultant's estimate: a saving of about $1,455 a month, or about $52,400 over the 3-year commitment. The clean-up step deletes C:\Lab4, so copy the file elsewhere first if you want to keep it.

    **Checkpoint question:** which single change saved the most money? \[Usually right-sizing from measured data (like-for-like versus CostSqlDbGp), not the discounts. Licensing and reservations multiply a saving; sizing decides what they multiply.\]


## Clean up

1. In Window A, stop the workload, drop the database and remove the lab folder:

    ```
    Get-Job -Name ZavaLoad* | Stop-Job -PassThru | Remove-Job
    sqlcmd -S localhost -E -C -Q "ALTER DATABASE ZavaInventory SET SINGLE_USER WITH ROLLBACK IMMEDIATE; DROP DATABASE ZavaInventory;"
    Set-Location C:\
    Remove-Item C:\Lab4 -Recurse -Force
    ```

    ![A computer screen with text Description automatically generated](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%204/media/image69.png)

1. If you saved the estimate and don't need it, open **Saved estimates** in the Pricing Calculator and delete it. No Azure resources were created, so there is nothing to remove in Azure.


## Conclusion

In this lab you replaced a like-for-like guess with evidence. You measured a live workload, let the Azure SQL sizing engine turn it into SKUs for all three targets, and read the reasons behind each size. You saw that every bill is built from compute, license and storage meters, which explains why Azure Hybrid Benefit helps General Purpose but not new Hyperscale databases. You saw how the percentile and scaling factor move the estimate, and why the setting belongs next to every number you present. Finally, you stacked right-sizing, Azure Hybrid Benefit, reservations and serverless into a recommendation with a defensible monthly cost.
