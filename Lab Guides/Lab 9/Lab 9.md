# **Lab 9: Secure Azure SQL workloads**

**Introduction**

You’ve just joined Zava Cycles as the database engineer. Zava sells bikes, components and riding gear, and its order system runs on Azure SQL Database. The company plans to launch an AI shopping assistant on the same data, but a pre-launch security review stopped the project: developers share one SQL administrator password, nothing records who reads or changes customer data, and nobody is alerted to attacks or problems.

Your job is to deploy the database the right way and close those gaps. Microsoft Entra ID will replace shared passwords, every sign-in and query will be audited, Microsoft Defender for SQL will watch for attacks and misconfigurations, and monitoring will tell you when the database is unhealthy.

This is the first of three connected labs. Lab 10 adds governance and data protection to the database you secure here, and Lab 11 adds AI capabilities. Do not delete your resources at the end of this lab.

**Objectives**

- Deploy Azure SQL Database with network access limited to your own IP address.

- Configure a Microsoft Entra administrator, grant access through an Entra group, and enforce Entra-only authentication.

- Send server audit events to Log Analytics and trace queries, schema changes and failed sign-ins to a specific identity.

- Monitor the database with metrics, an alert rule, Query Store and automatic tuning.

- Enable Microsoft Defender for SQL, remediate a vulnerability, and investigate a security alert.

**Prerequisites**

- An Azure subscription where you have **Contributor** rights on your resource group.

- Permission to create security groups in Microsoft Entra ID. If your lab account does not have this permission, your instructor will create the **SQL-Analysts** group for you (Exercise 1, Task 2).

- A query tool: SQL Server Management Studio (SSMS) 21 or later (SSMS 22 recommended) on your lab machine, **or** the Azure portal **Query editor**, which needs no installation. Exercise 1, Task 3 covers both.

**Instructor note:** Log Analytics takes 5 to 10 minutes to show audit data. Learners enable auditing early in Exercise 2, complete Exercise 3, then return to Exercise 2, Task 4. The guide tells them when.

### **Getting started: Set up your environment**

**Duration:** 8 minutes

**Portal tip:** Resource menus in the portal are grouped and collapsed. When a step says **Security > Networking**, click the **>** arrow next to **Security** to expand it, then click **Networking**. You can also type the page name in the menu’s **Search** box.

#### **Task 1: Create the Azure SQL Database**

1.  Open [**portal.azure.com**](https://portal.azure.com/).

2.  Sign in with your lab account.

3.  On the portal **Home** page, under **Azure services**, click **SQL databases**.

    - If the tile is not shown, type +++Azure SQL Database+++ in the search bar at the top, and then select **Azure SQL Database** under **Services**.

    - Do not select **Azure SQL** (a hub page with no **Create** button), **Azure SQL Database Hyperscale** (it pre-selects the Hyperscale tier), or **SQL Server databases** (Azure Arc).

    ![](./media/image2.png)

4.  On the **SQL databases** page, click **+ Create** on the toolbar.

    ![](./media/image3.png)

5.  From the drop-down list, select **SQL database**.

    - Do not select **SQL database (Free offer)**; it opens a different wizard with free-tier limits.

    ![](./media/image4.png)

6.  On the **Basics** tab, set the following values:

    - **Resource group:** Select your lab resource group.

    - **Database name:** Enter +++sqldb-zava+++.

7.  Under **Server**, click **Create new**.

    ![](./media/image5.png)

8.  In the **Create SQL Database Server** pane, set the following values:

    - **Server name:** +++sql-zava-jd427+++

    - **Location:** East US (or the region your instructor gives you)

    - **Authentication method:** Use SQL authentication

    - **Server admin login:** +++sqladminuser+++

    - **Password:** +++P@ssword12345+++

    - **Confirm password:** +++P@ssword12345+++

    ![](./media/image6.png)

9.  Click **OK**.

10. Set **Want to use SQL elastic pool?** to **No**.

11. Set **Workload environment** to **Development**.

12. Under **Compute + storage**, click **Configure database**.

    ![](./media/image7.png)

13. Set **Service tier** to **General Purpose**.

14. Set **Compute tier** to **Serverless**.

15. Set **Max vCores** to **2**.

16. Set **Auto-pause delay** to **1 hour**.

17. Click **Apply**.

    ![](./media/image8.png)

18. Set **Backup storage redundancy** to **Locally-redundant backup storage**.

    ![](./media/image9.png)

19. Click the **Networking** tab.

    ![](./media/image11.png)

20. Set the following values:

    - **Connectivity method:** Public endpoint

    - **Allow Azure services and resources to access this server:** No

    - **Add current client IP address:** Yes

    ![](./media/image12.png)

21. Click the **Security** tab.

    ![](./media/image13.png)

22. Set **Enable Microsoft Defender for SQL** to **Not now**. You enable it in Exercise 3.

23. Click the **Additional settings** tab.

     ![](./media/image14.png)

24. Under **Data source**, set **Use existing data** to **Sample**. This loads the AdventureWorksLT sample database.

25. In the confirmation dialog, click **OK**.

    ![](./media/image15.png)

26. Click **Review + create**.

    ![](./media/image16.png)

27. On the **Review + create** tab, click **Create**.

    ![](./media/image17.png)

28. Wait for the deployment to complete. This takes 2 to 3 minutes.

    ![](./media/image18.png)

#### **Task 2: Confirm network security**

Azure SQL blocks all public connections until a firewall rule allows them. In this task, you confirm that only your IP address is allowed.

1.  Click **Go to resource**.

    ![](./media/image19.png)

2.  On the database **Overview** page, click the **Server name** link to open the logical server.

    ![](./media/image20.png)

3.  In the left menu, click **Security > Networking**.

    ![](./media/image21.png)

4.  On the **Public access** tab, confirm that **Public network access** is set to **Selected networks**.

5.  Under **Firewall rules**, confirm that one rule named **ClientIPAddress_<date>** exists.

    ![](./media/image22.png)

6.  In the **Rule name** box, rename the rule to +++AllowMyPC+++.

    ![](./media/image23.png)

7.  Confirm that **Allow Azure services and resources to access this server** is cleared. Leaving it selected would let resources in any Azure tenant reach the server’s login prompt.

    ![](./media/image24.png)

8.  Click **Save**.

    ![](./media/image25.png)

9.  Click the **Connectivity** tab.

10. Confirm that **Minimum TLS version** is **1.2**.

    ![](./media/image26.png)

11. Under **Outbound networking**, confirm that **Restrict outbound networking** shows **Restrictions disabled**. Lab 11 calls Azure OpenAI from the database and needs this outbound access.

    ![](./media/image27.png)
#### **Task 3: Turn on the server’s managed identity**

Lab 11 uses this identity to call Azure OpenAI without storing an API key in the database. Turning it on now means Lab 11 can start straight away.

1.  In the logical server’s left menu, click **Security > Identity**.

    ![](./media/image25.png)

2.  Under **System assigned managed identity**, set **Status** to **On**.

    ![](./media/image25.png)

3.  Click **Save**.

    ![](./media/image25.png)

4.  Confirm that **Status** is **On** and that **Save** is greyed out, which means the change is saved.

    ![](./media/image26.png)

### **Exercise 1: Configure Microsoft Entra authentication**

**Duration:** 14 minutes

In this exercise, you make your account the Microsoft Entra administrator, grant an Entra group read access, and then turn off SQL authentication entirely.

    ![Set Entra admin, create contained users, enforce Entra-only; Entra MFA sign-in allowed and sqladminuser rejected](./media/image27.png)

#### **Task 1: Set the Microsoft Entra administrator**

1.  On the logical server’s left menu, click **Settings > Microsoft Entra ID**.

    ![](./media/image32.png)

2.  Click **Set admin**.

    ![](./media/image33.png)

3.  In the search box, search for your lab account.

4.  Select your lab account.

5.  Click **Select**.

    ![](./media/image34.png)

6.  Leave **Support only Microsoft Entra authentication for this server** cleared for now.

7.  Click **Save**.

    ![](./media/image35.png)

#### **Task 2: Create the SQL-Analysts group**

You will grant database access to an Entra group rather than to individual people. Check whether the group exists, and create it if not.

1.  In the portal search bar, type +++Microsoft Entra ID+++.

    ![](./media/image36.png)

2.  Under **Services**, select **Microsoft Entra ID**.

     ![](./media/image37.png)

3.  In the left menu, click **Manage > Groups**.

    ![](./media/image38.png)

4.  Search for +++SQL-Analysts+++.

    ![](./media/image38.png)

5.  If the group exists, note its exact name and go to Task 3. Otherwise, continue with the next step.

6.  Click **+ New group**.

7.  Set the following values:

    - **Group type:** Security

    - **Group name:** +++SQL-Analysts+++

    - **Membership type:** Assigned

    ![](./media/image40.png)

8.  Under **Members**, click **No members selected**.

9.  Search for and select your lab account.

10. Click **Select**.

    ![](./media/image41.png)

11. Click **Create**.

    ![](./media/image42.png)

12. Wait about one minute for the group to be created before you continue.

    **If you cannot create groups:** Some lab tenants do not allow learners to create groups (**+ New group** is greyed out or fails). Ask your instructor to create the group or to give you another lab user’s email address. In Task 4, replace **[SQL-Analysts]** with **[\<that user’s email\>]**. The user then appears as **EXTERNAL_USER** instead of **EXTERNAL_GROUP**.

    ![](./media/image43.png)

#### **Task 3: Connect to the database**

Use **Option A** if SSMS is installed on your lab machine; otherwise, use **Option B**. Later steps say “in your query window”; this means the tool you chose.

##### **Option A: SQL Server Management Studio (SSMS)** (Optional)

1.  On your lab machine, open **SQL Server Management Studio**.

2.  In the **Connect to Server** dialog, set **Server name** to +++sql-zava-jd427.database.windows.net+++.
   
3.  Set **Authentication** to **Microsoft Entra MFA**.
  
4.  In **User name**, enter your lab account.
    
5.  Set **Database name** to +++sqldb-zava+++.

6.  Click **Connect**.

7.  Complete the Microsoft Entra sign-in prompt with your lab account.

8.  In **Object Explorer**, expand **Databases**.

9.  Right-click **sqldb-zava** and select **New Query**.

##### **Option B: Azure portal Query editor (no installation)**

1.  Open the logical server **sql-zava-jd427**. Click **sqldb-zava**. In the left menu, expand **Settings** and click **SQL databases**.

   
    ![](./media/image43.png)

    
    ![](./media/image44.png)
    

2.  In the database’s left menu, click **Query editor**.

    ![](./media/image45.png)

    
5.  On the **Microsoft Entra authentication** tab, click **Connect as <your account>**.

    ![](./media/image46.png)

6.  Click **New query**.

    ![](./media/image47.png)


#### **Task 4: Create contained users for Entra identities**

1.  In your query window, run the following code:

    **T-SQL**

    ```
    -- Members of the Entra group inherit access; no per-person users needed
    CREATE USER [SQL-Analysts] FROM EXTERNAL PROVIDER;
    ALTER ROLE db_datareader ADD MEMBER [SQL-Analysts];

    -- Verify Entra-based principals
    SELECT name, type_desc, authentication_type_desc
    FROM sys.database_principals
    WHERE type IN ('E', 'X'); -- E = external user, X = external group
    ```

    **Expected result:** one row for **SQL-Analysts** with type **EXTERNAL_GROUP** and authentication type **EXTERNAL**.

    If you see **Principal 'SQL-Analysts' could not be found**, the group does not exist or the name differs. Return to Task 2, then run the whole script again; nothing after line 1 runs when line 1 fails.

    ![](./media/image49.png)

#### **Task 5: Enforce Entra-only authentication and validate**

1.  In the portal, return to the server’s **Settings > Microsoft Entra ID** page.

2.  Select **Support only Microsoft Entra authentication for this server**.

3.  In the **Enable Microsoft Entra-only authentication** dialog, click **Yes**.

    ![](./media/image50.png)

4.  Click **Save**.

    ![](./media/image51.png)

5.  In your query window, run the following check:

    **T-SQL**

    ```
    SELECT SUSER_SNAME() AS login_name, USER_NAME() AS database_user, DB_NAME() AS database_name,
           (SELECT COUNT(*) FROM SalesLT.Customer) AS customer_rows;
    ```

    **Expected result:** **login_name** is your lab account, **database_user** is **dbo** (you are the Entra admin), **database_name** is **sqldb-zava**, and **customer_rows** is 847.

     ![](./media/image52.png)   

6.  If you use the **Query editor**, test that SQL authentication is now refused:

    1.  In the logical server’s left menu, expand **Settings** and click **SQL databases**.

    2.  Click **sqldb-zava**.

        ![](./media/image53.png) 

    3.  In the database’s left menu, click **Query editor**.

         ![](./media/image54.png) 

    4.  On the toolbar, click **Sign out**.

    5.  Select the **SQL authentication** tab.

         ![](./media/image55.png) 

    6.  In **Username**, enter +++sqladminuser+++. In **Password**, enter +++P@ssword12345+++.

         ![](./media/image55.png) 

    7.  Click **Connect**.

         ![](./media/image56.png)   

    8.  Confirm that the conection is refused with a message that the server supports only Microsoft Entra authentication.

        ![](./media/image57.png) 

### **Exercise 2: Enable SQL auditing and monitoring**

#### **Task 1: Create a Log Analytics workspace**

1.  In the portal search bar, type +++Log Analytics workspaces+++.

2.  Under **Services**, select **Log Analytics workspaces**.

    ![](./media/image61.png) 

3.  Click **+ Create**.

    ![](./media/image62.png) 

4.  On the **Basics** tab, set the following values:

    - **Resource group:** Select your lab resource group.

    - **Name:** Enter +++law-zava-jd427+++.

    - **Region:** Select the same region as your server.

5.  Click **Review + Create**.

    ![](./media/image63.png) 

6.  Click **Create**.

    ![](./media/image64.png) 


7.  Wait for the deployment to complete. This takes less than a minute.

    ![](./media/image65.png) 


#### **Task 2: Enable server auditing and diagnostics**

1.  Open the logical server **sql-zava-jd427**.

    ![](./media/image66.png)

2.  In the left menu, click **Security > Auditing**.

    ![](./media/image67.png)

3.  Turn on **Enable Azure SQL Auditing**.

    ![](./media/image68.png)

4.  Under **Audit log destination**, select **Log Analytics**.

5.  Select the **law-zava-jd427** workspace.

    ![](./media/image69.png)

6.  Turn on **Enable Auditing of Microsoft support operations**.

7.  Send these audit logs to the same Log Analytics workspace, **law-zava-jd427**. Click **Save**.

    ![](./media/image70.png)

8.  Open the database **sqldb-zava**.

    ![](./media/image71.png)


9. In the left menu, click **Monitoring > Diagnostic settings**. Click **+ Add diagnostic setting**.

    ![](./media/image72.png)

10. In **Diagnostic setting name**, enter +++diag-to-law+++.

11. Under **Categories**, select the following checkboxes:

    - **SQL Insights**

    - **Errors**

    - **Timeouts**

    - **Deadlocks**

    ![](./media/image73.png)

12. Under **Metrics**, select **Basic**.

    ![](./media/image74.png)

13. Under **Destination details**, select **Send to Log Analytics workspace**.

14. Select the **law-zava-jd427** workspace.

15. Click **Save**.

    ![](./media/image75.png)
    
    **Note:** Server-level auditing covers every database on the server, including future ones. Do not also enable database-level auditing; it duplicates records.

#### **Task 3: Generate activity**

1.  Return to your query window.

2.  Run the following code. You must be signed in with Microsoft Entra authentication.

    **T-SQL**

    ```
    SELECT TOP (10) FirstName, LastName, EmailAddress FROM SalesLT.Customer;
    SELECT COUNT(*) AS orders FROM SalesLT.SalesOrderHeader;
    CREATE TABLE dbo.AuditDemo (Id int PRIMARY KEY);
    DROP TABLE dbo.AuditDemo;
    SELECT * FROM dbo.TableThatDoesNotExist; -- deliberate error 208
    ```
    ![](./media/image75.png)

3. Sign back in with **Microsoft Entra authentication**.

    ![](./media/image46.png)

    **Query editor users:** a sign-in rejected by the Query editor may not be recorded as a database authentication failure, so the failed sign-in query in Task 4 may return no rows. That is expected; note it in your evidence and continue.

    **Now go to Exercise 3.** Return to Task 4 after Exercise 3, when the logs have had time to arrive.

#### **Task 4: Review audit logs (after Exercise 3)**

1.  On the portal **Home** page, click **Log Analytics workspaces**.

    ![](./media/image78.png)

2.  Click **law-zava-jd427**.

    ![](./media/image79.png)

3.  In the left menu, select **Logs**.

    ![](./media/image80.png)

4.  If the **Welcome to Log Analytics** window opens, close it.

    ![](./media/image81.png)

5.  If the **Queries hub** window opens, close it.

    ![](./media/image82.png)

6.  Turn off the **Agent** toggle.

    ![](./media/image83.png)

7.  At the top right of the query pane, open the **Simple mode** drop-down list.

    ![](./media/image84.png)

8.  Select **KQL mode**.

    ![](./media/image85.png)

9.  Paste the following query into the query pane:

    **KQL: recent audit events**

    ```
    AzureDiagnostics
    | where Category == "SQLSecurityAuditEvents"
    | where TimeGenerated > ago(24h)
    | project TimeGenerated, action_name_s, server_principal_name_s,
              client_ip_s, succeeded_s, statement_s
    | order by TimeGenerated desc
    ```
    
10. Click **Run**.

11. In the results, find your **SELECT**, **CREATE TABLE** and **DROP TABLE** statements with your account name.

    ![](./media/image86.png)

12. Replace the query with the following query:

    **KQL: failed sign-ins**

    ```
    AzureDiagnostics
    | where Category == "SQLSecurityAuditEvents"
    | where action_name_s == "DATABASE AUTHENTICATION FAILED"
    | summarize Failures = count() by client_ip_s, bin(TimeGenerated, 15m)
    ```

13. Click **Run**.

14. Confirm that the failed sign-ins appear with your client IP address (SSMS users).

    - Query editor users may see no rows; see the note in Task 3.

    - If neither query returns rows, wait 5 minutes and run both queries again. Audit data takes 5 to 10 minutes to arrive.

    ![](./media/image87.png)    

#### **Task 5: Monitor and manage the operational environment**

Auditing answers “who did what”. Operational monitoring answers “is the database healthy, and is anything getting worse”.

1.  Open the database **sqldb-zava**.

    ![](./media/image88.png)

2.  In the left menu, click **Monitoring > Metrics**.

    ![](./media/image89.png)

3.  Set **Metric** to **App CPU percentage**.

    - This is the CPU metric for serverless databases. On a provisioned database, select **CPU percentage** instead.

4.  Set **Aggregation** to **Avg**.

    - In the time range picker at the top right, select **Last hour**.
 
        ![](./media/image90.png)

5.  Click **+ Add metric**.

    - Set **Metric** to **Data IO percentage**.

    - Set **Aggregation** to **Avg**.

    ![](./media/image91.png)

6.  Click **+ Add metric** again.

    - Set **Metric** to **Failed Connections : User Errors**.

    - Set **Aggregation** to **Sum**.

    ![](./media/image92.png)

7. In the left menu, click **Monitoring > Alerts**.

    - Click **+ Create > Alert rule**.

        ![](./media/image93.png)

    - On the **Condition** tab, select the **Failed Connections : User Errors** signal.

         ![](./media/image94.png)

8. Under **Alert logic**, set the following values:

    - **Threshold type:** Static

    - **Aggregation type:** Total

    - **Value is:** Greater than

    - **Unit:** Count

    - **Threshold value:** +++5+++
    
7. Under **When to evaluate**, set the following values:

    - **Check every:** 1 minute

    - **Lookback period:** 5 minutes

    - Review the estimated cost in the **Preview** pane (about USD 0.10 per month).

     ![](./media/image95.png)

8. Skip the **Actions** tab. Add an action group with your email address only if your instructor asks you to.

9. Click the **Details** tab.

    - Set **Severity** to **2 - Warning**.

    - Set **Alert rule name** to +++alert-failed-connections+++.

10. Click **Review + create**.

     ![](./media/image96.png)

11. Click **Create**.

     ![](./media/image97.png)

     ![](./media/image98.png)

12. In the database’s left menu, click **Intelligent performance > Query performance insight**.

13. Review the top queries by CPU.

    ![](./media/image100.png)

14. If the page shows “There is no performance data available”, wait a few minutes and click **Refresh**. The chart fills once Query Store has collected data.

     ![](./media/image99.png)

15. Click **Intelligent performance > Automatic tuning**.

    ![](./media/image101.png)

    - For **FORCE PLAN**, set **Desired state** to **ON**.

    ![](./media/image102.png)


18. Click **Apply**.

    ![](./media/image103.png)

19. In the **Do you want to continue?** dialog, click **OK**.

20. Confirm that **Current state** for **FORCE PLAN** is **ON**. It shows **Inherited from server** or **Forced by user**; both are correct.

    ![](./media/image104.png)

21. Return to your query window.

22. Run the following code to confirm that Query Store and automatic tuning are active:

    **T-SQL**

    ```
    SELECT actual_state_desc, query_capture_mode_desc,
           current_storage_size_mb, max_storage_size_mb
    FROM sys.database_query_store_options;

    SELECT name, desired_state_desc, actual_state_desc
    FROM sys.database_automatic_tuning_options;
    ```

    **Expected result:** Query Store **actual_state_desc** is **READ_WRITE**, and **FORCE_LAST_GOOD_PLAN** **actual_state_desc** is **ON**. In the Query editor, run each **SELECT** statement separately if only one result set is shown.

    ![](./media/image105.png)

### **Exercise 3: Configure Microsoft Defender for SQL**

#### **Task 1: Enable Defender for SQL**

1.  Open the database **sqldb-zava**.

    ![](./media/image106.png)

2.  In the left menu, click **Security > Microsoft Defender for Cloud**.

3.  Click **Enable**.

    ![](./media/image107.png)

4.  Wait 1 to 2 minutes, then refresh the page.

5.  Confirm that the page shows **Microsoft Defender for SQL: On** and **Enabled at the server-level**.

    ![](./media/image108.png)

6.  Under **Microsoft Defender for SQL**, click **Settings**.

    ![](./media/image109.png)

7.  Confirm that vulnerability assessment uses the **Express configuration** (the default; it needs no storage account).

    - If vulnerability assessment is off, or shows the classic configuration with no storage account, switch it to the express configuration and click **Save**. Otherwise, scans never produce results.

    ![](./media/image110.png)

    **Note:** Email notifications for security alerts are set for the whole subscription in **Defender for Cloud > Environment settings >** your subscription **> Email notifications**. You do not need to change them for this lab.

    **What you see at first:** Immediately after you enable Defender, the page shows **Recommendations (0)**, **Security Alerts (0)** and “No SQL vulnerability assessment scan results available”. This is normal: there are no results until you run a scan in Task 2. A scan takes 1 to 5 minutes, and its results appear under **View database vulnerability summary**. You do not use the **Recommendations** or **Security Alerts** counters on this page in this lab, so ignore them if they still show 0.

#### **Task 2: Run a vulnerability assessment**

1.  In your query window, connected to **sqldb-zava** (not master), run the following code to introduce a deliberate weakness and confirm that it took effect:

    **T-SQL**

    ```
    GRANT SELECT ON SalesLT.Customer TO public; -- every user inherits this

    -- Confirm the grant exists; this also wakes a paused serverless database
    SELECT DB_NAME() AS database_name,
           OBJECT_SCHEMA_NAME(major_id) AS schema_name,
           OBJECT_NAME(major_id) AS object_name,
           permission_name, state_desc
    FROM sys.database_permissions
    WHERE grantee_principal_id = DATABASE_PRINCIPAL_ID('public')
      AND class = 1; -- object-level permissions
    ```

    **Expected result:** a row with **database_name** **sqldb-zava**, **SalesLT**, **Customer**, **SELECT** and **GRANT**. If **database_name** is **master**, or there is no row, select **sqldb-zava** in your query tool and run the script again.

    ![](./media/image111.png)

2.  Return to the database’s **Security > Microsoft Defender for Cloud** page.

3.  Under **Vulnerabilities on this database**, click **Scan now**.

    ![](./media/image121.png)

4.  Wait for the scan to finish. This usually takes 1 to 5 minutes.

    ![](./media/image112.png)

5.  Click **View database vulnerability summary**.

    ![](./media/image113.png)

6.  On the **Vulnerability Assessment** page, confirm that the **Last scan time** is later than the time you ran the **GRANT**.

    - If it is not, click **Scan history**.

    - If the latest scan is still in progress, wait, and then click **Refresh**.

    ![](./media/image114.png)

7.  On the **Findings** tab, clear any severity or status filter.

8.  In the search box, enter +++PUBLIC+++.

9.  Open the finding **Minimal set of principals should be members of fixed low impact database roles in SQL databases** (rule VA2109).

10. Review its description, the query results (**SalesLT.Customer**, **SELECT**) and the remediation script.

11. In your query window, run the following code to remediate the finding:

    **T-SQL**

    ```
    GRANT SELECT ON SalesLT.Customer TO public;   -- every user inherits this

    -- Confirm the grant exists; this also wakes a paused serverless database
    SELECT DB_NAME() AS database_name,
       OBJECT_SCHEMA_NAME(major_id) AS schema_name,
       OBJECT_NAME(major_id) AS object_name,
       permission_name, state_desc
    FROM sys.database_permissions
    WHERE grantee_principal_id = DATABASE_PRINCIPAL_ID('public')
    AND class = 1;   -- object-level permissions

    ```
    ![](./media/image111.png)

12. Return to the **Vulnerability Assessment** page.

    ![](./media/image112.png)

13. Open a finding that lists configuration you intend to keep: **VA2109 - Minimal set of principals should be members of fixed low impact database roles in SQL databases**. It lists **SQL-Analysts** in **db_datareader**, which you granted on purpose in Exercise 1.

    ![](./media/image113.png)

14. Click **Add all results as baseline**.

    ![](./media/image115.png)

15. In the **Set baseline** dialog, click **Yes**.

    ![](./media/image116.png)

16. Confirm that the result status shows **In Baseline**.

    ![](./media/image117.png)

17. Click **Scan** again.

18. Wait for the scan to complete.

19. Click the **Passed** tab.

    ![](./media/image118.png)

20. Confirm that the PUBLIC finding (VA1054) now appears under **Passed** checks.

21. Confirm that the baselined finding (VA2109) shows **Passed per custom baseline**.

    **Where is the firewall rule finding?** Server-level firewall rules are stored in the master database, so the check that flags your **AllowMyPC** rule (VA2065) appears in the server’s (master) results, not in **sqldb-zava**. You can view it on the logical server’s **Security > Microsoft Defender for Cloud** page. Baselining it there is optional.

     ![](./media/image123.png)

##### **If the vulnerability count still shows 0**

    - **No scan has completed yet.** Click **Scan now**, wait for the notification (1 to 5 minutes), and then open **View database vulnerability summary**. Ignore the **Recommendations (0)** counter; it is not used in this lab.

    - **The scan ran before the GRANT.** Run the confirmation query again, and then scan again.

    - **The GRANT ran in master.** In the Query editor, the connection is always to the database you opened it from; in SSMS, check the database drop-down on the toolbar.

    - **Defender is still provisioning, or vulnerability assessment is not configured.** On the Defender for Cloud page, confirm **On** and **Enabled at the server-level**, and then check **Settings** for the express configuration (Task 1, step 7). If you enabled Defender less than 2 minutes ago, wait, refresh the page, and scan again.

    - **A filter hides the finding.** Clear the search box and any severity filter, and look in both the failed and passed lists.

     ![](./media/image119.png)

### **Review your lab resources**

1.  In the portal, open **Resource Manager > Recent resources**.

2.  Confirm that the following resources exist:

    - **sqldb-zava** (SQL database)

    - **sql-zava-jd427** (SQL server)

    - **law-zava-jd427** (Log Analytics workspace)

      ![](./media/image120.png)

**Note: Do not delete these resources.** Lab 10 and Lab 11 use them.
