![](https://raw.githubusercontent.com/microsoft/sqlworkshops/master/graphics/microsoftlogo.png)

# RAG Implementation with SQL Database in Fabric​

This module walks through the steps to generate and store vector embeddings for relational data, perform semantic similarity searches using SQL's VECTOR_DISTANCE function. 

## Setting up SQL Database in Fabric for Semantic Retrieval using Vector

## Setup of database credential

A database scoped credential is a record in the database that contains authentication information for connecting to a resource outside the database. For this module, we will be creating one that contains the api key for connecting to Azure OpenAI services.

Open the database that you created in the first module. 

Click New Query button to open a query editor window. Copy and Paste the below code and click Run button.
> [!TIP]
>
> **Below code is used to create a database scoped credential with Azure OpenAI endpoint.**

```SQL

if not exists(SELECT * FROM sys.symmetric_keys WHERE [name] = '##MS_DatabaseMasterKey##')
begin
    CREATE master key encryption by password = N'V3RYStr0NGP@ssw0rd!';
end
go
----Below code is used to create a database scoped credential with Azure OpenAI endpoint

CREATE DATABASE SCOPED CREDENTIAL [https://<yourendpoint>.openai.azure.com/]
    WITH IDENTITY = 'HTTPEndpointHeaders', secret = '{"api-key":"<YourKey>"}';
GO
drop external model SalesLT_AOAI_Embeddings
CREATE EXTERNAL MODEL SalesLT_AOAI_Embeddings
WITH (
    LOCATION = 'https://demovectorinternal.openai.azure.com/openai/deployments/pookamembedding/embeddings?api-version=2023-05-15', --update with your URL from Azure AI foundry
    API_FORMAT = 'Azure OpenAI',
    MODEL_TYPE = EMBEDDINGS,
    MODEL = 'text-embedding-3-small',
    CREDENTIAL = [https://demovectorinternal.openai.azure.com/]
);
 ```

 !["A picture of Azure portal showing the Endpoint to use for the Database Scoped Credential"](../../img/graphics/Introduction/endpoint.png)


## Creating embeddings for relational data

### Understanding embeddings in Azure OpenAI

An embedding is a special format of data representation that machine learning models and algorithms can easily use. The embedding is an information dense representation of the semantic meaning of a piece of text. Each embedding is a vector of floating-point numbers. Vector embeddings can help with semantic search by capturing the semantic similarity between terms. For example, "cat" and "kitty" have similar meanings, even though they are spelled differently. 

Embeddings created and stored in the SQL database in Microsoft Fabric during this module will power a vector similarity search.

### Preparing the database and creating embeddings

This next section of the module will have us alter the  product table to add a new vector type column. we will then use a stored procedure to create embeddings for the products and store the vector arrays in that column.

1. Copy and Paste the below T-SQL code to a new SQL query window and click Run button:

>[!TIP]
> **This code adds a vector datatype as well as chunk column to the Product table. Chunk will store the text we send over to the embeddings REST endpoint.**

```SQL

    ALTER TABLE [SalesLT].[Product]
    ADD  embeddings VECTOR(1536), chunk nvarchar(2000);
```

!["A picture of clicking the run button on the query sheet for adding 2 columns to the product table"](../../img/graphics/2025-01-10_1.30.19_PM.png)


2. Next we will use the AI_GENERATE_EMBEDDINGS , a built-in function that creates embeddings (vector arrays) using the precreated AI model definition stored in the database.
We will be create embeddings for all products in the Products table
> [!TIP]
>
> **Looking at the SQL, the text we are embedding contains the product name, product color (if available), the category name the product belongs to, the model name of the product, and the description of the product.**


Copy and paste the following T-SQL code into a new query window and click Run button.
Note (Fabric SQL editor): The Fabric portal SQL editor performs client-side parsing/validation, and may misinterpret the USE token inside AI_GENERATE_EMBEDDINGS(... USE MODEL ...) as the USE <database> statement. Use the CROSS APPLY pattern (or run in SSMS/VS Code) to avoid the editor parse error.

> [!IMPORTANT]
>
> **This code will take 40 to 90 seconds to run** 

 ```SQL

    UPDATE p
SET
    [chunk] = x.text_to_embed,
    [embeddings] = AI_GENERATE_EMBEDDINGS(x.text_to_embed USE MODEL SalesLT_AOAI_Embeddings)
FROM [SalesLT].[Product] AS p
CROSS APPLY (
    SELECT
        p.Name + N' ' +
        ISNULL(p.Color, N'No Color') + N' ' +
        c.Name + N' ' +
        m.Name + N' ' +
        ISNULL(d.Description, N'') AS text_to_embed
    FROM [SalesLT].[ProductCategory] AS c
    JOIN [SalesLT].[ProductModel]    AS m ON m.ProductModelID = p.ProductModelID
    LEFT JOIN [SalesLT].[vProductAndDescription] AS d
           ON d.ProductID = p.ProductID AND d.Culture = 'en'
    WHERE c.ProductCategoryID = p.ProductCategoryID
) AS x;
```

4. To ensure all the embeddings were created, run the following code in a new query window: 

    ```SQL
    SELECT COUNT(*) FROM SalesLT.Product WHERE embeddings is null;
    ```

    You should get 0 for the result.

5. Run the next query in a new query window to see the results of the above update to the Products table:

> [!TIP]
>
> You can see that the chunk column is the combination of multiple data points about a product and the embeddings column contains the vector arrays.

```SQL

SELECT TOP 10 chunk, embeddings FROM SalesLT.Product
```

!["A picture of the query result showing the chunk and embeddings columns and their data." ](../../img/graphics/2025-01-15_6.34.32_AM.png)

### Vector similarity searching

Vector similarity searching is a technique used to find and retrieve data points that are similar to a given query, based on their vector representations. The similarity between two vectors is usually measured using a distance metric, such as cosine similarity or Euclidean distance. These metrics quantify the similarity between two vectors by calculating the angle between them or the distance between their coordinates in the vector space.

Vector similarity searching has numerous applications, such as recommendation systems, search engines, image and video retrieval, and natural language processing tasks. It allows for efficient and accurate retrieval of similar items, enabling users to find relevant information or discover related items quickly and effectively.

The **VECTOR_DISTANCE** function is a new feature of the SQL Database in fabric that can calculate the distance between two vectors enabling similarity searching right in the database. 

The syntax is as follows:

```SQL-nocopy
VECTOR_DISTANCE ( distance_metric, vector1, vector2 )
```

You will be using this function in samples as well as in the RAG chat application; both utilizing the vectors you just created for the Products table.

1. The first query will pose the question "I am looking for a red bike and I dont want to spend a lot". The key words that should help with our similarity search are red, bike, and dont want to spend a lot. Run the following SQL in a new query window:

```SQL
DECLARE @search_text NVARCHAR(MAX) = N'looking for a red bike and i dont want to spend a lot';
DECLARE @search_vector VECTOR(1536);

SELECT
    @search_vector = AI_GENERATE_EMBEDDINGS(
        @search_text USE MODEL SalesLT_AOAI_Embeddings
    );

SELECT TOP (4)
    p.ProductID,
    p.Name,
    p.chunk,
    VECTOR_DISTANCE(
        'cosine',
        @search_vector,
        p.embeddings
    ) AS distance
FROM [SalesLT].[Product] p
ORDER BY distance;
```
Results show that the search found exactly that, an affordable red bike. The distance column shows us how similar it found the results to be using VECTOR_DISTANCE, with a lower score being a better match.

**Results**

| ProductID | Name | chunk | distance |
|:---------|:---------|:---------|:---------|
| 763 | Road-650 Red, 48 | Road-650 Red, 48 Red Road Bikes Road-650 Value-priced bike with many features of our top-of-the-line models. Has the same light, stiff frame, and the quick acceleration we're famous for. | 0.16352240013483477 |
| 760 | Road-650 Red, 60 | Road-650 Red, 60 Red Road Bikes Road-650 Value-priced bike with many features of our top-of-the-line models. Has the same light, stiff frame, and the quick acceleration we're famous for. | 0.16361482158949225 |
| 759 | Road-650 Red, 58 | Road-650 Red, 58 Red Road Bikes Road-650 Value-priced bike with many features of our top-of-the-line models. Has the same light, stiff frame, and the quick acceleration we're famous for. | 0.16432339626539993 |
| 762 | Road-650 Red, 44 | Road-650 Red, 44 Red Road Bikes Road-650 Value-priced bike with many features of our top-of-the-line models. Has the same light, stiff frame, and the quick acceleration we're famous for. | 0.1652894865541471 |
!["A picture of running Query 1 and getting results outlined in the Query 1 results table." ](../../img/graphics/2025-01-15_6.36.01_AM.png)

2. In the previous example, we were clear on what we were looking for; cheap red bike. In this next example, you are going to have the search flex its AI muscles a bit by saying we want a bike seat that needs to be good on trails. This will require the search to look for adjacent values that have something in common with trails. 
Run the below SQL in a new query window.

    
```SQL
DECLARE @search_text nvarchar(max) = N'Do you sell any padded seats that are good on trails?';
DECLARE @search_vector vector(1536);
SELECT @search_vector = x.embeddings
FROM (
    SELECT [embeddings] = AI_GENERATE_EMBEDDINGS(@search_text USE MODEL SalesLT_AOAI_Embeddings)
) AS x;

-- If embedding generation fails, bail early
IF (@search_vector IS NULL)
    RETURN;

SELECT TOP (4)
    p.ProductID,
    p.Name,
    p.chunk,
    vector_distance('cosine', @search_vector, p.embeddings) AS distance
FROM [SalesLT].[Product] AS p
WHERE p.embeddings IS NOT NULL
ORDER BY distance;
```

These results are very interesting for it found products based on word meanings such as absorb shocks and bumps and foam-padded. It was able to make connections to riding conditions on trails and find products that would fit that need.

**Search Results**

| Name | chunk | distance |
|:---------|:---------|:---------|
| ML Mountain Seat/Saddle | ML Mountain Seat/Saddle No Color Saddles ML Mountain Seat/Saddle 2 Designed to absorb shock. | 0.17265341238606102 |
| LL Road Seat/Saddle | LL Road Seat/Saddle No Color Saddles LL Road Seat/Saddle 1 Lightweight foam-padded saddle. | 0.17667274723850412 |
| ML Road Seat/Saddle | ML Road Seat/Saddle No Color Saddles ML Road Seat/Saddle 2 Rubber bumpers absorb bumps. | 0.18802953111711573 |
| HL Mountain Seat/Saddle | HL Mountain Seat/Saddle No Color Saddles HL Mountain Seat/Saddle 2 Anatomic design for a full-day of riding in comfort. Durable leather. | 0.18931317298732764 |
!["A picture of running Query 3 and getting results outlined in the Query 3 results table"](../../img/graphics/2025-01-15_6.38.06_AM.png)

3. Create a new stored procedure to find products. Copy and Paste the below code in a new query window and click Run:

```SQL
    CREATE OR ALTER PROCEDURE [SalesLT].[find_products]
    @text nvarchar(max),
    @top int = 10,
    @min_similarity decimal(19,16) = 0.50
AS
BEGIN
    SET NOCOUNT ON;

    IF (@text IS NULL OR LTRIM(RTRIM(@text)) = N'')
        RETURN;

    DECLARE @qv vector(1536);
    SET @qv = AI_GENERATE_EMBEDDINGS(@text USE MODEL SalesLT_AOAI_Embeddings);
    IF (@qv IS NULL)
        RETURN;

    WITH vector_results AS (
        SELECT
            p.Name AS product_name,
            ISNULL(p.Color, 'No Color') AS product_color,
            c.Name AS category_name,
            m.Name AS model_name,
            d.Description AS product_description,
            p.ListPrice AS list_price,
            p.Weight AS product_weight,
            VECTOR_DISTANCE('cosine', @qv, p.embeddings) AS distance
        FROM
            [SalesLT].[Product] p
            JOIN [SalesLT].[vProductAndDescription] d
                ON p.ProductID = d.ProductID
               AND d.Culture = 'en'
            JOIN [SalesLT].[ProductCategory] c
                ON p.ProductCategoryID = c.ProductCategoryID
            JOIN [SalesLT].[ProductModel] m
                ON p.ProductModelID = m.ProductModelID
        WHERE
            p.embeddings IS NOT NULL
    )
    SELECT TOP (@top)
        product_name,
        product_color,
        category_name,
        model_name,
        product_description,
        list_price,
        product_weight,
        distance
    FROM vector_results
    WHERE (1 - distance) > @min_similarity
    ORDER BY distance ASC;
END;
GO
```

4. Next, you need to encapsulate the **STORED PROCEDURE** into a wrapper so that the result set can be utilized by our GraphQL endpoint. Using the **WITH RESULT SET** syntax allows you to change the names and data types of the returning result set. This is needed in this example because the usage of sp_invoke_external_rest_endpoint and the return output from extended stored procedures 

Copy and Paste the below code in a new query window and click Run:  

```SQL
CREATE OR ALTER PROCEDURE [SalesLT].[find_products_api]
    @text nvarchar(max),
    @top int = 10,
    @min_similarity decimal(19,16) = 0.50
AS
BEGIN
    SET NOCOUNT ON;

    EXEC [SalesLT].[find_products]
        @text = @text,
        @top = @top,
        @min_similarity = @min_similarity
    WITH RESULT SETS
    (
        (
            [product_name]         nvarchar(4000),
            [product_color]        nvarchar(4000),
            [category_name]        nvarchar(4000),
            [model_name]           nvarchar(4000),
            [product_description]  nvarchar(max),
            [list_price]           decimal(19,4),
            [product_weight]       decimal(19,4),
            [distance]             float
        )
    );
END;
GO
```

5. Let us test this newly created procedure to see the results by running the following SQL in a new query window:

```SQL
exec SalesLT.find_products_api 'I am looking for a red bike'
```
!["A picture of running the find_products_api stored procedure"](../../img/graphics/2025-01-14_6.57.09_AM.png)

Congratulations! In this module, you learned how to build a RAG application using SQL database in fabric, and Azure OpenAI. You explored generating vector embeddings for relational data, performing semantic similarity searches with SQL.

In the next module we would explore natural language responses via GPT-4.1 and create GraphQL API to retrieve data.
