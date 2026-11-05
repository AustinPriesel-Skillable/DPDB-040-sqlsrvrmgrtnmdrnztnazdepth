# **Lab 8: Configure Cross-Region Disaster Recovery with Failover Groups**

**Lab overview**

Fabrikam Retail’s OrdersDB takes orders from the web storefront around the clock. The business has set these disaster recovery targets for a regional outage:

- **RPO:** lose no more than a few seconds of orders.

- **RTO:** be taking orders again within 1 hour.

- **No application changes after a failover:** the storefront must keep using the same connection string.

- **Cost:** the standby copy exists only for disaster recovery, so Fabrikam doesn’t want to pay SQL Server licensing for it. Fabrikam also has SQL Server Standard core licenses with Software Assurance that it can bring to Azure.

You build and rehearse the DR configuration before go-live.

**Objectives**

After completing this lab, you will be able to:

- Create a failover group that replicates a database to a second region.

- Designate the secondary database as a license-free standby replica.

- Use the failover group’s listener endpoint so that applications don’t change after a failover.

- Perform and time a planned failover and a failback, and confirm that no data was lost.

- Apply Azure Hybrid Benefit and explain how both licensing benefits change the cost of a DR pair.

### **Getting Started: Deploy the Primary Database and the Secondary Server**

#### **Task 1: Create OrdersDB on the primary server**

1.  In the portal search bar, type +++SQL databases+++. Under **Services**, select **Azure SQL Database**.

    ![](./media/image1.png)

2.  Click **+ Create**. If a menu opens, select **SQL database**.

    ![A screenshot of a computer Description automatically generated](./media/image2.png)

3.  On the **Basics** tab, set **Resource group** to **ResourceGroup1** and **Database name** to +++OrdersDB+++. Under **Server**, click **Create new**.

    ![A screenshot of a computer Description automatically generated](./media/image3.png)

4.  In the **Create SQL Database Server** pane, set **Server name** to +++sql-fabrikam-pri-jd427+++, **Location** to **Central US**, **Authentication method** to **Use SQL authentication**, **Server admin login** to +++sqladminuser+++, and **Password** and **Confirm password** to +++Fabrikam!Lab2026+++. Click **OK**.

    ![](./media/image4.png)

5.  Set **Want to use SQL elastic pool?** to **No** and **Workload environment** to **Development**.

6.  Under **Compute + storage**, click **Configure database**.

    ![A screenshot of a computer Description automatically generated](./media/image5.png)

7.  Set **Service tier** to **General Purpose**, **Compute tier** to **Provisioned**, **Hardware configuration** to **Standard-series (Gen5)** and **vCores** to **2**. Set **Would you like to make this database zone redundant?** to **No**. Click **Apply**.

    **Why provisioned, not serverless?** The license-free standby replica and Azure Hybrid Benefit, both used in this lab, apply only to the provisioned compute tier.

    ![A screenshot of a computer Description automatically generated](./media/image6.png)

    ![A screenshot of a computer Description automatically generated](./media/image7.png)

8.  Under **Backup storage redundancy**, select **Locally-redundant backup storage**. The failover group, not the backups, protects this database against a regional outage.

    Click **Next: Networking**.

    ![](./media/image8.png)

9.  Set **Connectivity method** to **Public endpoint**, **Allow Azure services and resources to access this server** to **No**, and **Add current client IP address** to **Yes**.

    ![](./media/image9.png)

10. Click the **Additional settings** tab. Set **Use existing data** to **Sample** and click **OK** in the confirmation dialog.

    ![A screenshot of a computer Description automatically generated](./media/image10.png)

11. Click **Review + create**, then **Create**. Don’t wait; go straight to Task 2.

    ![A screenshot of a computer Description automatically generated](./media/image11.png)

    ![A screenshot of a computer Description automatically generated](./media/image12.png)

    ![A screenshot of a computer Description automatically generated](./media/image13.png)

#### **Task 2: Create the secondary server**

1.  In the portal search bar, type +++SQL servers+++, select **SQL servers** under **Services**, and click **+ Create**.

    ![A screenshot of a computer Description automatically generated](./media/image14.png)

    ![A screenshot of a computer Description automatically generated](./media/image15.png)

2.  On the **Basics** tab, set:

    - **Resource group:** ResourceGroup1

    - **Server name:** +++sql-fabrikam-dr-jd427+++

    - **Location:** East US

    - **Authentication method:** Use SQL authentication

    - **Server admin login:** +++sqladminuser+++

    - **Password** and **Confirm password:** +++Fabrikam!Lab2026+++

    ![](./media/image16.png)

    ![](./media/image17.png)

3.  Click **Next: Networking**. Set **Allow Azure services and resources to access this server** to **No**.

    ![A screenshot of a computer Description automatically generated](./media/image18.png)

4.  Click **Review + create**, then **Create**.

    ![A screenshot of a computer Description automatically generated](./media/image19.png)

    Once the deployment completes, navigate to **Home**.

    ![A screenshot of a computer Description automatically generated](./media/image20.png)

    **Expected result:** both deployments complete within 5 minutes. The secondary server holds no database yet; the failover group creates it.

    **Why add your IP to the secondary server too?** Server firewall rules aren’t copied by a failover group. If they exist only on the primary server, clients are locked out after a failover.

### **Exercise 1: Create a Failover Group with a License-Free Standby Replica**

A failover group replicates one or more databases to a server in another region and gives them two listener endpoints that always point at the current primary and secondary. You add OrdersDB and mark its secondary as a standby replica, used only for DR.

#### **Task 1: Create the failover group**

1.  Open the primary server **sql-fabrikam-pri-jd427**.

    ![A screenshot of a computer Description automatically generated](./media/image21.png)

2.  In the left menu, click **Data management > Failover groups**, then click **+ Add group**.

    ![A screenshot of a computer Description automatically generated](./media/image22.png)

3.  Set:

    - **Failover group name:** +++fog-fabrikam-jd427+++

    - **Server:** select **sql-fabrikam-dr-jd427** (East US)

    - A blue banner at the top of the pane says **Failover policy has been set to the recommended value of Customer managed**. There is no field to change; the portal sets it for you.

    **Why customer managed?** With this policy, Fabrikam decides when to fail over. Microsoft-managed failover waits for Microsoft to declare a regional outage, which can take longer than Fabrikam’s 1-hour RTO.

    ![A screenshot of a computer Description automatically generated](./media/image23.png)

4.  Under **Databases within the group**, click **Configure database**.

    ![A screenshot of a computer Description automatically generated](./media/image24.png)

5.  Select **OrdersDB**.

    If the pane offers a **Standby replica** option for OrdersDB, select it and select the check box confirming that you’ll use the secondary only for standby. If it isn’t offered here, you set it in Task 3.

    ![A screenshot of a computer Description automatically generated](./media/image25.png)

6.  Click **Create**.

    ![A screenshot of a computer Description automatically generated](./media/image26.png)

    ![](./media/image27.png)

    **Expected result:** the failover group is created within a minute, and initial seeding copies OrdersDB to East US. For this small database, seeding takes 2 to 5 minutes.

#### **Task 2: Check the replication and the listeners**

1.  Open the primary server **sql-fabrikam-pri-jd427** (if you left it, type +++SQL servers+++ in the portal search bar, select **SQL servers** under **Services**, and click the server). In the server’s left menu, click **Data management > Failover groups**, then click **fog-fabrikam-jd427**. If it isn’t listed yet, click **Refresh**.

    ![A screenshot of a computer Description automatically generated](./media/image28.png)

2.  Click the **Databases within group** tab.

    **Expected result:** under **Primary server: sql-fabrikam-pri-jd427** (Central US), **OrdersDB** shows **Online**. Under **Secondary server: sql-fabrikam-dr-jd427** (East US), **OrdersDB** shows **Non-readable**. Non-readable means seeding has finished and the secondary is a standby replica, which can’t serve reads. If the secondary still shows seeding, click **Refresh** every minute until it changes.

    ![](./media/image29.png)

3.  Click the **Configuration details** tab and find the two listener endpoints:

    - **Read/write listener endpoint:** fog-fabrikam-jd427.database.windows.net

    - **Read-only listener endpoint:** fog-fabrikam-jd427.secondary.database.windows.net

    **Why the listener matters:** the storefront connects to **fog-fabrikam-jd427.database.windows.net**, never to a server name. After a failover, Azure points the listener at the new primary, so the connection string doesn’t change.

    ![A screenshot of a computer Description automatically generated](./media/image30.png)

#### **Task 3: Confirm the standby replica**

1.  Open the secondary server **sql-fabrikam-dr-jd427** and, in the left menu, click **Settings > SQL databases**.

    Click **OrdersDB**.

    ![](./media/image31.png)

2.  On the **Overview** page, check **Replica type** in the **Essentials** section.

    **Expected result:** **Replica type** is **Standby**. Azure doesn’t charge SQL Server licensing for a standby replica’s vCores. You still pay for its compute and storage.

    **If Replica type shows Geo instead,** the failover group pane didn’t set it. On the secondary OrdersDB, click **Data management > Replicas**, find OrdersDB, select the ellipsis (**…**) and choose the option to convert it to a standby replica. Confirm the standby check box and click **Yes**. Refresh the **Overview** page.

    **What standby means:** a standby replica is for disaster recovery only. It can’t serve read queries, so the read-only listener isn’t available for reporting while the secondary is standby. If Fabrikam later wants to offload reads to East US, the secondary must become a normal geo-replica and pay for its license.

    ![A screenshot of a computer Description automatically generated](./media/image32.png)

### **Exercise 2: Rehearse a Planned Failover**

You save an order, fail the group over to East US, and confirm that the order is there and that the listener now points at East US.

#### **Task 1: Write an order and check where the listener points**

1.  Click the search bar at the top of the portal and type +++SQL databases+++.

2.  Under **Services**, click **Azure SQL Database**.

3.  The list shows two rows called **OrdersDB**. Look at the **Server** column and click the **OrdersDB** whose server is **sql-fabrikam-pri-jd427**, with **Location** **Central US**.

    ![A screenshot of a computer Description automatically generated](./media/image33.png)

4.  The OrdersDB **Overview** page opens. In **Essentials**, check that **Server name** shows **sql-fabrikam-pri-jd427.database.windows.net**.

    ![A screenshot of a computer Description automatically generated](./media/image34.png)

5.  In the left menu, click **Query editor**.

    ![A screenshot of a computer Description automatically generated](./media/image35.png)

6.  Sign in with +++sqladminuser+++ and +++Fabrikam!Lab2026+++ and click **Connect**.

7.  Run:

    **T-SQL (on the primary OrdersDB)**

    ```
    CREATE TABLE dbo.DrDrill (Step nvarchar(50) PRIMARY KEY, StepTimeUtc datetime2(0) NOT NULL DEFAULT SYSUTCDATETIME());
    INSERT dbo.DrDrill (Step) VALUES (N'Order saved before failover');
    SELECT Step, StepTimeUtc FROM dbo.DrDrill;
    ```

    **Expected result:** one row.

    ![A screenshot of a computer Description automatically generated](./media/image36.png)

8.  On the portal toolbar, click the **Cloud Shell** icon (**>_**) and choose **Bash**. If this is your first time, select **No storage account required**, choose your subscription and click **Apply**.

    ![](./media/image37.png)

    ![A screenshot of a computer Description automatically generated](./media/image38.png)

9.  Run this, replacing **jd427** with your suffix:

    **Bash**

    ```
    nslookup fog-fabrikam-jd427.database.windows.net
    ```

    **Expected result:** the **Aliases** section includes **sql-fabrikam-pri-jd427.database.windows.net**. The read/write listener points at the Central US server. Leave Cloud Shell open.

    ![A screenshot of a computer screen Description automatically generated](./media/image39.png)

    **Close** the Cloud Shell.

#### **Task 2: Fail over to East US**

1.  Return to **sql-fabrikam-pri-jd427 > Data management > Failover groups > fog-fabrikam-jd427**.

    ![](./media/image40.png)

2.  Stay on the **Configuration details** tab.

3.  Scroll right along the **sql-fabrikam-dr-jd427** (**East US**) row, the **Secondary** row.

4.  If there’s an ellipsis (**…**) at the end of that row, click it and select **Failover** (not **Forced failover**).

    ![](./media/image41.png)

5.  Read the warning and click **Yes**. Note the time.

    ![](./media/image42.png)

    **Failover vs. forced failover:** a planned **Failover** first synchronizes the secondary, so no data is lost; you use it for drills and planned moves. **Forced failover** switches roles immediately without waiting for the secondary to catch up, so recent transactions can be lost. You use it only when the primary region is down and waiting would break the RTO.

6.  Click **Refresh** until the roles swap.

    ![](./media/image43.png)

    **Expected result:** within 1 to 2 minutes, **sql-fabrikam-dr-jd427** in East US is **Primary** and **sql-fabrikam-pri-jd427** in Central US is **Secondary**.

#### **Task 3: Confirm no data loss and the new listener target**

1.  In Cloud Shell, run the same command again:

    **Bash**

    ```
    nslookup fog-fabrikam-jd427.database.windows.net
    ```

    **Expected result:** the **Aliases** section now includes **sql-fabrikam-dr-jd427.database.windows.net**. The listener moved to East US, with no change to the name applications use. If it still shows the old server, wait a minute and run it again; DNS records can take up to a minute to update.

    ![A screenshot of a computer Description automatically generated](./media/image44.png)

2.  In the portal search bar, type +++SQL databases+++.

3.  Under **Services**, click **Azure SQL Database**.

4.  In the list, click the **OrdersDB** whose **Server** column shows **sql-fabrikam-dr-jd427**, with **Location** **East US**.

    ![](./media/image45.png)

5.  In the left menu, click **Query editor**.

    ![A screenshot of a computer Description automatically generated](./media/image46.png)

6.  Sign in with +++sqladminuser+++ and +++Fabrikam!Lab2026+++. If you see a firewall error, click **Allowlist IP**, then **OK**. Run:

    **T-SQL (on the new primary OrdersDB, East US)**

    ```
    INSERT dbo.DrDrill (Step) VALUES (N'Order saved after failover');
    SELECT Step, StepTimeUtc FROM dbo.DrDrill ORDER BY StepTimeUtc;
    ```

    ![A screenshot of a computer Description automatically generated](./media/image47.png)

    **Expected result:** two rows. **Order saved before failover** made it across, so the planned failover lost no data (RPO zero), and the database in East US accepts new orders.

    **What you measured:** the time from clicking **Failover** to the roles swapping is your failover time, about 1 to 2 minutes, well inside the 1-hour RTO. In a real outage, add the time it takes Fabrikam to decide to fail over; that decision is usually the largest part of the RTO.

### **Exercise 3: Fail Back and Optimize the Cost of the DR Pair**

After the drill, you return OrdersDB to Central US, where the storefront normally runs. Then you apply Fabrikam’s spare licenses to the primary.

#### **Task 1: Fail back to Central US**

1.  Open **sql-fabrikam-dr-jd427** (now the primary). In the left menu, click **Data management > Failover groups** and click **fog-fabrikam-jd427**.

    ![A screenshot of a computer Description automatically generated](./media/image48.png)

2.  On the toolbar, click **Failover** and click **Yes**.

    ![A screenshot of a computer Description automatically generated](./media/image49.png)

    ![A screenshot of a computer Description automatically generated](./media/image50.png)

3.  Click **Refresh** until the roles swap back.

    **Expected result:** **sql-fabrikam-pri-jd427** in Central US is **Primary** again, and **sql-fabrikam-dr-jd427** is **Secondary**.

    ![A screenshot of a computer Description automatically generated](./media/image51.png)

4.  Open **sql-fabrikam-dr-jd427 > OrdersDB** (the secondary again) and check **Replica type** on the **Overview** page.

    **Expected result:** **Standby**, so the license-free benefit applies again now that this database is the secondary.

    ![A screenshot of a computer Description automatically generated](./media/image52.png)

#### **Task 2: Apply Azure Hybrid Benefit to the primary**

1.  Open **sql-fabrikam-pri-jd427 > OrdersDB** (the primary). In the left menu, click **Settings > Compute + storage**.

    ![A screenshot of a computer Description automatically generated](./media/image53.png)

2.  Under **Save money** (or **License type**), select **Yes** for **Already have a SQL Server License?** and select the check box confirming that you have SQL Server licenses with Software Assurance. Leave every other setting unchanged and click **Apply**.

    ![A screenshot of a computer Description automatically generated](./media/image54.png)

    **Expected result:** a notification confirms the update, and the cost summary on the page drops, because the SQL license part of the price is removed.

    **Licensing rule:** 2 vCores of General Purpose use 2 SQL Server Standard core licenses with Software Assurance, which Fabrikam has spare. The standby secondary needs no licenses at all.

#### **Task 3: Review the cost of the DR pair**

1.  On the same **Compute + storage** page, look at the **Cost summary** and note the **Estimated cost/month** and the **Azure Hybrid Benefit discount** line.

    ![A screenshot of a computer Description automatically generated](./media/image55.png)

2.  Open the secondary, **sql-fabrikam-dr-jd427 > OrdersDB**, click **Settings > Compute + storage**, and look at its **Cost summary**.

    ![A screenshot of a computer Description automatically generated](./media/image56.png)

    **Expected result:** both databases show compute and storage costs, but neither includes a SQL license charge: the primary because of Azure Hybrid Benefit, the secondary because it is a standby replica.

### **Conclusion**

In this lab, you built cross-region disaster recovery for Fabrikam’s OrdersDB. You created a failover group from Central US to East US and gave applications a listener endpoint that never changes. You rehearsed a planned failover and a failback, measured a failover time of minutes against a 1-hour RTO, and confirmed that no orders were lost. Finally, you cut the cost of the DR pair by making the secondary a license-free standby replica and applying Azure Hybrid Benefit to the primary, and reviewed when each benefit applies.

### **Clean Up Resources**

A failover group must be deleted before its servers, so work in this order. Keep **ResourceGroup1**.

1.  **Delete the failover group:** open **sql-fabrikam-pri-jd427 > Data management > Failover groups**, click **fog-fabrikam-jd427**, click **Delete** on the toolbar, and click **Yes**.

    **Expected result:** the group is removed. Both copies of OrdersDB still exist, now as separate databases.

2.  **Delete the secondary server:** open **sql-fabrikam-dr-jd427**, click **Delete** on the toolbar, type +++sql-fabrikam-dr-jd427+++ and click **Delete**. This also deletes its copy of OrdersDB.

3.  **Delete the primary server:** open **sql-fabrikam-pri-jd427**, click **Delete**, type +++sql-fabrikam-pri-jd427+++ and click **Delete**.

4.  In the portal search bar, type +++All resources+++ and select **All resources**. Set the **Resource group** filter to **ResourceGroup1** and click **Refresh**.

    **Expected result:** none of the Lab 8 resources is listed, and **ResourceGroup1** still exists.
