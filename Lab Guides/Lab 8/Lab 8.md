# Lab 8: Configure Cross-Region Disaster Recovery with Failover Groups

**Duration:** about 40 minutes

## Lab overview

Lab 8 is based on **Module 4: Business continuity, disaster recovery, and cost optimization**. Lab 7 covered recovery inside one region. This lab covers the loss of a whole region: you replicate a database to a second region with a failover group, rehearse a failover and a failback, and cut the cost of the standby copy with the license-free standby replica and Azure Hybrid Benefit.

| Module 4 topic | Where you apply it |
|---|---|
| Recovery objectives (RTO/RPO) | Exercise 2 measures the failover time and checks that no data was lost |
| HA and DR strategies across Azure SQL destinations | Exercise 1 builds a failover group, the cross-region DR option for Azure SQL Database and Managed Instance |
| Backup, retention, and failover planning | Exercise 2 rehearses a planned failover; Exercise 3 fails back and reviews when to use a forced failover |
| Cost optimization and licensing strategies | Exercise 1 makes the secondary a license-free standby replica; Exercise 3 applies Azure Hybrid Benefit to the primary |
| Azure Hybrid Benefit, reservations, and Hyperscale economics | Exercise 3 compares the cost of the DR pair before and after both licensing benefits |

## Scenario

Fabrikam Retail’s OrdersDB takes orders from the web storefront around the clock. The business has set these disaster recovery targets for a regional outage:

- **RPO:** lose no more than a few seconds of orders.
- **RTO:** be taking orders again within 1 hour.
- **No application changes after a failover:** the storefront must keep using the same connection string.
- **Cost:** the standby copy exists only for disaster recovery, so Fabrikam doesn’t want to pay SQL Server licensing for it. Fabrikam also has SQL Server Standard core licenses with Software Assurance that it can bring to Azure.

You build and rehearse the DR configuration before go-live.

## Objectives

After completing this lab, you will be able to:

- Create a failover group that replicates a database to a second region.
- Designate the secondary database as a license-free standby replica.
- Use the failover group’s listener endpoint so that applications don’t change after a failover.
- Perform and time a planned failover and a failback, and confirm that no data was lost.
- Apply Azure Hybrid Benefit and explain how both licensing benefits change the cost of a DR pair.

## Lab values

Use these values throughout the lab. Replace **jd427** with your initials and three digits so that server names are unique.

- **Resource group:** ResourceGroup1
- **Primary region:** Central US
- **Secondary region:** East US 2
- **Primary server:** sql-fabrikam-pri-jd427 (Central US)
- **Secondary server:** sql-fabrikam-dr-jd427 (East US 2)
- **Database:** OrdersDB
- **Failover group:** fog-fabrikam-jd427
- **Admin login on both servers:** sqladminuser
- **Password:** Fabrikam!Lab2026

---

# Getting Started: Deploy the Primary Database and the Secondary Server

**Duration:** 7 minutes

## Task 1: Create OrdersDB on the primary server

1. Open [**portal.azure.com**](https://portal.azure.com/) and sign in with your lab account.

2. In the portal search bar, type **SQL databases**. Under **Services**, select **Azure SQL Database** (resource type **Microsoft.Sql/servers/databases**). Click **+ Create**. If a menu opens, select **SQL database**, not **SQL database (Free offer)**.

3. On the **Basics** tab, set **Resource group** to **ResourceGroup1** and **Database name** to **OrdersDB**. Under **Server**, click **Create new**.

4. In the **Create SQL Database Server** pane, set **Server name** to **sql-fabrikam-pri-jd427**, **Location** to **Central US**, **Authentication method** to **Use SQL authentication**, **Server admin login** to **sqladminuser**, and **Password** and **Confirm password** to **Fabrikam!Lab2026**. Click **OK**.

5. Set **Want to use SQL elastic pool?** to **No** and **Workload environment** to **Development**.

6. Under **Compute + storage**, click **Configure database**. Set **Service tier** to **General Purpose**, **Compute tier** to **Provisioned**, **Hardware configuration** to **Standard-series (Gen5)** and **vCores** to **2**. Set **Would you like to make this database zone redundant?** to **No**. Click **Apply**.

    > **Why provisioned, not serverless?** The license-free standby replica and Azure Hybrid Benefit, both used in this lab, apply only to the provisioned compute tier.

7. Under **Backup storage redundancy**, select **Locally-redundant backup storage**. The failover group, not the backups, protects this database against a regional outage.

8. Click **Next: Networking**. Set **Connectivity method** to **Public endpoint**, **Allow Azure services and resources to access this server** to **No**, and **Add current client IP address** to **Yes**.

9. Click the **Additional settings** tab. Set **Use existing data** to **Sample** and click **OK** in the confirmation dialog.

10. Click **Review + create**, then **Create**. Don’t wait; go straight to Task 2.

## Task 2: Create the secondary server

1. In the portal search bar, type **SQL servers**, select **SQL servers** under **Services**, and click **+ Create**.

2. On the **Basics** tab, set:

    1. **Resource group:** ResourceGroup1
    2. **Server name:** sql-fabrikam-dr-jd427
    3. **Location:** East US 2
    4. **Authentication method:** Use SQL authentication
    5. **Server admin login:** sqladminuser
    6. **Password** and **Confirm password:** Fabrikam!Lab2026

3. Click **Next: Networking**. Set **Allow Azure services and resources to access this server** to **No**, and set **Add current client IP address** to **Yes** if the option is shown.

4. Click **Review + create**, then **Create**.

    **Expected result:** both deployments complete within 5 minutes. The secondary server holds no database yet; the failover group creates it.

    > **Why add your IP to the secondary server too?** Server firewall rules aren’t copied by a failover group. If they exist only on the primary server, clients are locked out after a failover.

---

# Exercise 1: Create a Failover Group with a License-Free Standby Replica

**Duration:** 10 minutes

A failover group replicates one or more databases to a server in another region and gives them two listener endpoints that always point at the current primary and secondary. You add OrdersDB and mark its secondary as a standby replica, used only for DR.

## Task 1: Create the failover group

1. Open the primary server **sql-fabrikam-pri-jd427**. In the left menu, click **Data management > Failover groups**, then click **+ Add group**.

2. Set:

    1. **Failover group name:** fog-fabrikam-jd427
    2. **Server:** select **sql-fabrikam-dr-jd427** (East US 2)
    3. **Read/write failover policy:** Customer managed (if your portal shows **Manual**, select that)

    > **Why customer managed?** With this policy, Fabrikam decides when to fail over. Microsoft-managed failover waits for Microsoft to declare a regional outage, which can take longer than Fabrikam’s 1-hour RTO.

3. Under **Databases within the group**, click **Configure database** (or **Select databases**), select **OrdersDB**, and click **Select**.

4. If the pane offers a **Standby replica** option for OrdersDB, select it and select the check box confirming that you’ll use the secondary only for standby. If it isn’t offered here, you set it in Task 3.

5. Click **Create**.

    **Expected result:** the failover group is created within a minute, and initial seeding copies OrdersDB to East US 2. For this small database, seeding takes 2 to 5 minutes.

## Task 2: Check the replication and the listeners

1. On **Data management > Failover groups**, click **fog-fabrikam-jd427**.

2. Check the map and the server list.

    **Expected result:** **sql-fabrikam-pri-jd427** is listed as **Primary** in Central US and **sql-fabrikam-dr-jd427** as **Secondary** in East US 2. Wait until OrdersDB shows as replicated (for example **Seeding completed** or **Readable**) before you continue; click **Refresh** every minute.

3. Find the two listener endpoints on the same page:

    1. **Read/write listener endpoint:** fog-fabrikam-jd427.database.windows.net
    2. **Read-only listener endpoint:** fog-fabrikam-jd427.secondary.database.windows.net

    > **Why the listener matters:** the storefront connects to **fog-fabrikam-jd427.database.windows.net**, never to a server name. After a failover, Azure points the listener at the new primary, so the connection string doesn’t change.

## Task 3: Confirm the standby replica

1. Open the secondary server **sql-fabrikam-dr-jd427** and, in the left menu, click **Settings > SQL databases** (or **Data management > Databases**). Click **OrdersDB**.

2. On the **Overview** page, check **Replica type** in the **Essentials** section.

    **Expected result:** **Replica type** is **Standby**. Azure doesn’t charge SQL Server licensing for a standby replica’s vCores. You still pay for its compute and storage.

    > **If Replica type shows Geo instead,** the failover group pane didn’t set it. On the secondary OrdersDB, click **Data management > Replicas**, find OrdersDB, select the ellipsis (**…**) and choose the option to convert it to a standby replica. Confirm the standby check box and click **Yes**. Refresh the **Overview** page.

    > **What standby means:** a standby replica is for disaster recovery only. It can’t serve read queries, so the read-only listener isn’t available for reporting while the secondary is standby. If Fabrikam later wants to offload reads to East US 2, the secondary must become a normal geo-replica and pay for its license.

---

# Exercise 2: Rehearse a Planned Failover

**Duration:** 12 minutes

You save an order, fail the group over to East US 2, and confirm that the order is there and that the listener now points at East US 2.

## Task 1: Write an order and check where the listener points

1. Open **sql-fabrikam-pri-jd427 > OrdersDB** (the primary), click **Query editor**, and sign in with **sqladminuser** and **Fabrikam!Lab2026**. If you see a firewall error, click **Allowlist IP**, then **OK**.

2. Run:

    **T-SQL (on the primary OrdersDB)**

    ```sql
    CREATE TABLE dbo.DrDrill (Step nvarchar(50) PRIMARY KEY, StepTimeUtc datetime2(0) NOT NULL DEFAULT SYSUTCDATETIME());
    INSERT dbo.DrDrill (Step) VALUES (N'Order saved before failover');
    SELECT Step, StepTimeUtc FROM dbo.DrDrill;
    ```

    **Expected result:** one row.

3. On the portal toolbar, click the **Cloud Shell** icon (**>_**) and choose **Bash**. If this is your first time, select **No storage account required**, choose your subscription and click **Apply**.

4. Run this, replacing **jd427** with your suffix:

    **Bash**

    ```bash
    nslookup fog-fabrikam-jd427.database.windows.net
    ```

    **Expected result:** the **Aliases** section includes **sql-fabrikam-pri-jd427.database.windows.net**. The read/write listener points at the Central US server. Leave Cloud Shell open.

## Task 2: Fail over to East US 2

1. Return to **sql-fabrikam-pri-jd427 > Data management > Failover groups > fog-fabrikam-jd427**.

2. On the toolbar, click **Failover** (not **Forced failover**). Read the warning and click **Yes**. Note the time.

    > **Failover vs. forced failover:** a planned **Failover** first synchronizes the secondary, so no data is lost; you use it for drills and planned moves. **Forced failover** switches roles immediately without waiting for the secondary to catch up, so recent transactions can be lost. You use it only when the primary region is down and waiting would break the RTO.

3. Click **Refresh** until the roles swap.

    **Expected result:** within 1 to 2 minutes, **sql-fabrikam-dr-jd427** in East US 2 is **Primary** and **sql-fabrikam-pri-jd427** in Central US is **Secondary**.

## Task 3: Confirm no data loss and the new listener target

1. In Cloud Shell, run the same command again:

    **Bash**

    ```bash
    nslookup fog-fabrikam-jd427.database.windows.net
    ```

    **Expected result:** the **Aliases** section now includes **sql-fabrikam-dr-jd427.database.windows.net**. The listener moved to East US 2, with no change to the name applications use. If it still shows the old server, wait a minute and run it again; DNS records can take up to a minute to update.

2. Open the new primary: **sql-fabrikam-dr-jd427 > OrdersDB**. Click **Query editor** and sign in. If you see a firewall error, click **Allowlist IP**, then **OK**. Run:

    **T-SQL (on the new primary OrdersDB, East US 2)**

    ```sql
    INSERT dbo.DrDrill (Step) VALUES (N'Order saved after failover');
    SELECT Step, StepTimeUtc FROM dbo.DrDrill ORDER BY StepTimeUtc;
    ```

    **Expected result:** two rows. **Order saved before failover** made it across, so the planned failover lost no data (RPO zero), and the database in East US 2 accepts new orders.

    > **What you measured:** the time from clicking **Failover** to the roles swapping is your failover time, about 1 to 2 minutes, well inside the 1-hour RTO. In a real outage, add the time it takes Fabrikam to decide to fail over; that decision is usually the largest part of the RTO.

---

# Exercise 3: Fail Back and Optimize the Cost of the DR Pair

**Duration:** 10 minutes

After the drill, you return OrdersDB to Central US, where the storefront normally runs. Then you apply Fabrikam’s spare licenses to the primary.

## Task 1: Fail back to Central US

1. Open **sql-fabrikam-dr-jd427** (now the primary). In the left menu, click **Data management > Failover groups** and click **fog-fabrikam-jd427**.

2. On the toolbar, click **Failover** and click **Yes**.

3. Click **Refresh** until the roles swap back.

    **Expected result:** **sql-fabrikam-pri-jd427** in Central US is **Primary** again, and **sql-fabrikam-dr-jd427** is **Secondary**.

4. Open **sql-fabrikam-dr-jd427 > OrdersDB** (the secondary again) and check **Replica type** on the **Overview** page.

    **Expected result:** **Standby**, so the license-free benefit applies again now that this database is the secondary.

    > **If Replica type shows Geo,** convert it back to standby as in Exercise 1, Task 3. Always check the replica type after a failover or failback: a secondary that isn’t standby is charged the SQL license again.

## Task 2: Apply Azure Hybrid Benefit to the primary

1. Open **sql-fabrikam-pri-jd427 > OrdersDB** (the primary). In the left menu, click **Settings > Compute + storage**.

2. Under **Save money** (or **License type**), select **Yes** for **Already have a SQL Server License?** and select the check box confirming that you have SQL Server licenses with Software Assurance. Leave every other setting unchanged and click **Apply**.

    **Expected result:** a notification confirms the update, and the cost summary on the page drops, because the SQL license part of the price is removed.

    > **Licensing rule:** 2 vCores of General Purpose use 2 SQL Server Standard core licenses with Software Assurance, which Fabrikam has spare. The standby secondary needs no licenses at all.

## Task 3: Review the cost of the DR pair

1. On the same **Compute + storage** page, look at the **Cost summary**.

2. Open the secondary, **sql-fabrikam-dr-jd427 > OrdersDB**, click **Settings > Compute + storage**, and look at its **Cost summary**.

    **Expected result:** both databases show compute and storage costs, but neither includes a SQL license charge: the primary because of Azure Hybrid Benefit, the secondary because it is a standby replica.

    > **The cost of DR, in Module 4 terms:**
    >
    > - **Without either benefit,** a DR pair costs about twice a single database, because the secondary pays for compute, storage and the SQL license.
    > - **The license is about 40% of the vCore price,** so the standby replica removes about 40% of the secondary’s compute cost, and Azure Hybrid Benefit removes about the same from the primary.
    > - **Reserved capacity** would cut the compute price further for a 1-year or 3-year commitment. It is bought separately in **Reservations**, and isn’t possible in a lab subscription.
    > - **When the benefits don’t apply:** serverless databases can’t use either benefit, and a standby replica can’t serve reads. If you need the secondary for reporting, it becomes a normal geo-replica with a license charge.

---

# Conclusion

In this lab, you built cross-region disaster recovery for Fabrikam’s OrdersDB. You created a failover group from Central US to East US 2 and gave applications a listener endpoint that never changes. You rehearsed a planned failover and a failback, measured a failover time of minutes against a 1-hour RTO, and confirmed that no orders were lost. Finally, you cut the cost of the DR pair by making the secondary a license-free standby replica and applying Azure Hybrid Benefit to the primary, and reviewed when each benefit applies.

---

# Clean Up Resources

A failover group must be deleted before its servers, so work in this order. Keep **ResourceGroup1**.

1. **Delete the failover group:** open **sql-fabrikam-pri-jd427 > Data management > Failover groups**, click **fog-fabrikam-jd427**, click **Delete** on the toolbar, and click **Yes**.

    **Expected result:** the group is removed. Both copies of OrdersDB still exist, now as separate databases.

2. **Delete the secondary server:** open **sql-fabrikam-dr-jd427**, click **Delete** on the toolbar, type the server name and click **Delete**. This also deletes its copy of OrdersDB.

3. **Delete the primary server:** open **sql-fabrikam-pri-jd427**, click **Delete**, type the server name and click **Delete**.

4. In the portal search bar, type **All resources** and select **All resources**. Set the **Resource group** filter to **ResourceGroup1** and click **Refresh**.

    **Expected result:** none of the Lab 8 resources is listed, and **ResourceGroup1** still exists.
