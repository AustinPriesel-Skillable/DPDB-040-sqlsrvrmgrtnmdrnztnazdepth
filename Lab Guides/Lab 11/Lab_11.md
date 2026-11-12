# Lab 11: Enable AI-Ready Capabilities in Azure SQL

## Introduction

Zava wants its shopping assistant to answer natural-language questions such as “a bike that copes with rough mountain trails”. Keyword search misses questions like this, and copying product data into a separate vector store would bypass the governance built in Labs 9 and 10.

In this lab you turn the secured database from Labs 9 and 10 into a retrieval engine for that assistant, keeping embeddings next to the product data under the same security. You store embeddings in the native **vector** type, generate them inside the database with **AI_GENERATE_EMBEDDINGS** and an Azure OpenAI model in Microsoft Foundry reached through a managed identity, run exact and approximate semantic search, and combine it with keyword search.

### Objectives

By the end of this lab, you will be able to:
- Design an AI-ready Azure SQL architecture and justify its key decisions.
- Implement vectors, embeddings and semantic retrieval patterns, including hybrid search and retrieval-augmented generation (RAG).
- Optimize performance for AI and operational workloads with Query Store, caching, filtering and indexing decisions.


### Prerequisites

- **Lab 9 completed (required).** Lab 11 uses its server **sql-zava-jd427**, its database **sqldb-zava** with the AdventureWorksLT sample, your Microsoft Entra admin account (SQL authentication is turned off), the **SQL-Analysts** database user and the firewall rule for your IP address.
- The server’s system-assigned managed identity is turned on (Lab 9, Getting started, Task 3). Setup Task 1 checks this.
- **Lab 10 completed (recommended, not required).** Lab 11 refers to Lab 10’s governance controls but does not use any objects Lab 10 creates.
- Permission to create a Microsoft Foundry resource and deploy models in your lab resource group (Setup Task 0), or a Foundry resource from your instructor with a deployment named **text-embedding-3-small** (1536 dimensions).
- Rights to assign roles on that resource (you have them on a resource you create), or an instructor who does it for you.
- SQL Server Management Studio 22 or Visual Studio Code with the MSSQL extension, both of which support the vector type and external models.


**Starting without Lab 9?** Do these Lab 9 steps first (about 15 minutes): Getting started, Task 1 (create **sqldb-zava** on **sql-zava-jd427** with sample data and your client IP) and Task 3 (managed identity); then Exercise 1, Task 1 (make yourself the Microsoft Entra admin). If you skip creating the **SQL-Analysts** group, remove the **GRANT … TO [SQL-Analysts]** line in Exercise 1, Task 5.

### Setup: Connect the Database to Azure OpenAI

**Values used in this lab:** server **sql-zava-@lab.labinstance.id**, database **sqldb-zava-@lab.labinstance.id** and resource group **ResourceGroup1**, and your Foundry resource **aif-zava-@lab.labinstance.id** (Task 0). Wherever a script shows **<lab-user-number>**, replace it, including the angle brackets, with your lab user number [@lab.labinstance.id].

### Task 0: Create the Foundry resource and deploy the embedding model

Each learner needs a Microsoft Foundry resource in the lab resource group. Resource names must be unique across Azure, so include your **lab user number**: the digits in your sign-in name (for example, User1-67029880 gives `aif-zava-@lab.labinstance.id`). If your instructor provides a resource instead, skip this task and use that name wherever this guide shows +++aif-zava-@lab.labinstance.id+++.

1. In the Azure portal, click **Foundry** under **Azure services** or search for and select `Foundry`.

1. Click **Create a Foundry resource**.

1. Set **Resource group** to **ResourceGroup1**, **Name** to `aif-zava-@lab.labinstance.id` and **Region** to **canadaeast, eastus2 or westus**. If asked for a default project name, use `proj-zava-@lab.labinstance.id`

1. If the portal reports that the sub-domain name is already used, add a digit (for example `aif-zava-@lab.labinstance.id`). Note the exact name: every later step uses it.

1. Click **Review + create**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image3.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image4.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image5.png)

1. Click the **Cloud Shell** icon (`>_`) at the top of the portal, choose **Bash**, and run the commands below to deploy the embedding model. Capacity 50 (50,000 tokens per minute) lets the load in Exercise 2 run without throttling.

    ```bash
    # Replace the example name on the AOAI_NAME line with your Foundry resource name
    RG=ResourceGroup1
    AOAI_NAME=aif-zava-@lab.labinstance.id
    
    az cognitiveservices account deployment create -g $RG -n $AOAI_NAME \
      --deployment-name text-embedding-3-small \
      --model-name text-embedding-3-small --model-version 1 --model-format OpenAI \
      --sku-name Standard --sku-capacity 50
    
    az cognitiveservices account deployment list -g $RG -n $AOAI_NAME \
      --query "[].{deployment:name, model:properties.model.name, state:properties.provisioningState}" -o table
    ```

    **Expected result:** one row, **text-embedding-3-small | text-embedding-3-small | Succeeded**. The deployment name must be exactly **text-embedding-3-small**. If you get a quota error, use +++--sku-capacity 30+++. Cloud Shell may warn that the subscription is not registered to Microsoft.CloudShell; you can ignore it.

>[!Knowledge] **Portal alternative:** open the resource, click **Go to Foundry portal**, then **Models + endpoints > + Deploy model > Deploy base model**. Choose **text-embedding-3-small**, keep the deployment name exactly +++text-embedding-3-small+++, and set the rate limit to 50K tokens per minute.

### Task 1: Grant the server identity access to the model

1. In the logical server’s left menu, click **Security > Identity**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image35.png)

1. Under **System assigned managed identity**, set **Status** to **On**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image36.png)

1. Click **Save**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image37.png)

1. Confirm that **Status** is **On** and that **Save** is greyed out, which means the change is saved.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image38.png)

1. In the Azure portal, open your Foundry resource **aif-zava-@lab.labinstance.id**. Do **not** use the Microsoft Foundry overview page; **IAM is only on the resource itself.**

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image6.png)

1. Select **Access control (IAM) > + Add > Add role assignment**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image7.png)

1. Search for and select `Cognitive Services OpenAI User`, then **Next**.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image8.png)

1. Select **Managed identity**, click **+ Select members**, set **Managed identity** to **SQL server**, select **sql-zava-@lab.labinstance.id**, click **Select**, then click **Review + assign** twice. Do not choose Foundry or Foundry project.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image9.png)

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image10.png)

1. Confirm the assignment: on the **Role assignments** tab, click **Refresh** and search for +++sql-zava-@lab.labinstance.id+++. It must be listed under **Cognitive Services OpenAI User** with type **SQL server**. Wait about 5 minutes before Task 2.

    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image11.png)


    **CLI alternative (Cloud Shell, Bash). In Bash, never leave <…> placeholders: < is read as a file redirect.**
    
    ```bash
    # Replace the example Foundry name with your own before running
    RG=ResourceGroup1
    SERVER=sql-zava-@lab.labinstance.id
    AOAI_NAME=aif-zava-@lab.labinstance.id
    
    az sql server update -g $RG -n $SERVER -i   # turns on the identity; no change if already on
    
    SQL_MI=$(az sql server show -g $RG -n $SERVER --query identity.principalId -o tsv)
    AOAI_ID=$(az cognitiveservices account show -g $RG -n $AOAI_NAME --query id -o tsv)
    echo "SQL identity: $SQL_MI"; echo "Foundry resource: $AOAI_ID"
    
    az role assignment create --assignee-object-id $SQL_MI \
      --assignee-principal-type ServicePrincipal \
      --role "Cognitive Services OpenAI User" --scope $AOAI_ID
    
    az role assignment list --assignee $SQL_MI --scope $AOAI_ID -o table
    ```

**Outbound access:** on the server’s **Security > Networking > Connectivity** tab, confirm that **Outbound networking** shows **Restrictions disabled**; no change is needed. Only if restrictions are enabled, open **Configure outbound networking restrictions** and add **aif-zava-*{lab-user-number}*.openai.azure.com**, or every model call fails.

### Task 2: Create the credential, external model and embedding procedure

**CREATE EXTERNAL MODEL** stores the endpoint, credential and model as one database object, and **AI_GENERATE_EMBEDDINGS** calls it from any query. This replaces hand-written REST calls and JSON parsing.

1. Replace **<lab-user-number>** with `@lab.labinstance.id` in all three places (the credential name, **LOCATION** and **CREDENTIAL**), for example with **Ctrl+H** in the editor, so the address matches your Foundry resource exactly. Then run the whole script at once without highlighting anything. Any text in angle brackets is a placeholder; if one is left in, the script fails with “Only alphanumeric characters… are allowed in the hostname”.

    ```T-sql
    -- 1. Clean up anything left from earlier attempts (makes the script safe to rerun)
    IF EXISTS (SELECT 1 FROM sys.external_models WHERE name = 'EmbeddingModel')
        DROP EXTERNAL MODEL EmbeddingModel;
    
    DECLARE @sql NVARCHAR(MAX) = N'';
    SELECT @sql += N'DROP DATABASE SCOPED CREDENTIAL ' + QUOTENAME(name) + N';'
    FROM sys.database_scoped_credentials
    WHERE name LIKE N'%openai.azure.com%';
    EXEC (@sql);
    GO
    
    -- 2. Master key (only if missing); use your own strong password
    IF NOT EXISTS (SELECT 1 FROM sys.symmetric_keys WHERE name = '##MS_DatabaseMasterKey##')
        CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'Use-A-Strong-Pa55word!';
    GO
    
    -- 3. Credential: the name must equal the endpoint URL
    CREATE DATABASE SCOPED CREDENTIAL [https://aif-zava-@lab.labinstance.id.openai.azure.com]
    WITH IDENTITY = 'Managed Identity',
         SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}';
    GO
    
    -- 4. Register the embedding deployment as a database object (preview)
    CREATE EXTERNAL MODEL EmbeddingModel
    WITH (
        LOCATION   = 'https://aif-zava-@lab.labinstance.id.openai.azure.com/openai/deployments/text-embedding-3-small/embeddings?api-version=2024-10-21',
        API_FORMAT = 'Azure OpenAI',
        MODEL_TYPE = EMBEDDINGS,
        MODEL      = 'text-embedding-3-small',
        CREDENTIAL = [https://aif-zava-@lab.labinstance.id.openai.azure.com]
    );
    GO
    
    -- 5. One entry point for the rest of the lab
    CREATE OR ALTER PROCEDURE dbo.get_embedding
        @text NVARCHAR(MAX),
        @embedding VECTOR(1536) OUTPUT
    AS
    BEGIN
        SET @embedding = CAST(AI_GENERATE_EMBEDDINGS(@text USE MODEL EmbeddingModel) AS VECTOR(1536));
        IF @embedding IS NULL
            THROW 50001, 'Embedding call failed. Check role assignment and external model.', 1;
    END;
    GO
    
    -- 6. Test
    DECLARE @v VECTOR(1536);
    EXEC dbo.get_embedding N'waterproof cycling jacket', @v OUTPUT;
    SELECT VECTORPROPERTY(@v, 'Dimensions') AS dims;
    ```
    
    **Expected result:** **dims = 1536**. HTTP 401 or 403 means the role assignment has not applied yet: wait a few minutes and rerun. HTTP 404 means the deployment is not named exactly +++text-embedding-3-small+++. “DNS resolution… failed” means the resource name is wrong. Red squiggles under +++USE+++ or **END** in the Query editor are not errors. See Troubleshooting for other messages.

**Why a managed identity:** No API key is stored in the database, and access can be revoked centrally, consistent with the Entra-only design from Lab 9. **Why an external model:** the endpoint and model can be changed in one place with **ALTER EXTERNAL MODEL**, and its use is controlled by permissions.

**Preview feature:** **CREATE EXTERNAL MODEL** and **AI_GENERATE_EMBEDDINGS** are in preview for Azure SQL Database at the time of this revision. If either statement fails, create **dbo.get_embedding** from Appendix A instead and continue; every later step works unchanged.

![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image12.png)

## Exercise 1: Create Vector-Enabled Database Objects

### Task 1: Explore the vector type

1. Run the following query:
    ```T-sql
    DECLARE @a VECTOR(3) = '[1, 0, 0]',
            @b VECTOR(3) = '[0.7, 0.7, 0]';
    
    SELECT VECTOR_DISTANCE('cosine',    @a, @b) AS cosine_distance,
           VECTOR_DISTANCE('euclidean', @a, @b) AS euclidean_distance,
           VECTOR_DISTANCE('dot',       @a, @b) AS negative_dot_product;
    ```
    
    Lower distance means more similar for all three metrics. Azure OpenAI embeddings are normalised, so cosine and dot product rank identically; cosine is the usual default.
    
    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image13.png)

### Task 2: Create the vector table

1. Run the following query:

    ```T-sql
    CREATE TABLE dbo.ProductEmbeddings (
        ProductID   INT            NOT NULL CONSTRAINT PK_ProductEmbeddings PRIMARY KEY,
        ProductName NVARCHAR(200)  NOT NULL,
        Category    NVARCHAR(50)   NOT NULL,
        ListPrice   MONEY          NOT NULL,
        Description NVARCHAR(MAX)  NOT NULL,   -- the exact text that is embedded
        Embedding   VECTOR(1536)   NULL,       -- native vector type (float32; float16 in preview)
        EmbeddedAt  DATETIME2(0)   NULL,
        CONSTRAINT FK_ProductEmbeddings_Product
            FOREIGN KEY (ProductID) REFERENCES SalesLT.Product (ProductID)
    );
    ```

### Design decisions:
- **Separate table:** each float32 vector is about 6 KB (1536 × 4 bytes), or about 3 KB with half-precision float16 (preview); keeping it out of **SalesLT.Product** protects order-processing performance.
- **Stored source text:** **Description** records exactly what was embedded, so re-embedding after a model change is reproducible.
- **Relational columns alongside the vector:** **Category** and **ListPrice** allow filtering before ranking.


### Task 3: Insert source records

1. Run the following query:

    ```T-sql
    INSERT dbo.ProductEmbeddings (ProductID, ProductName, Category, ListPrice, Description)
    SELECT p.ProductID, p.Name, pc.Name, p.ListPrice,
           CONCAT(p.Name, N'. Category: ', pc.Name, N'. ',
                  ISNULL(N'Color: ' + p.Color + N'. ', N''), d.Description)
    FROM SalesLT.Product p
    JOIN SalesLT.ProductCategory pc ON pc.ProductCategoryID = p.ProductCategoryID
    CROSS APPLY (   -- one English description per product
        SELECT TOP (1) pd.Description
        FROM SalesLT.ProductModelProductDescription pmd
        JOIN SalesLT.ProductDescription pd ON pd.ProductDescriptionID = pmd.ProductDescriptionID
        WHERE pmd.ProductModelID = p.ProductModelID AND pmd.Culture = N'en'
        ORDER BY pd.ProductDescriptionID
    ) AS d
    WHERE NOT EXISTS (SELECT 1 FROM dbo.ProductEmbeddings e WHERE e.ProductID = p.ProductID);
    
    SELECT COUNT(*) AS products FROM dbo.ProductEmbeddings;
    ```
    
    **Expected result:** about 290 rows with NULL embeddings. A few product models have more than one English description, so the query takes one per product; it also skips products already loaded, so it is safe to rerun.
    
    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image14.png)

### Task 4: Apply governance to AI objects

1. Run the following query:

    ```T-sql
    IF DATABASE_PRINCIPAL_ID('SQL-Analysts') IS NOT NULL
        GRANT SELECT ON dbo.ProductEmbeddings TO [SQL-Analysts];   -- database user from Lab 9, Exercise 1, Task 4
    ELSE
        PRINT 'SQL-Analysts not found: GRANT skipped (Lab 9 group not created).';
    
    ADD SENSITIVITY CLASSIFICATION TO dbo.ProductEmbeddings.Embedding
    WITH (LABEL = 'General', INFORMATION_TYPE = 'AI Embedding', RANK = LOW);
    
    -- Applications get EXECUTE on search procedures only (created in Exercise 3)
    IF DATABASE_PRINCIPAL_ID('AiSearchUsers') IS NULL
        CREATE ROLE AiSearchUsers;
    
    IF DATABASE_PRINCIPAL_ID('AssistantApp') IS NULL
        CREATE USER AssistantApp WITHOUT LOGIN;   -- stand-in for an app managed identity
    
    ALTER ROLE AiSearchUsers ADD MEMBER AssistantApp;
    
    -- Verify
    SELECT r.name AS role_name, m.name AS member_name
    FROM sys.database_role_members rm
    JOIN sys.database_principals r ON r.principal_id = rm.role_principal_id
    JOIN sys.database_principals m ON m.principal_id = rm.member_principal_id
    WHERE r.name = 'AiSearchUsers';
    
    SELECT grantee = USER_NAME(grantee_principal_id), permission_name
    FROM sys.database_permissions
    WHERE major_id = OBJECT_ID('dbo.ProductEmbeddings');
    ```
    
    **Expected result:** **AiSearchUsers | AssistantApp**, and **SQL-Analysts | SELECT**. If SQL-Analysts was not created in Lab 9, the grant is skipped and nothing later depends on it. The script can be rerun safely.

Embeddings can leak the meaning of the text they encode, so they inherit the sensitivity of their source. Product data is public; customer or HR text would need the Lab 10 controls. Do not grant **EXECUTE** on the external model to **AiSearchUsers**: the search procedures call it as owner, so applications cannot send arbitrary text to the model.

![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image15.png)

## Exercise 2: Load and Query Vector Data

### Task 1: Generate and store embeddings

This loop embeds 25 rows per set-based **UPDATE**, calling the external model directly, and pauses 5 seconds between batches to stay within the deployment’s rate limit. It is restartable because it only touches rows that are still NULL. Expect 1 to 2 minutes.

1. Run the following query:

    ```T-sql
    SET NOCOUNT ON;
    DECLARE @batch INT = 25, @rows INT = 1, @done INT = 0;
    
    WHILE @rows > 0
    BEGIN
        UPDATE TOP (@batch) dbo.ProductEmbeddings
        SET Embedding  = AI_GENERATE_EMBEDDINGS(Description USE MODEL EmbeddingModel),
            EmbeddedAt = SYSUTCDATETIME()
        WHERE Embedding IS NULL;
    
        SET @rows = @@ROWCOUNT;
        SET @done += @rows;
        RAISERROR('%d rows embedded', 0, 1, @done) WITH NOWAIT;
    
        WAITFOR DELAY '00:00:05';   -- stays within the deployment's rate limit
    END;
    ```
    
    If it stops with HTTP 429 (throttling), wait 30 seconds and rerun; it resumes where it stopped. If your deployment capacity is below 50, use +++@batch = 5+++ and +++WAITFOR DELAY '00:00:06'+++. Each row is still one model request, so in production generate embeddings off the request path, for example from a scheduled job, Azure Functions or a Microsoft Fabric pipeline, and run large re-embeds off-peak.
    
    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image16.png)

### Task 2: Validate the load

1. Run the following query:
    
    ```sql
    SELECT COUNT(*)                                     AS total_rows,
           SUM(IIF(Embedding IS NULL, 1, 0))            AS missing,
           MIN(VECTORPROPERTY(Embedding, 'Dimensions')) AS dims
    FROM dbo.ProductEmbeddings;
    ```
    
    **Expected result:** **missing = 0**, **dims = 1536**.
    
    ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image17.png)

### Task 3: Run a semantic search

### Question1: a bike that copes with rough mountain trails

1. Run the following query:

    ```sql
    DECLARE @question NVARCHAR(400) = N'a bike that copes with rough mountain trails';
    DECLARE @query_vector VECTOR(1536);
    EXEC dbo.get_embedding @question, @query_vector OUTPUT;
    
    SELECT TOP (5) ProductID, ProductName, Category, ListPrice,
           CAST(VECTOR_DISTANCE('cosine', Embedding, @query_vector) AS DECIMAL(6,4)) AS distance
    FROM dbo.ProductEmbeddings
    ORDER BY distance;
    ```

1. Rerun with each question below, changing only **@question**, and record the results.

    | Question | Top result | Distance | Relevant? |
    |---|---|---|---|
    | `a bike that copes with rough mountain trails` | | | |
    | `something to keep my hands warm on cold morning rides` | | | |
    | `carry water on a long ride` | | | |
    | `a gift for a child learning to ride` | | | |

   ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image18.png)

### Question2: something to keep my hands warm on cold morning rides

1. Run the following query:

    ```sql
    DECLARE @question NVARCHAR(400) = N'something to keep my hands warm on cold morning rides';
    DECLARE @query_vector VECTOR(1536);
    EXEC dbo.get_embedding @question, @query_vector OUTPUT;
    
    SELECT TOP (5) ProductID, ProductName, Category, ListPrice,
           CAST(VECTOR_DISTANCE('cosine', Embedding, @query_vector) AS DECIMAL(6,4)) AS distance
    FROM dbo.ProductEmbeddings
    ORDER BY distance;
    ```

   ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image19.png)

### Question3: carry water on a long ride

1. Run the following query:

    ```sql
    DECLARE @question NVARCHAR(400) = N'carry water on a long ride';
    DECLARE @query_vector VECTOR(1536);
    EXEC dbo.get_embedding @question, @query_vector OUTPUT;
    
    SELECT TOP (5) ProductID, ProductName, Category, ListPrice,
           CAST(VECTOR_DISTANCE('cosine', Embedding, @query_vector) AS DECIMAL(6,4)) AS distance
    FROM dbo.ProductEmbeddings
    ORDER BY distance;
    ```

   ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image20.png)

### Question4: a gift for a child learning to ride

1. Run the following query:

    ```sql
    DECLARE @question NVARCHAR(400) = N'a gift for a child learning to ride';
    DECLARE @query_vector VECTOR(1536);
    EXEC dbo.get_embedding @question, @query_vector OUTPUT;
    
    SELECT TOP (5) ProductID, ProductName, Category, ListPrice,
           CAST(VECTOR_DISTANCE('cosine', Embedding, @query_vector) AS DECIMAL(6,4)) AS distance
    FROM dbo.ProductEmbeddings
    ORDER BY distance;
    ```

   ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image21.png)

### Task 4: Find similar products

1. Run the following query:

    ```sql
    DECLARE @seed VECTOR(1536) =
        (SELECT TOP (1) Embedding
         FROM dbo.ProductEmbeddings
         WHERE ProductName LIKE N'Mountain-200%'
         ORDER BY ProductID);
    
    SELECT TOP (5) ProductName, Category,
           CAST(VECTOR_DISTANCE('cosine', Embedding, @seed) AS DECIMAL(6,4)) AS distance
    FROM dbo.ProductEmbeddings
    WHERE ProductName NOT LIKE N'Mountain-200%'
    ORDER BY distance;
    ```

   ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image22.png)

## Exercise 3: Perform Hybrid and Semantic Search

### Task 1: Create a full-text index

1. Run the following query:

    ```sql
    CREATE FULLTEXT CATALOG ftc_products AS DEFAULT;
    
    CREATE FULLTEXT INDEX ON dbo.ProductEmbeddings (Description LANGUAGE 1033)
        KEY INDEX PK_ProductEmbeddings
        WITH CHANGE_TRACKING AUTO;
    
    SELECT FULLTEXTCATALOGPROPERTY('ftc_products', 'PopulateStatus') AS populate_status;   -- 0 = done
    ```

   ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image23.png)

### Task 2: Compare keyword and semantic search

1. Run the following query:

    ```sql
    DECLARE @question NVARCHAR(400) = N'Road-650 Red 52';
    
    SELECT TOP (5) p.ProductName, k.RANK AS keyword_rank
    FROM FREETEXTTABLE(dbo.ProductEmbeddings, Description, @question) AS k
    JOIN dbo.ProductEmbeddings p ON p.ProductID = k.[KEY]
    ORDER BY k.RANK DESC;
    
    DECLARE @query_vector VECTOR(1536);
    EXEC dbo.get_embedding @question, @query_vector OUTPUT;
    
    SELECT TOP (5) ProductName,
           CAST(VECTOR_DISTANCE('cosine', Embedding, @query_vector) AS DECIMAL(6,4)) AS distance
    FROM dbo.ProductEmbeddings
    ORDER BY distance;
    ```

    >[!Note] Run it again with +++keep my hands warm on cold rides+++. Keyword search wins on exact model, size and colour; semantic search wins on concepts. Hybrid search combines both.

   ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image24.png)

### Task 3: Filtered semantic search

1. Run the following query:

    ```sql
    DECLARE @query_vector VECTOR(1536);
    EXEC dbo.get_embedding N'comfortable bike for long weekend rides', @query_vector OUTPUT;
    
    SELECT TOP (5) ProductName, Category, ListPrice,
           CAST(VECTOR_DISTANCE('cosine', Embedding, @query_vector) AS DECIMAL(6,4)) AS distance
    FROM dbo.ProductEmbeddings
    WHERE ListPrice < 1000
      AND Category LIKE N'%Bikes'
    ORDER BY distance;
    ```

    Relational predicates narrow the candidates before ranking: fewer distance calculations and more relevant results.

   ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image25.png)

### Task 4: Hybrid search with Reciprocal Rank Fusion

RRF adds 1 / (60 + rank) from each result list, so items ranked well by both methods rise to the top without normalising different score types.

1. Run the following query:
    
    ```sql
    CREATE OR ALTER PROCEDURE dbo.hybrid_search
        @question NVARCHAR(400),
        @top INT = 5
    WITH EXECUTE AS OWNER   -- callers need only EXECUTE permission
    AS
    BEGIN
        SET NOCOUNT ON;
    
        DECLARE @query_vector VECTOR(1536);
        EXEC dbo.get_embedding @question, @query_vector OUTPUT;
    
        WITH vec AS (
            SELECT TOP (50) ProductID,
                   ROW_NUMBER() OVER (ORDER BY VECTOR_DISTANCE('cosine', Embedding, @query_vector)) AS rnk
            FROM dbo.ProductEmbeddings
            ORDER BY VECTOR_DISTANCE('cosine', Embedding, @query_vector)
        ),
        kw AS (
            SELECT TOP (50) k.[KEY] AS ProductID,
                   ROW_NUMBER() OVER (ORDER BY k.RANK DESC) AS rnk
            FROM FREETEXTTABLE(dbo.ProductEmbeddings, Description, @question) AS k
            ORDER BY k.RANK DESC
        )
        SELECT TOP (@top) p.ProductName, p.Category, p.ListPrice,
               v.rnk AS vector_rank,
               w.rnk AS keyword_rank,
               CAST(ISNULL(1.0 / (60 + v.rnk), 0) + ISNULL(1.0 / (60 + w.rnk), 0) AS DECIMAL(8,6)) AS rrf_score
        FROM vec v
        FULL OUTER JOIN kw w ON w.ProductID = v.ProductID
        JOIN dbo.ProductEmbeddings p ON p.ProductID = COALESCE(v.ProductID, w.ProductID)
        ORDER BY rrf_score DESC;
    END;
    GO
    
    GRANT EXECUTE ON dbo.hybrid_search TO AiSearchUsers;
    GO
    
    EXEC dbo.hybrid_search N'Road-650 Red 52';
    EXEC dbo.hybrid_search N'keep my hands warm on cold rides';
    
    -- Governed access: the app can search but cannot read the table directly
    EXECUTE AS USER = 'AssistantApp';
    
    EXEC dbo.hybrid_search N'carry water on a long ride', 3;   -- works: returns 3 products
    
    BEGIN TRY
        SELECT TOP (1) * FROM dbo.ProductEmbeddings;   -- should be blocked
    END TRY
    BEGIN CATCH
        SELECT ERROR_MESSAGE() AS expected_result;     -- "The SELECT permission was denied..."
    END CATCH;
    
    REVERT;
    
    SELECT USER_NAME() AS [who_am_i];   -- should be dbo again
    ```

    Compare **vector_rank** and **keyword_rank**: exact-model queries are led by keyword matches, conceptual queries by vector matches.
    
    **Expected result:** each of the first two searches returns five products with **vector_rank**, **keyword_rank** and **rrf_score**. The governed test returns three products and then a row saying “The SELECT permission was denied…”. That denial is the point of the test: the app can search but cannot read the table. **who_am_i** must show **dbo**; if it shows AssistantApp, run +++REVERT;+++.

   ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image26.png)

### Task 5: Optimize performance for AI and operational workloads

##### 5a. Measure where time is spent

1. Run the following query:

    ```sql
    SET STATISTICS TIME ON;
    
    DECLARE @v VECTOR(1536);
    EXEC dbo.get_embedding N'carry water on a long ride', @v OUTPUT;   -- A: model call
    
    SELECT TOP (5) ProductName
    FROM dbo.ProductEmbeddings
    ORDER BY VECTOR_DISTANCE('cosine', Embedding, @v);                 -- B: vector ranking
    
    SET STATISTICS TIME OFF;
    ```

    In the **Messages** tab, compare elapsed times. The model call (A) dominates; the vector ranking (B) takes a few milliseconds for about 290 rows.

   ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image27.png)

##### 5b. Cache repeated question vectors

1. Run the following query:
    
    ```sql
    CREATE TABLE dbo.QueryEmbeddingCache (
        Question  NVARCHAR(400) NOT NULL PRIMARY KEY,
        Embedding VECTOR(1536)  NOT NULL,
        CachedAt  DATETIME2(0)  NOT NULL DEFAULT SYSUTCDATETIME()
    );
    GO
    
    CREATE OR ALTER PROCEDURE dbo.get_embedding_cached
        @text NVARCHAR(400),
        @embedding VECTOR(1536) OUTPUT
    AS
    BEGIN
        SELECT @embedding = Embedding
        FROM dbo.QueryEmbeddingCache
        WHERE Question = @text;
    
        IF @embedding IS NULL
        BEGIN
            EXEC dbo.get_embedding @text, @embedding OUTPUT;
            INSERT dbo.QueryEmbeddingCache (Question, Embedding) VALUES (@text, @embedding);
        END
    END;
    GO
    
    SET STATISTICS TIME ON;
    
    DECLARE @v VECTOR(1536);
    EXEC dbo.get_embedding_cached N'carry water on a long ride', @v OUTPUT;   -- miss
    EXEC dbo.get_embedding_cached N'carry water on a long ride', @v OUTPUT;   -- hit
    
    SET STATISTICS TIME OFF;
    ```

    The second call skips the model entirely. Popular questions in a shopping assistant repeat often, so this removes most latency and cost.

   ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image28.png)

##### 5c. Find the most expensive queries with Query Store

1. Run the following query:

    ```sql
    EXEC sp_query_store_flush_db;
    
    SELECT TOP (5)
           LEFT(qt.query_sql_text, 70)                                AS query_text,
           SUM(rs.count_executions)                                   AS executions,
           CAST(AVG(rs.avg_duration) / 1000.0 AS DECIMAL(10,1))       AS avg_duration_ms,
           CAST(AVG(rs.avg_cpu_time) / 1000.0 AS DECIMAL(10,1))       AS avg_cpu_ms,
           CAST(AVG(rs.avg_logical_io_reads) AS DECIMAL(10,0))        AS avg_reads
    FROM sys.query_store_query_text qt
    JOIN sys.query_store_query q          ON q.query_text_id = qt.query_text_id
    JOIN sys.query_store_plan p           ON p.query_id = q.query_id
    JOIN sys.query_store_runtime_stats rs ON rs.plan_id = p.plan_id
    GROUP BY qt.query_sql_text
    ORDER BY SUM(rs.avg_cpu_time * rs.count_executions) DESC;
    ```

    Find the query containing **VECTOR_DISTANCE**. Its CPU and reads are small, confirming the SQL side scales well at this size; the cost to watch is the model call and the full scan as the table grows.

   ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image29.png)

##### 5d. Decide on vector indexing and workload isolation

Record your decision for Zava (about 290 rows today, 500,000 support-article chunks planned), and how you would keep AI reads away from order processing (Exercise 1, Task 1).

1. Run the following query:

    ```sql
    -- Preview feature: syntax and limitations may change between releases; skip if it fails
    -- Run CREATE VECTOR INDEX once; rerunning it reports that the index already exists
    CREATE VECTOR INDEX vix_ProductEmbeddings_Embedding
    ON dbo.ProductEmbeddings (Embedding)
    WITH (METRIC = 'cosine', TYPE = 'diskann');
    GO
    
    -- Approximate nearest-neighbour search through the index
    DECLARE @qv VECTOR(1536);
    EXEC dbo.get_embedding_cached N'carry water on a long ride', @qv OUTPUT;
    
    SELECT TOP (5) WITH APPROXIMATE
           t.ProductName,
           CAST(s.distance AS DECIMAL(6,4)) AS distance
    FROM VECTOR_SEARCH(
             TABLE      = dbo.ProductEmbeddings AS t,
             COLUMN     = Embedding,
             SIMILAR_TO = @qv,
             METRIC     = 'cosine'
         ) AS s
    ORDER BY s.distance;
    ```
    
    Newer vector index versions take the number of results from **SELECT TOP (n) WITH APPROXIMATE**; earlier preview releases used **TOP_N** inside **VECTOR_SEARCH** and now return “does not support explicit TOP_N parameter”. If **WITH APPROXIMATE** gives a syntax error on your server, remove those two words. At this size the approximate results should match the exact results from Exercise 2. If your release makes the table read-only while a vector index exists, drop the index before changing data.

   ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image30.png)

### Cleanup

This is the last lab in the series. In Cloud Shell (Bash), delete all resources to stop charges:

**Bash**

```bash
az group delete -n ResourceGroup1 --yes --no-wait
az group exists -n ResourceGroup1   # returns false when the deletion has finished
```

**--no-wait** returns immediately, so **az group exists** keeps returning **true** for a few minutes. Run it again until it returns **false**. The command deletes everything in the group, including the Lab 9 resources, so run it only when you have finished Labs 9 to 11.

**Optional:** turn Microsoft Defender for SQL back to Free only if it was enabled at subscription level. A “Subscription Not Registered” message means it was never enabled and there is nothing to do.

**Bash (optional)**

`az security pricing create -n SqlServers --tier Free`

**Optional:** a deleted Foundry resource keeps its name for a while. To reuse the name in a later lab run, purge it:

**Bash (optional)**

```bash
az cognitiveservices account list-deleted -o table
az cognitiveservices account purge -l eastus -g ResourceGroup1 -n aif-zava-67029880   # use your resource name
```

   ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image31.png)

   ![](https://raw.githubusercontent.com/technofocus-pte/sqlsrvrmgrtnmdrnztnazdepth/refs/heads/main/Lab%20Guides/Lab%2011/media/image32.png)

If you used a shared Foundry resource from your instructor instead of creating your own, ask them to remove your server’s role assignment from it.
