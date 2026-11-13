# Lab 10: Implement Governance and Data Protection

## Introduction

Zava Cycles is preparing for a privacy audit of its customer database. The auditor has set four requirements. Sales reps may see only their own customers. Nobody outside management may read full email addresses or password data. Every change to product prices must be provable after the fact. And the company must show evidence that these controls work, not just a description of them.

In this lab, you take on the role of the database administrator responsible for meeting those requirements in Azure SQL Database. You start by building a secured environment: an Azure SQL database with Microsoft Entra-only authentication, server auditing to Log Analytics, and Microsoft Defender for SQL. You then add data protection in layers, from roles and column permissions to row-level security and dynamic data masking. Next, you classify sensitive data so that audit logs report who accessed it, and you record price changes in a tamper-evident ledger table. Finally, you prove that everything works using Azure Policy compliance, a PASS/FAIL validation script and a vulnerability assessment scan.

### Objectives

After completing this lab, you will be able to:
- Secure an Azure SQL logical server with Microsoft Entra-only authentication, a restricted firewall, server auditing and Microsoft Defender for SQL.
- Apply least privilege to data by combining database roles, column-level DENY permissions, row-level security filter and block predicates, and dynamic data masking.
- Test access controls by impersonating different users and interpreting the expected permission and security-policy errors.
- Classify sensitive columns and use Log Analytics (KQL) to report who accessed classified data.
- Create a ledger table and verify its integrity with a database digest.
- Enforce and evaluate platform governance with built-in Azure Policy definitions, and set audit log retention to meet a compliance requirement.
- Validate governance controls with a repeatable PASS/FAIL script and a vulnerability assessment re-scan, using baselines to approve intended findings.
- Remove lab resources and policy assignments safely when the work is complete.


---

# Getting Started: Build the Secured Environment

### Task 1: Switch and verify Entra-only authentication, enable auditing and confirm network security

You created the server with SQL authentication, as in Lab 9. Now you make your account the Microsoft Entra administrator and turn SQL authentication off, so every sign-in uses Entra ID.

1. Return to the first tab. When the database deployment completes, click **Go to resource**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image22.png)

1. Click the **Server name** link on the **Overview** page to open the logical server.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image23.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image24.png)

1. In the server’s left menu, click **Settings > Microsoft Entra ID**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image25.png)

1. Click **Set admin**, search for your lab account +++@lab.CloudPortalCredential(User1).Username+++, select it and click **Select**. Click **Save**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image26.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image27.png)

1. On the same page, select **Support only Microsoft Entra authentication for this server** and click **Save**. The **sqladminuser** login remains defined but can no longer sign in.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image28.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image29.png)

1. In the server’s left menu, click **Security > Auditing**. Turn **Enable Azure SQL Auditing** on.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image30.png)

1. Select **Log Analytics**, choose your subscription and your workspace (**law-zava-@lab.labinstance.id**), and click **Save**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image31.png)

1. Click **Security > Networking**. On the **Public access** tab, confirm **Public network access** is **Selected networks**, a single firewall rule exists for your client IP, and **Allow Azure services and resources to access this server** is cleared.

    >[!Note] Server-level auditing covers every database on the server, including future ones. Do not also enable database-level auditing; it duplicates records.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image32.png)


### Task 2: Open Cloud Shell and set variables

1. On the portal toolbar, click the **Cloud Shell** icon (**>_**).

1. Choose **Bash**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image33.png)

1. If this is your first time, select **No storage account required**, choose your subscription and click **Apply**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image34.png)

1. Replace the example values with your own names and run:

    **Bash**

    ```
    RG="ResourceGroup1"
    SERVER="sql-zava-@lab.labinstance.id"
    DB="sqldb-zava-@lab.labinstance.id"
    LAW="law-zava-@lab.labinstance.id"

    # Check the values point at real resources
    az sql db show -g $RG -s $SERVER -n $DB \
      --query "{db:name, status:status}" -o table

    az monitor log-analytics workspace show -g $RG -n $LAW \
      --query name -o tsv
    ```

    **Expected result:** `sqldb-zava-@lab.labinstance.id` with status **Online** (or **Paused**), and your workspace name.

    >[!Note] Cloud Shell closes after about 20 minutes without input, and the variables are lost. Later Cloud Shell steps start with an `echo` check; if it shows empty values, or a command reports **expected one argument**, run the four variable lines again. To find your names, run `az group list --query "[].name" -o tsv` and `az monitor log-analytics workspace list -o table`.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image35.png)


### Task 5: Assign Azure Policy definitions (start evaluation early)

Policy evaluation takes time, so you assign the policies now and review the results in Exercise 3. Assign these four built-in definitions with their default **Audit** or **AuditIfNotExists** effect:
- Auditing on SQL server should be enabled
- Azure SQL Database should have Microsoft Entra-only authentication enabled
- Azure Defender for SQL should be enabled for unprotected Azure SQL servers
- Azure SQL Database should be running TLS version 1.2 or newer


Use **one** of the two options below, not both; using both creates duplicate assignments.

### Option A: Cloud Shell (recommended, about 1 minute)

**Bash**

```
RGID=$(az group show -n $RG --query id -o tsv)

POLICIES=(
  "Auditing on SQL server should be enabled"
  "Azure SQL Database should have Microsoft Entra-only authentication enabled"
  "Azure Defender for SQL should be enabled for unprotected Azure SQL servers"
  "Azure SQL Database should be running TLS version 1.2 or newer"
)

for P in "${POLICIES[@]}"; do
  Q="[?policyType=='BuiltIn' && displayName=='$P'].name | [0]"
  DEF=$(az policy definition list --query "$Q" -o tsv)
  if [ -z "$DEF" ]; then echo "NOT FOUND: $P"; continue; fi
  az policy assignment create --name "sqlgov-${DEF:0:12}" \
    --display-name "$P" --policy "$DEF" --scope "$RGID" -o none \
    && echo "Assigned: $P"
done

# Ask Azure Policy to evaluate the resource group now
az policy state trigger-scan -g $RG --no-wait
```

**Expected result:** four **Assigned:** lines. If you see **NOT FOUND**, the display name has been revised; assign that definition with Option B, searching by keyword (for example Entra-only or Defender for SQL). If you see **AuthorizationFailed**, you lack the policy role; tell your instructor.

### Option B: Azure portal

1. In the portal search bar at the top, type `Policy` and select **Policy** under **Services**. In the left menu, click **Authoring > Definitions**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image36.png)

1. Click the **Category** filter, select only **SQL**, and click **Apply**.

1. In the **Search** box, type +++Auditing on SQL server+++. Click **Auditing on SQL server should be enabled** (the blue link) to open it.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image37.png)

1. Click **Assign policy**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image38.png)

1. On the **Basics** tab, click the **Scope** ellipsis (**…**), select your subscription **and then your lab resource group**, and click **Select**. Do not leave the scope at the subscription only. Leave **Policy enforcement** at **Default (enforcement)**. Click **Review + create**, then **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image39.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image40.png)

1. Return to **Authoring > Definitions**, clear the **Search** box and type **Entra-only authentication**. The search returns about six results.

    >[!Note] Do not choose the Managed Instance ones, the ones ending “during creation”, or the initiative without “enabled”.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image41.png)

1. Open **Azure SQL Database should have Microsoft Entra-only authentication enabled** (the blue link) and select **Assign policy**. Set the scope as in step 5, click **Review + create**, then select **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image42.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image43.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image44.png)

1. Clear the **Search** box and type **unprotected Azure SQL servers**. Open **Azure Defender for SQL should be enabled for unprotected Azure SQL servers** (the name may start with “Microsoft Defender” instead). Click **Assign policy**, set the scope as in step 5, then select **Review + create** and **Create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image45.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image46.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image47.png)

1. Clear the **Search** box and type **TLS version 1.2**. Open **Azure SQL Database should be running TLS version 1.2 or newer** (not the SQL Managed Instance one) and assign it as in step 5.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image48.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image49.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image50.png)

    **“No policies found in the given scope”?** Check the **Search** box contains only the search text (clear anything else), and that the filters show **Category : SQL**, **Definition type : All definition types** and **Policy type : All policy types**. Then click **Refresh**.


### Task 6: Connect to the database

Use **Option A** if SSMS is installed on your lab machine; otherwise use **Option B**. Later steps say “in your query window”, meaning whichever you chose.

### Option A: SQL Server Management Studio (SSMS)

1. Open SSMS. In the **Connect to Server** window, set **Server type** to **Database Engine**, **Server name** to your server name plus `.database.windows.net` (for example `sql-zava-jd427.database.windows.net`), and **Authentication** to **Microsoft Entra MFA** with your lab account as **User name**.

1. Set **Database name** to `sqldb-zava`. Leave **Encrypt** at **Mandatory** and **Trust server certificate** cleared. Click **Connect**, complete the sign-in, then click **New Query**.


### Option B: Azure portal Query editor

1. Open the database **sqldb-zava**. In the left menu, click **Query editor**. (Use the default editor, not **Classic experience (preview)**.)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image51.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image52.png)

1. On the **Microsoft Entra authentication** tab, click **Connect as *{your account}***. Click **New query**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image53.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image54.png)


### Check the connection

1. Run the following query:

    ```
    SELECT SUSER_SNAME() AS login_name, USER_NAME() AS database_user, DB_NAME() AS database_name,
           (SELECT COUNT(*) FROM SalesLT.Customer) AS customer_rows;
    ```

    **Expected result:** `login_name` is your lab account, `database_user` is **dbo** (you are the Entra admin), `database_name` is `sqldb-zava`, and `customer_rows` is **847**.

    **Running the scripts in this lab:** copy only the code lines, not the grey label above each block. Some scripts use **GO** to separate batches. SSMS handles GO; if the Query editor reports an error near GO, highlight each section between GO lines and run it on its own, in order.
    
    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image55.png)


## Exercise 1: Configure Data Access Controls

**Duration:** 12 minutes

You layer four controls on `SalesLT.Customer`: roles grant table access, a column DENY hides password data, RLS filters rows per rep, and DDM masks contact details. Users without logins stand in for real Entra users so you can impersonate them.

### Task 1: Create roles and test users

1. In your query window, run:

    ```
    CREATE ROLE SalesReps;
    CREATE ROLE SalesManagers;

    CREATE USER Rep_Pamela WITHOUT LOGIN;
    CREATE USER Rep_David WITHOUT LOGIN;
    CREATE USER Mgr_Linda WITHOUT LOGIN;

    ALTER ROLE SalesReps ADD MEMBER Rep_Pamela;
    ALTER ROLE SalesReps ADD MEMBER Rep_David;
    ALTER ROLE SalesManagers ADD MEMBER Mgr_Linda;

    GRANT SELECT ON SCHEMA::SalesLT TO SalesReps, SalesManagers;
    GRANT INSERT ON SalesLT.Customer TO SalesReps;

    -- Column-level protection
    DENY SELECT ON SalesLT.Customer (PasswordHash, PasswordSalt) TO SalesReps;
    ```

   **Production note:** Users without logins cannot sign in; they exist only for impersonation tests and are allowed with Entra-only authentication. In production, add Entra groups to these roles instead, for example `CREATE USER [SG-SalesReps] FROM EXTERNAL PROVIDER;`

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image56.png)


### Task 2: Implement row-level security

1. Check which sales reps exist in the data:

    ```
    SELECT SalesPerson, COUNT(*) AS customers
    FROM SalesLT.Customer GROUP BY SalesPerson ORDER BY customers DESC;
    ```

    **Expected result:** about nine rows, including `adventure-works\pamela0` and `adventure-works\david8`.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image57.png)

1. Create the mapping table, predicate function and security policy. If the values in your data differ, adjust the `SalesPerson` values in the INSERT to match the query above.

    ```
    CREATE SCHEMA Security;
    GO

    CREATE TABLE Security.SalesRepMap (
        UserName    sysname       NOT NULL PRIMARY KEY,
        SalesPerson nvarchar(256) NOT NULL
    );

    INSERT Security.SalesRepMap VALUES
        (N'Rep_Pamela', N'adventure-works\pamela0'),
        (N'Rep_David',  N'adventure-works\david8');
    GO

    CREATE FUNCTION Security.fn_customer_predicate (@SalesPerson nvarchar(256))
    RETURNS TABLE
    WITH SCHEMABINDING
    AS RETURN
        SELECT 1 AS allowed
        WHERE IS_MEMBER('SalesManagers') = 1
           OR IS_MEMBER('db_owner') = 1
           OR EXISTS (SELECT 1 FROM Security.SalesRepMap m
                      WHERE m.UserName = USER_NAME() AND m.SalesPerson = @SalesPerson);
    GO

    CREATE SECURITY POLICY Security.CustomerPolicy
        ADD FILTER PREDICATE Security.fn_customer_predicate(SalesPerson) ON SalesLT.Customer,
        ADD BLOCK PREDICATE Security.fn_customer_predicate(SalesPerson) ON SalesLT.Customer AFTER INSERT
        WITH (STATE = ON);
    ```

    The **filter** predicate hides other reps’ rows on read; the **block** predicate stops a rep inserting a customer for someone else.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image58.png)


### Task 3: Apply dynamic data masking

1. In your query window, run:

    ```
    ALTER TABLE SalesLT.Customer
        ALTER COLUMN EmailAddress ADD MASKED WITH (FUNCTION = 'email()');

    ALTER TABLE SalesLT.Customer
        ALTER COLUMN Phone ADD MASKED WITH (FUNCTION = 'partial(0,"XXX-XXX-",4)');

    GRANT UNMASK TO SalesManagers;
    ```

    >[!Note] DDM is a presentation control, not encryption. Users with ad hoc query rights can infer masked values with predicates, so combine DDM with permissions.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image59.png)


### Task 4: Test as each persona

Each block below switches to a test user, runs one test, and switches back with `REVERT`. **Run each block on its own:** select all of its lines and click **Run** (Query editor) or press **F5** (SSMS). Two blocks fail on purpose; running them separately lets you see every result and every expected error.

1. Block A: what Rep_Pamela can see. Run: <!-- TODO: Need to check/add image -->

    ```
    EXECUTE AS USER = 'Rep_Pamela';
    SELECT TOP (5) COUNT(CustomerID) OVER () AS my_customers,
           FirstName, LastName, EmailAddress, Phone, SalesPerson
    FROM SalesLT.Customer;
    REVERT;
    ```

    **Expected result:** five rows. `my_customers` is the number of rows Pamela can see (well below 847), `SalesPerson` is always `adventure-works\pamela0`, `EmailAddress` is masked (for example +++a@lab.LabInstance.Id+++@@lab.LabInstance.Id.com) and `Phone` shows only the last four digits (+++@lab.LabInstance.Id-0173+++).

    **Why COUNT(CustomerID) and not COUNT(\*)?** `COUNT(*)` needs SELECT permission on every column of the table, including `PasswordHash`, which SalesReps are denied, so it fails with a PasswordHash permission error. Counting a permitted column such as `CustomerID` returns the same number without touching the denied columns.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image60.png)

1. Block B: the password column. Run:

    ```
    EXECUTE AS USER = 'Rep_Pamela';
    SELECT TOP (1) PasswordHash FROM SalesLT.Customer;
    REVERT;
    ```

    **Expected result:** **The SELECT permission was denied on the column 'PasswordHash' of the object 'Customer'** (error 230). The Query editor also shows **Failed to execute the query**; that is the expected outcome.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image61.png)

1. Block C: inserting a customer for another rep. Run:

    ```
    EXECUTE AS USER = 'Rep_Pamela';
    INSERT SalesLT.Customer (FirstName, LastName, SalesPerson, PasswordHash, PasswordSalt)
    VALUES (N'Test', N'Blocked', N'adventure-works\david8', 'x', 'y');
    REVERT;
    ```

    **Expected result:** an error stating that **the target object has a block predicate that conflicts with this operation** (error 33504). No row is inserted.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image62.png)

1. Block D: what Mgr_Linda can see. Run:

    ```
    EXECUTE AS USER = 'Mgr_Linda';
    SELECT TOP (5) COUNT(*) OVER () AS all_customers,
           FirstName, LastName, EmailAddress, Phone
    FROM SalesLT.Customer;
    REVERT;
    ```

    **Expected result:** `all_customers` is **847**, and `EmailAddress` and `Phone` are shown in clear text because SalesManagers has UNMASK.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image63.png)

1. Confirm you are back to your own identity:

    `SELECT USER_NAME() AS current_user_name; -- expect dbo`

    **If this does not return dbo,** an error stopped a block before `REVERT` ran, and your session is still impersonating a test user. Run `REVERT;` and check again. Later scripts fail with permission errors if you skip this.

    | Persona | Rows visible | EmailAddress | PasswordHash | Insert for another rep |
    |---|---|---|---|---|
    | Rep_Pamela | Only her customers | Masked (+++a@lab.LabInstance.Id+++@@lab.LabInstance.Id.com) | Permission denied | Blocked |
    | Mgr_Linda | All customers (847) | Clear text | Readable | Not granted |
    | Entra admin (db_owner) | All customers | Clear text | Readable | Allowed |

    **Evidence to capture:** Query results for both personas, including the two expected errors.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image64.png)

    ---


## Exercise 2: Implement Auditing and Compliance Controls

**Duration:** 13 minutes

You label sensitive columns so audit logs report when they are read, create a ledger table whose history can be cryptographically verified, and set audit retention.

### Task 1: Classify sensitive data

1. In the portal, open the database **sqldb-zava**. In the left menu, click **Security > Data Discovery & Classification**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image65.png)

1. Click the **Classification** tab. Select the recommendations for **SalesLT.Customer**, click **Accept selected recommendations**, then click **Save**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image66.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image67.png)

1. Make the labels repeatable with T-SQL. This sets the exact labels the auditor expects, overriding any portal recommendation for these columns:

    ```
    ADD SENSITIVITY CLASSIFICATION TO SalesLT.Customer.EmailAddress, SalesLT.Customer.Phone
        WITH (LABEL = 'Confidential - GDPR', INFORMATION_TYPE = 'Contact Info', RANK = MEDIUM);

    ADD SENSITIVITY CLASSIFICATION TO SalesLT.Customer.PasswordHash, SalesLT.Customer.PasswordSalt
        WITH (LABEL = 'Highly Confidential', INFORMATION_TYPE = 'Credentials', RANK = CRITICAL);

    SELECT OBJECT_NAME(sc.major_id) AS table_name, c.name AS column_name,
           sc.label, sc.information_type, sc.rank_desc
    FROM sys.sensitivity_classifications sc
    JOIN sys.columns c ON c.object_id = sc.major_id AND c.column_id = sc.minor_id
    ORDER BY sc.rank DESC;
    ```

    **Expected result:** at least four rows, with `PasswordHash` and `PasswordSalt` ranked **CRITICAL**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image68.png)


### Task 2: Generate a sensitive read

1. Run this query as yourself. Auditing records that it touched classified columns:

    `SELECT TOP (20) FirstName, LastName, EmailAddress, Phone FROM SalesLT.Customer;`

    **Audit data takes 5 to 10 minutes to reach Log Analytics.** Continue with Tasks 3 to 5 now; you check the audit record in Task 5.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image69.png)


### Task 3: Create a ledger table

1. In your query window, run:

    ```
    CREATE SCHEMA Compliance;
    GO

    CREATE TABLE Compliance.PriceChanges (
        ChangeId  int IDENTITY PRIMARY KEY,
        ProductId int NOT NULL,
        OldPrice  money NOT NULL,
        NewPrice  money NOT NULL,
        ChangedBy sysname NOT NULL DEFAULT SUSER_SNAME(),
        ChangedAt datetime2(0) NOT NULL DEFAULT SYSUTCDATETIME()
    )
    WITH (SYSTEM_VERSIONING = ON, LEDGER = ON);
    GO

    INSERT Compliance.PriceChanges (ProductId, OldPrice, NewPrice) VALUES (680, 1431.50, 1399.00);

    UPDATE Compliance.PriceChanges SET NewPrice = 1379.00 WHERE ProductId = 680;

    SELECT * FROM Compliance.PriceChanges_Ledger
    ORDER BY ledger_transaction_id, ledger_sequence_number;
    ```

    **Expected result:** three rows. The INSERT, then a DELETE and INSERT pair for the UPDATE, each tied to a transaction and to your account.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image70.png)


### Task 4: Generate and verify a digest

1. (Optional) To see what a digest looks like, generate one on its own. It is a small JSON document holding the latest block ID and its hash:

    `EXECUTE sp_generate_database_ledger_digest;`

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image71.png)

1. Generate a digest and verify the ledger against it in one script. Run the whole script in one go; it captures the digest itself, so there is nothing to copy and paste:


    ```
    CREATE TABLE #digest (latest_digest nvarchar(max));
    INSERT INTO #digest EXECUTE sp_generate_database_ledger_digest;

    DECLARE @d nvarchar(max) = (SELECT N'[' + latest_digest + N']' FROM #digest);

    SELECT @d AS digest_used;              -- the exact digest being verified
    EXECUTE sp_verify_database_ledger @d;  -- verifies the ledger against it

    DROP TABLE #digest;
    ```

    **Expected result:** no error, and a result named **last_verified_block_id** with the value **0** (or higher). This proves the ledger data matches the digest. The procedure expects a JSON array, which is why the script wraps the digest in square brackets.

    **Query editor shows only one result?** The editor displays the last result set, so you see `last_verified_block_id` but not `digest_used`. That is expected. In SSMS, both results appear.

    **Production note:** enable automatic digest storage (database **Security > Ledger**) to immutable Azure Blob Storage or Azure Confidential Ledger, so digests are kept outside the control of database administrators.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image72.png)


### Task 5: Set retention and produce a compliance report

Auditors expect logs to be kept for a defined period and summarised on request.

1. In Cloud Shell, set the workspace retention to 90 days and confirm it:

    ```Bash
    echo "RG=$RG LAW=$LAW"  # both must show values; if empty, rerun Getting Started, Task 4

    # Keep audit data for 90 days (adjust to your policy)
    az monitor log-analytics workspace update -g $RG -n $LAW --retention-time 90 -o none

    az monitor log-analytics workspace show -g $RG -n $LAW --query retentionInDays -o tsv
    ```

    **Expected result:** **90**. Retention beyond the included 31 days adds a small storage charge.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image73.png)

1. Open your Log Analytics workspace (for example **law-zava-jd427**). In the left menu, click **Logs**. Close the **Queries hub** window if it opens.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image74.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image75.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image76.png)

1. Turn the **Agent** toggle off.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image77.png)

1. Switch from **Simple mode** to **KQL mode** using the drop-down at the top right of the query pane.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image78.png)

1. Confirm the audit record from Task 2. Paste and run the query:

    ```KQL
    AzureDiagnostics
    | where Category == "SQLSecurityAuditEvents"
    | where isnotempty(data_sensitivity_information_s)
    | project TimeGenerated, server_principal_name_s, statement_s,
              data_sensitivity_information_s
    | order by TimeGenerated desc
    ```

    The `data_sensitivity_information_s` column lists the labels each statement touched, giving the auditor a personal-data access report without parsing SQL. If no rows appear yet, wait 2 to 3 minutes and run it again.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image79.png)

1. Run the summary report:

    ```KQL
    AzureDiagnostics
    | where Category == "SQLSecurityAuditEvents"
    | where TimeGenerated > ago(24h)
    | where isnotempty(data_sensitivity_information_s)
    | summarize SensitiveReads = count(), LastAccess = max(TimeGenerated)
        by server_principal_name_s, database_principal_name_s
    | order by SensitiveReads desc
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image80.png)

1. Click **Save > Save as query**, name it **Sensitive data access report**, and click **Save**, so it can be rerun for every audit.

    **Evidence to capture:** Classification list, the ledger view, successful digest verification, the retention value, and both KQL results.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image81.png)


## Exercise 3: Validate Governance Policies

**Duration:** 12 minutes

Azure Policy proves the platform is configured correctly; a validation script proves the data controls behave correctly; a vulnerability re-scan proves the changes introduced no new risk.

### Task 1: Review policy compliance

1. In Cloud Shell, list the compliance state of your assignments:

    ```bash
    echo "RG=$RG LAW=$LAW"  # both must show values; if empty, rerun Getting Started, Task 4

    Q="[?contains(policyAssignmentName,'sqlgov')].{policy:policyDefinitionName,"
    Q+=" resource:resourceId, state:complianceState}"

    az policy state list -g $RG --query "$Q" -o table
    ```

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image82.png)

1. In the portal, open **Policy > Compliance**. In the list, find the four lab policies and check the **Compliance state** column. To see the detail, click one (for example **Auditing on SQL server should be enabled**); it lists your server as **Compliant**. You do not need to open all four.

    **Expected result:** your server is **Compliant** for all four assignments, because Getting Started set Entra-only authentication, server auditing, Defender for SQL and TLS 1.2.

    **Same policy listed twice, or more than four policies?** A duplicate usually means the policy was assigned by both Option A and Option B, or once at subscription scope; it does not change the result. The clean-up section at the end removes every copy. Other policies in the list belong to your subscription and are not part of this lab.

    **If the list is empty or shows Not started,** evaluation has not finished. Run `az policy state trigger-scan -g $RG --no-wait`, continue with Task 2, and check again at the end of the lab. Record the status you see; “evaluation pending” is an acceptable result for the lab.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image83.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image84.png)


### Task 2: Run the validation script

1. In your query window, run:


    ```T-Sql
    DECLARE @results TABLE (control nvarchar(100), outcome nvarchar(10));
    DECLARE @all int = (SELECT COUNT(*) FROM SalesLT.Customer), @rep int, @email nvarchar(50);

    EXECUTE AS USER = 'Rep_Pamela';
    SET @rep = (SELECT COUNT(CustomerID) FROM SalesLT.Customer); -- not COUNT(*): PasswordHash is denied
    SET @email = (SELECT TOP (1) EmailAddress FROM SalesLT.Customer);
    REVERT;

    INSERT @results VALUES
    ('RLS filters rep rows', IIF(@rep > 0 AND @rep < @all, 'PASS', 'FAIL')),
    ('Email masked for reps', IIF(@email LIKE '%XXX%', 'PASS', 'FAIL')),
    ('PasswordHash denied to reps', IIF(EXISTS (SELECT 1 FROM sys.database_permissions
        WHERE state_desc = 'DENY' AND grantee_principal_id = DATABASE_PRINCIPAL_ID('SalesReps')
        AND minor_id = COLUMNPROPERTY(OBJECT_ID('SalesLT.Customer'), 'PasswordHash', 'ColumnId')), 'PASS', 'FAIL')),
    ('Security policy enabled', IIF(EXISTS (SELECT 1 FROM sys.security_policies
        WHERE name = 'CustomerPolicy' AND is_enabled = 1), 'PASS', 'FAIL')),
    ('Sensitive columns classified', IIF((SELECT COUNT(*) FROM sys.sensitivity_classifications) >= 4, 'PASS', 'FAIL')),
    ('Price changes in ledger', IIF(EXISTS (SELECT 1 FROM sys.tables
        WHERE name = 'PriceChanges' AND ledger_type_desc = 'UPDATABLE_LEDGER_TABLE'), 'PASS', 'FAIL'));

    SELECT * FROM @results;
    ```

    **Expected result:** six rows, every one **PASS**. For a FAIL, return to the task that created the control, fix it, and rerun. If `@all` looks too small, check that `SELECT USER_NAME();` returns `dbo` (see Exercise 1, Task 4).

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image85.png)


### Task 3: Re-scan for vulnerabilities

1. Open the database **sqldb-zava**. In the left menu, click **Security > Microsoft Defender for Cloud**. Confirm **Microsoft Defender for SQL** shows **On**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image86.png)

1. Under **Vulnerabilities on this database**, click **Scan now**. The scan takes 1 to 5 minutes; watch the **Notifications** (bell) icon. **To save time,** fill in the evidence table in Task 4 while it runs.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image86.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image87.png)

1. When the scan completes, refresh the page and click **View database vulnerability summary**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image88.png)

1. Check that the last scan time is after you clicked **Scan now**; if not, click **Scan history** and wait for the scan to finish.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image89.png)

1. Clear any filters and review the **Findings** tab. Expect two new findings caused by the governance objects you created in Exercise 1. Open each one and check its **Results** list contains only what you created:

    | ID | Security check | Why it appears |
    |---|---|---|
    | VA1281 | All memberships for user-defined roles should be intended (Medium) | Rep_Pamela and Rep_David in SalesReps; Mgr_Linda in SalesManagers |
    | VA2130 | Track all users with access to the database (Low) | The three test users Rep_Pamela, Rep_David and Mgr_Linda |

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image90.png)

1. For each of these findings, click **Add all results as baseline**, then **Yes**. The status changes to **Healthy** and each result shows **In Baseline**. Any other finding, and any **High** finding, must be investigated and recorded in the evidence table.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image91.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image92.png)

1. Click **Scan** again and wait for it to complete, then click **Refresh**.

    **Expected result:** **Total vulnerabilities** is **0**, and VA1281 and VA2130 appear on the **Passed** tab as **Passed per custom baseline**.

    **Ignore the Recommendations (0) and Security Alerts (0) counters** on the Defender for Cloud page. They are updated later by Defender for Cloud and are not used in this lab; findings come from **View database vulnerability summary**.


>[!Alert] **Continuing to Lab 11?** Do not clean up; keep all resources. Do this section only when you have finished with the environment. Deletion is permanent: the server, the database and its sample data cannot be recovered.

# Clean Up Resources

>[!Alert] **Continuing to Lab 11?** Do not clean up; keep all resources. Do this section only when you have finished with the environment. Deletion is permanent: the server, the database and its sample data cannot be recovered.
>
**Duration:** about 5 minutes (after the lab; not part of the 50 minutes)

Delete the policy assignments first, because an assignment made at subscription scope is not removed when you delete your resources. Then delete the three lab resources. Deleting the server also stops Defender for SQL billing for it.

### Task 1: Delete the policy assignments

### Option A: Azure portal

1. In the portal search bar, type +++Policy+++ and select **Policy**. In the left menu, click **Authoring > Assignments**.

1. In the **Search** box, type `unprotected`. Each lab policy can appear more than once, for example at **Depth-lod…** (subscription) scope and at **Depth-lod…/ResourceGroup1** scope. You must delete every copy.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image93.png)

1. For each row, click the ellipsis (**…**) at the end of the row, select **Delete assignment**, and click **Yes**.

1. Repeat steps 2 and 3, typing each of these in the **Search** box: **Auditing on SQL server**, **Entra-only authentication** and **TLS version 1.2**.

1. Clear the **Search** box.

    **Expected result:** none of the four lab policies is listed. Only assignments that existed before the lab remain.


### Option B: Cloud Shell

This lists every assignment of the four lab policies, at subscription and resource group scope, then deletes them. Check the list before you run the delete loop: it should show only the four lab policies (some may appear twice).


```bash
bash
echo "RG=$RG"  # must show a value; if empty, set RG again

F="contains(displayName,'Auditing on SQL server should be enabled')"
F+=" || contains(displayName,'SQL Database should have Microsoft Entra-only')"
F+=" || contains(displayName,'for unprotected Azure SQL servers')"
F+=" || contains(displayName,'SQL Database should be running TLS version 1.2')"

# 1. List what will be deleted
az policy assignment list -g $RG --disable-scope-strict-match \
  --query "[?$F].{name:name, scope:scope, policy:displayName}" -o table

# 2. Delete them
az policy assignment list -g $RG --disable-scope-strict-match \
  --query "[?$F].[name, scope]" -o tsv |
while read NAME SCOPE; do
  az policy assignment delete --name "$NAME" --scope "$SCOPE" \
    && echo "Deleted: $NAME"
done
```

**Expected result:** one **Deleted:** line for each assignment in the list. Run step 1 again; it returns no rows.

### Task 2: Delete the lab resources

> **Delete the resources, not the resource group,** if your lab environment provided the resource group (for example ResourceGroup1). If you created the resource group yourself for this lab and it holds nothing else, you can instead open **Resource groups**, select it, click **Delete resource group**, type its name and click **Delete**.

### Option A: Azure portal

1. In the portal search bar, type **All resources** and select **All resources**. Set the **Resource group** filter to your lab resource group.

1. Select the check boxes for the three lab resources: the database **sqldb-zava**, the SQL server **sql-zava-jd427** and the Log Analytics workspace **law-zava-jd427** (your names may differ).

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2010/media/image94.png)

1. On the toolbar, click **Delete** (it may be under **…** or **More**). In the confirmation pane, check the list shows only these three resources, type `delete` in the confirmation box, and click **Delete**.

1. Watch the **Notifications** (bell) icon. Deletion takes 1 to 3 minutes. Click **Refresh**.

    **Expected result:** the three resources are no longer listed.


### Option B: Cloud Shell


```bash
bash
echo "RG=$RG SERVER=$SERVER LAW=$LAW"  # all three must show values

# Deleting the server also deletes its databases
az sql server delete -g $RG -n $SERVER --yes

# --force deletes the workspace permanently instead of soft-deleting it
az monitor log-analytics workspace delete -g $RG -n $LAW --yes --force true

# Confirm: none of the three lab resources is listed
az resource list -g $RG --query "[].{name:name, type:type}" -o table
```

>[!knowledge] **About the workspace:** a workspace deleted in the portal is soft-deleted and can be recovered for 14 days, and its name stays reserved during that time. To reuse the same name sooner, delete it permanently: open the workspace, click **Delete**, select **Delete the workspace permanently**, and confirm. The Cloud Shell command above deletes it permanently.

### Task 3: Confirm the clean-up

1. In **Policy > Authoring > Assignments**, confirm none of the four lab policies is listed.

1. In **All resources**, filtered to your lab resource group, confirm `sqldb-zava`, `sql-zava-jd427` and `law-zava-jd427` are gone.
