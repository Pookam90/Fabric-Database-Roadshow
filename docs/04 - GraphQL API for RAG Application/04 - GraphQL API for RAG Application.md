![](https://raw.githubusercontent.com/microsoft/sqlworkshops/master/graphics/microsoftlogo.png)

# Build a GraphQL API for RAG applications

In this section of the lab, you will be deploying a GraphQL API that uses embeddings, vector similarity search, and relational data to return a set of products that could be used by a chat application leveraging a Large Language Model (LLM).

In this section, we will create a stored procedure that will be used by the GraphQL API for taking in questions and returning products.

## Chat completion 
Let's create a new stored procedure to create a new flow that not only uses vector similarity search to get products based on a question asked by a user, but to take the results, pass them to Azure OpenAI Chat Completion, and craft an answer they would typically see with an AI chat application.

1. The first step in augmenting our RAG application API is to create a stored procedure that takes the retrieved products and passes them in a prompt to an Azure OpenAI Chat Completion REST endpoint. The prompt consists of telling the endpoint who they are, what products they have to work with, and the exact question that was asked by the user. 

   Copy/Paste the below T-SQL Code in a new query window and Run the code:


```SQL
    

    CREATE OR ALTER PROCEDURE [SalesLT].[prompt_answer]
    @user_question nvarchar(max),
    @products      nvarchar(max),
    @answer        nvarchar(max) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    IF (@user_question IS NULL OR LTRIM(RTRIM(@user_question)) = N'') RETURN;

    -- Avoid NULL concatenation 
    SET @products = COALESCE(@products, N'');

    DECLARE @payload  nvarchar(max),
            @ret      int,
            @response nvarchar(max);

    -- Escape user-provided text to keep JSON valid
    DECLARE @q nvarchar(max) = STRING_ESCAPE(@user_question, 'json');
    DECLARE @p nvarchar(max) = STRING_ESCAPE(@products, 'json');

    SET @payload = N'{
      "messages": [
        { "role": "system", "content": "You are a sales assistant who helps customers find the right products for their question and activities." },
        { "role": "user", "content": "The products available are the following: ' + @p + N'" },
        { "role": "user", "content": "' + @q + N'" }
      ]
    }';

    EXEC @ret = sp_invoke_external_rest_endpoint
        @url        = N'https://demovectorinternal.openai.azure.com/openai/deployments/chatcompletion/chat/completions?api-version=2025-01-01-preview',
        @method     = N'POST',
        @payload    = @payload,
        @credential = N'https://demovectorinternal.openai.azure.com/',
        @timeout    = 230,
        @response   = @response OUTPUT;

    IF (@ret = 0)
        SET @answer = JSON_VALUE(@response, '$.result.choices[0].message.content');
END
GO

```

2. Now that you have created the chat completion stored procedure, we need to create a new find_products_chat stored procedure that adds a call to this chat completion endpoint.  

    Copy/Paste the below T-SQL Code in a new query window and Run the code:


    ```SQL
    CREATE OR ALTER PROCEDURE [SalesLT].[find_products_chat]
  @text nvarchar(max),
  @top int = 3,
  @min_similarity decimal(19,16) = 0.50
AS
BEGIN
  SET NOCOUNT ON;
  IF (@text IS NULL OR LTRIM(RTRIM(@text)) = N'') RETURN;

  DECLARE @qv vector(1536),
          @products_text nvarchar(max),
          @answer nvarchar(max);

  -- Inline embedding generation 
  SELECT @qv = AI_GENERATE_EMBEDDINGS(@text USE MODEL [SalesLT_AOAI_Embeddings]); 
  IF (@qv IS NULL) RETURN;

  ;WITH vector_results AS (
    SELECT
      p.Name AS product_name,
      ISNULL(p.Color,'No Color') AS product_color,
      c.Name AS category_name,
      m.Name AS model_name,
      d.Description AS product_description,
      p.ListPrice AS list_price,
      p.Weight AS product_weight,
      VECTOR_DISTANCE('cosine', @qv, p.embeddings) AS distance
    FROM [SalesLT].[Product] p
    JOIN [SalesLT].[vProductAndDescription] d ON p.ProductID = d.ProductID
    JOIN [SalesLT].[ProductCategory] c ON p.ProductCategoryID = c.ProductCategoryID
    JOIN [SalesLT].[ProductModel] m ON p.ProductModelID = m.ProductModelID
    WHERE d.Culture = 'en'
      AND p.embeddings IS NOT NULL
  ),
  topk AS (
    SELECT TOP (@top) *
    FROM vector_results
    WHERE (1 - distance) > @min_similarity
    ORDER BY distance ASC
  )
  SELECT @products_text =
    COALESCE(
      STRING_AGG(
        CONCAT(
          'Name: ', product_name,
          ', Color: ', product_color,
          ', Category: ', category_name,
          ', Model: ', model_name,
          ', Price: ', CONVERT(nvarchar(40), list_price),
          ', Weight: ', CONVERT(nvarchar(40), product_weight),
          ', Similarity: ', CONVERT(nvarchar(40), (1 - distance)),
          ', Description: ', COALESCE(product_description,'')
        ),
        CHAR(10)
      ),
      N'No matching products found.'
    )
  FROM topk;

  EXEC [SalesLT].[prompt_answer] @text, @products_text, @answer OUTPUT;

  SELECT @answer AS [answer];
END
GO
    ```

3. The last step before we can create a **GraphQL** endpoint is to wrap the new find products chat stored procedure.

    Copy/Paste the below T-SQL Code in a new query window and Run the code:

```SQL
 CREATE OR ALTER PROCEDURE SalesLT.[find_products_chat_api]
    @text NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    EXEC SalesLT.find_products_chat @text
    WITH RESULT SETS
    (
        (
            answer NVARCHAR(MAX)
        )
    );
END
GO
```

4. You can test this new  procedure to see how Azure OpenAI will answer a question with product data.
    Copy/Paste the below T-SQL Code in a new query window and Run the code:

    ```SQL
    exec SalesLT.find_products_chat_api 'I am looking for a red bike'
    ```
> [!TIP]
>
> Above execution will result in this answer: **"It sounds like the Road-650 Red, 62 Red Road Bikes Road-650 would be an excellent choice for you. This value-priced bike comes in red and features a light, stiff frame that is known for its quick acceleration. It also incorporates many features from top-of-the-line models. Would you like more details about this bike or help with anything else?"** . Note: Your answer could be different.
    !["A picture of running the find_products_chat_api stored procedure"](../../img/graphics/2025-01-17_6.15.05_AM.png)
  
## Create GraphQL API

1. To create the GraphQL API, click on the **New API for GraphQL** button on the toolbar just as you did previously.
    !["A picture of clicking on the New API for GraphQL button on the toolbar"](../../img/graphics/2025-01-15_6.52.37_AM.png)
   

1. In the **New API for GraphQL** dialog box, use the **Name Field** and name the API **find_products_chat_api**.
After naming the API, click the **green Create button**.
    !["A picture of clicking the green Create button in the New API for GraphQL dialog box"](../../img/graphics/2025-01-17_7.34.47_AM.png)
  

1.  In the next dialog box use the **Search box** in the **Explorer section** on the left and enter in **find_products_chat_api**.
    !["A picture of enter in find_products_chat_api in the search box"](../../img/graphics/2025-01-17_6.24.33_AM.png)
   

1. Choose the stored procedure in the results. You can ensure it is the **find_products_chat_api** stored procedure by hovering over it with your mouse/pointer. It will also indicate the selected database item in the preview section. It should state **"Preview data: SalesLT.find_products_chat_api"**.
    !["A picture of choosing the find_products_chat_api stored procedure in the results"](SearchStoredProcedureNLoad_cht_api.png)
    

Once you have selected the **find_products_chat_api stored procedure**, click the **green Load button** on the bottom right of the modal dialog box.
   

1. You will now be on the **GraphQL Query editor page**. Copy/Paste the below code in the GraphQL query editor.

    ```GraphQL
    query {
        executefind_products_chat_api(text: "I am looking for padded seats that are good on trails") {
                answer
        }
    }
    ```

    !["A picture of replacing the sample code on the left side of the GraphQL query editor with the supplied code"](../../img/graphics/2025-01-17_6.40.03_AM.png)
   
1. Now, **click the Run button** in the upper left of the GraphQL query editor.
    !["A picture of clicking the Run button in the upper left of the GraphQL query editor"](../../img/graphics/2025-01-17_6.37.51_AM.png)
     

1. And you can review the response in the **Results** section of the editor
    !["A picture of reviewing the response in the Results section of the editor"](../../img/graphics/2025-01-17_6.41.24_AM.png)
    

1. Copy the following code in the the GraphQL editor and see what answer the chat completion endpoint provides!

    ```GraphQL
    query {
        executefind_products_chat_api(text: "Do you have any racing shorts?") {
                answer
        }
    }
    ```


The API you just created could now be handed off to an application developer to be included in a RAG application that uses vector similarity search and data from the database.

Here is a fully functioning Chat application for you to try
[Chat App](https://fabcon-euro.azurewebsites.net/)

Congratulations!! In this module, you  learned how to build a RAG application using SQL database in fabric, and Azure OpenAI. You explored generating vector embeddings for relational data, performing semantic similarity searches with SQL, and integrating natural language responses via GPT-4.1.
