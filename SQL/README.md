# Pragmatic SQL: tips to improve your practice and wellbeing

Whether you are data analyst, data engineer, data scientist or just a data enthusiast, you might find here tips to make your work with SQL more efficient and focus on what matters instead of drowning in unmaintainable code.

Code sections with header “EXECUTABLE” contain sample data and can be copied and tested in your setup. Sections with header “MOCKUP” present a dummy concept and can’t be executed without providing references to tables and columns that exist in your database environment.

DISCLAIMER: The examples presented were prepared using Postgres 9.4 but the majority of concepts are transferable to other databases. Just be aware that slightly different SQL syntax might be required.

## Code formatting and structure
Always try to write code good enough to go directly to production. Even when you are doing just some one-off analysis. It is quite likely that you will need to revisit the code in some time, share it, fix it or hand it over to somebody else.

Use indentations and code sectioning via CTEs to make your code easy to read and navigate (if you are not familiar with CTEs, read further).
Avoid using SELECT * to reduce the load imposed by collecting unnecessary columns.
When joining multiple tables, always explicitly refer to the source table names or aliases.
When defining table aliases and CTEs, use descriptive names that outline the data context or use short codes based on the table name itself (i.e. tb_sales_weekly -> sw). Using a,b,c…x or some random names will not help you much when you will need to fix your code after 2 weeks and no time to spare.
Remove code that is commented out and redundant. It just drains your attention and nobody knows what it was supposed to do anyway. Use a version control system like GIT instead if you are worried about losing some legacy gems.
Organize tables used in FROM and JOIN statements so that you start with small tables in terms of row count (e.g. dimensional tables) and then join the larger ones (e.g. transactional tables). You might notice some nice performance gains.

```
-- MOCKUP ------------------------
SELECT 
  st.id
  ,mt.column_a
  ,lt.column_b
  ...
FROM small_table AS st
LEFT JOIN medium_table AS mt
  ON st.id = mt.id
LEFT JOIN large_table AS lt
  ON st.id = lt.id
```

The small fraction of time you spend formatting your code will in return give you the possibility to rethink what you are doing, uncover patterns you might have missed and maybe even enlighten you with some new perspective.

If your code is clean and clear, reasonably commented and its logic easy to understand, you can save a great amount of time and frustration for your future self and your collaborators. You will also be able to reuse chunks of your code across different use cases.

Practice makes perfect, try to get used to it. Even an odd structure is usually better than none in the long run. And in the end, structure is what SQL stands for.

## 1=1
Does your query contain several WHERE conditions? Do you need to evaluate these conditions individually?

```
-- MOCKUP ------------------------
SELECT 
  ...
FROM table_a
WHERE column_a = 1 -- condition_1
  AND column_b > 5 -- condition_2
```

Checking condition_1 is fairly straight-forward, we can just comment out the line with condition_2. Checking condition_2 is however much more complicated. We would need to modify the WHERE clause, check it and then rewrite the code back.

This might sound as quite an odd tip but it can definitely help you with this nuisance. 1=1.

```
-- MOCKUP ------------------------
SELECT 
  ...
FROM table_a
WHERE 1=1
  AND column_a = 1 -- condition_1
  AND column_b > 5 -- condition_2
```

Adding “1=1” as the first condition of a WHERE clause evaluates as TRUE and logically it does not affect the clause. Yet, it can save you a lot of mundane rewriting of WHERE and AND.

If you are hesitant to keep that in your code, use it only when you need to play around with the WHERE conditions.

## Leading commas
Using leading commas might help during data exploration when you comment out column references to focus only on a particular set of columns. Writing your query with leading commas will prevent the syntax error you get when there is a comma left before FROM statement.

```
-- MOCKUP ------------------------
-- using trailing commas
SELECT 
  column_a,
  column_b, -- this comma causes a syntax error
  --column_c,
  --column_d,
FROM table_a

-- MOCKUP ------------------------
-- using leading commas
SELECT 
  column_a
  ,column_b
  --,column_c
  --,column_d
FROM table_a
```

## Checking DISTINCT values
If you need to check unique values in a column, this is the typical approach:
```
-- MOCKUP ------------------------
SELECT DISTINCT column_a
FROM table_a
```
The approach is valid but might fall short in providing very useful information. How often do these values appear?
```
-- MOCKUP ------------------------
SELECT column_a, COUNT(1)
FROM table_a
GROUP BY column_a
```
The proposed query is a bit longer but the insight it provides might save you some time trying to fix cases that appear 2–3 times in a bazillion of rows and could eventually be ignored.

## CTEs
CTEs (common table expressions) are named temporary result sets you can refer to in your code in the same way as regular tables or views. They substantially improve the readability of your code and might even help the query performance.
```
-- MOCKUP ------------------------
-- using subquery
SELECT 
  st.id
  ,sls.sales_total
FROM stores AS st
LEFT JOIN (
  -- subquery -------------------
  SELECT 
    id, 
    SUM(amount) AS sales_total
  FROM sales
  GROUP BY 
    id
  -------------------------------
  ) AS sls
  ON st.id= sls.id
```
```
-- MOCKUP ------------------------
-- using CTE
WITH
-- CTE -------------------------
sales AS (
  SELECT 
    id, 
    SUM(amount) AS sales_total
  FROM sales
  GROUP BY 
    id
)
---------------------------------
SELECT 
  st.id
  ,sls.sales_total
FROM stores AS st
LEFT JOIN sales AS sls
  ON st.id= sls.id
```
Use CTEs to isolate processing steps, logically section your code, remap values, generate sample data, and/or preprocess data before joining. If possible, try to organize your CTEs to follow some logical flow of information so that you don’t need to jump up and down between code sections as you would need to do with subqueries. Also, name your CTEs meaningfully to help you navigate your code seamlessly.

## Sample data
Do you need to test a new approach or practice some new concept? Do you have some particular values you want to test in mind but don’t know where/how to find these examples in your data? Let’s build up on the CTE concept.
```
-- EXECUTABLE --------------------
WITH
-- creates CTE "sample" with columns "id" and "value"
sample(id, value) AS (
  VALUES
    -- comma-separated list of row values
     ('A', 12)
    ,('B', 3)
    ,('C', NULL)
)
SELECT 
  id
  ,value
FROM sample
```
## Do some tests
Are you writing a custom function, e.g. to process a text column that performs some cleaning or data transformation? Do you need to do some tests on edge cases to be sure it works correctly? Extend the sample data CTE concept.
```
-- MOCKUP ------------------------
WITH
test_cases(input_value, output_value_expected) AS (
  VALUES
     (' A,,b,  c   ', 'a,b,c')
    ,('', NULL)
    ,('C', 'c')
    ,('Dd,  d', 'd')
)
SELECT
  input_value
  ,output_value_expected
  ,test_function(input_value) AS output_value
  ,(test_function(input_value) = output_value_expected) AS test_case_passed
FROM test_cases
```
This can get particularly handy if you are working with REGULAR EXPRESSIONS.

## NULLs and empty strings
NULL represents a missing value. It is not a value itself. NULL is also somehow contagious, wherever it enters, it usually results in NULL. In addition, NULL is neither equal nor unequal to NULL. This might sound unintuitive and its implications could cause syntax error (good scenario) or data quality issues (bad scenario).

Be cautious also when ordering by columns that contain NULL. As NULLs might be considered larger in value than any proper non-NULL values, it can give you some unexpected results. Check your database documentation or do a test. In Postgres, the NULL ordering behavior can be adjusted with the following:
```
-- MOCKUP ------------------------
ORDER BY column_with_nulls DESC NULLS LAST -- NULLs will be at the bottom
ORDER BY column_with_nulls DESC NULLS FIRST -- NULLs will be at the top
```
In CHARACTER type columns (TEXT, VARCHAR, NVARCHAR) you might have encountered a situation where some of them are NULL and some might appear empty without the NULL keyword. Those are likely empty strings, i.e. quotes with no characters inside. Empty strings can enter data directly from source systems or due to some data cleaning and transformations on their way to you. And it might happen that NULLs and empty strings are not used consistently in a single table or column.

If you need to normalize such a field to either NULL or empty string you can use COALESCE or NULLIF functions.
```
-- EXECUTABLE --------------------
WITH
test_cases(inconsistent_column) AS (
  VALUES
     ('A')
    ,(NULL)
    ,('b')
    ,('')
)
SELECT *
    ---------------------------------------------
    -- NORMALIZING MISSING VALUES
    ---------------------------------------------
    -- COALESCE replaces NULL with '' (empty string)
    ,COALESCE(inconsistent_column, '')      AS normalize_empty
    -- NULLIF assigns NULL if the value is '' (empty string)
    ,NULLIF(inconsistent_column, '')        AS normalize_null
    ---------------------------------------------
    -- FILTERING CONDITIONS
    -- only rows evaluated as TRUE are returned
    ---------------------------------------------
    -- filtering out NULLS
    ,(NULLIF(inconsistent_column, '') IS NOT NULL)      AS filter_not_null
    -- filtering out empty strings
    ,(COALESCE(inconsistent_column, '') != '')          AS filter_not_empty
    -- filtering out 'a'
    ,(inconsistent_column != 'a')                       AS filter_not_equal
    -- filtering out 'a','b' and 'C'
    ,(inconsistent_column not in ('a','b','C'))         AS filter_not_in
FROM test_cases
```
## Arrays
To keep it light, arrays can be thought of as lists of values of the same data type. You can use them to simplify logical conditions and make your code more readable and easier to maintain.
```
-- EXECUTABLE --------------------
-- defining array with TEXT type elements 'a','b','c','d'
SELECT 
  -- defining a TEXT array column
  '{a,b,c,d}'::TEXT[]                AS option_1
  ,ARRAY['a','b','c','d']::TEXT[]    AS option_2
  -- creating array from a string with elements separated by ','
  ,string_to_array('a,b,c,d', ',')   AS option_3
```
Plenty of functions and operators are available for arrays but let’s start small. Here are some I use most frequently in WHERE, CASE WHEN or JOIN statements.
```
-- EXECUTABLE --------------------
-- defining array with TEXT type elements 'a','b','c','d'
WITH
test_cases(column_a, column_b, column_c) AS (
  VALUES
     ('A','A','A')
    ,('B',NULL,'b')
    ,('A','a','')
    ,('NA', 'd', 'a')
)
SELECT *
  -- TRUE if any item of the array hold value 'A'
  ,('A' = ANY(ARRAY[column_a, column_b, column_c])) AS test_any
  -- TRUE if all items of the array hold value 'A'
  ,('A' = ALL(ARRAY[column_a, column_b, column_c])) AS test_all
  -- TRUE if at least one value exists in both arrays (intersection)
  ,(ARRAY['NA', '', NULL] 
    && ARRAY[column_a, column_b, column_c]) AS test_instersection
FROM test_cases
```
And another very useful one - UNNEST. If you need to transform a set of columns or elements from an array into individual rows, UNNEST can help.
```
-- EXECUTABLE --------------------
WITH
id_tags(id,tags) AS (
  VALUES
     ('A', 'a,alpha,A')
    ,('B', 'b,beta,B')
    ,('C', NULL)
)
SELECT
  i.id
  ,t.tags
FROM id_tags AS i
LEFT JOIN UNNEST(string_to_array(tags, ',')) AS t(tags) 
  ON TRUE
```
There is also another way to use UNNEST but be careful with this one.
```
-- EXECUTABLE --------------------
WITH
id_tags(id,tags) AS (
  VALUES
     ('A', 'a,alpha,A')
    ,('B', 'b,beta,B')
    ,('C', NULL)
)
SELECT
  id
  ,UNNEST(string_to_array(tags, ',')) AS tags
FROM id_tags
```
Using this approach, if you are unnesting arrays that contain NULL, the rows where unnest would return NULL will not be generated! Also, if you would want to unnest more than one array at the same time, be particularly cautious. If these arrays are of various lengths, it will generate the rows as a cartesian product of all the array elements (all combinations of array elements), whereas if all the arrays are of the same length, unnest will generate rows with index-wise coupling of the elements.
```
-- EXECUTABLE --------------------
-- UNNEST returns cartesian product
-- NULL-containing row is not unnested 
WITH
id_tags(id,tags_1, tags_2) AS (
  VALUES
     ('A', '{a,alpha,A}'::TEXT[], '{1,2}'::TEXT[])
    ,('B', '{b,beta,B}'::TEXT[], '{4,5}'::TEXT[])
    ,('C', NULL::TEXT[], '{1}'::TEXT[])
)
SELECT
  id
  ,UNNEST(tags_1) AS tags_1
  ,UNNEST(tags_2) AS tags_2
FROM id_tags
```
```
-- EXECUTABLE --------------------
-- UNNEST returns index-wise coupling of the elements
-- NULL-containing row is not unnested
WITH
id_tags(id,tags_1, tags_2) AS (
  VALUES
     ('A', '{a,alpha,A}'::TEXT[], '{1,2,3}'::TEXT[])
    ,('B', '{b,beta,B}'::TEXT[], '{4,5,6}'::TEXT[])
    ,('C', NULL::TEXT[], '{1}'::TEXT[])
)
SELECT
  id
  ,UNNEST(tags_1) AS tags_1
  ,UNNEST(tags_2) AS tags_2
FROM id_tags
```
## Value remapping
How many times have you seen something like this repeated several times within a single script?
```
-- MOCKUP ------------------------
CASE 
  WHEN category in ('A', 'B') THEN 1
  WHEN category in ('C', 'D') THEN 2
  ELSE 3
END AS priority
```
Updating these statements is annoying and it might lead to errors if you do not update all the affected statements. We can do it in a better way using a CTE.
```
-- EXECUTABLE --------------------
-- using ARRAYs
WITH
mapping(categories, priority) AS (
  VALUES
     ('{A,B}'::TEXT[], 1)
    ,('{C,D}'::TEXT[], 2)
)
SELECT 
  cc.category
  -- COALESCE is used to supply value 3 where the mapping is not defined
  ,COALESCE(m.priority, 3) AS priority
FROM category_classification AS cc
LEFT JOIN mapping AS m
  ON cc.category = ANY(m.categories)
```
CTE “mapping” creates a result set (just like a table) with columns “categories” and “priority”. Column “categories” contains an ARRAY of values that we want to use to map the “priority”. If arrays make you somehow uncomfortable (as it did to me some time ago), you can use single values and define multiple rows for each “priority” instead.
```
-- EXECUTABLE --------------------
-- using single values
WITH
mapping(categories, priority) AS (
  VALUES
     ('A', 1)
    ,('B', 1)
    ,('C', 2)
    ,('D', 2)
)
SELECT 
  cc.category
  -- COALESCE is used to supply value 3 where the mapping is not defined
  ,COALESCE(m.priority, 3) AS priority
FROM category_classification AS cc
LEFT JOIN mapping AS m
  ON cc.category = m.categories
```
In case you would need to do this in several scripts, consider creating a proper table to store the mapping so that it can be referenced wherever needed by your team or organization.

## Start small
When working on new scripts or fixing the broken ones, you usually don’t want to spend too much time waiting for the results. You already know you will need to run the code several times to capture any remaining bugs.

If you are using CTEs, you can subset your code quite easily, reduce the time to get the results and reiterate with incremental improvements.
```
-- MOCKUP ------------------------
WITH
stores AS (
  SELECT 
    id
    ,name
  FROM tb_stores
  WHERE 1=1
    -- filter some particular items
    AND country in ('123','223')
  -- limit the number of row the CTE returns
  LIMIT 100
)
SELECT 
  s.*
  ,o.product_id
  ,o.quantity
FROM stores AS s
LEFT JOIN tb_orders AS o
  ON s.store_id = c.id
```
## Complex conditions
Do you know this one? Familiar?
```
-- MOCKUP ------------------------
WHERE column_a = 1 
  OR column_b = 1 
  OR column_c = 1 
  OR column_d = 1 
  OR column_e = 1
```
We can do better here simply by “inverting” the IN operator.
```
-- MOCKUP ------------------------
WHERE 1=1
  AND 1 in (
     column_a
    ,column_b
    ,column_c
    ,column_d
    ,column_e
  )
```
What about this other one?
```
-- MOCKUP ------------------------
-- ILIKE is case insensitive alternative to LIKE operator
WHERE column_a ILIKE '%url%' 
  OR column_a ILIKE '%domain%'
  OR column_a ILIKE '%http%'
  OR column_a ILIKE '%site%'
  OR column_a ILIKE '%web%'
```
Here we can use ARRAY and list all the conditions in one line.
```
-- MOCKUP ------------------------
WHERE 1=1
  AND column_a ILIKE ANY(
    ARRAY ['%url%','%domain%','%http%','%site%','%web%']
    )
```
If the filtering conditions are very complex and you need to handle a lot of exceptions, consider using CTE as in the value remapping example. Define your conditions in a tabular form via CTE and then use a JOIN to apply the filtering logic. Get creative, might be worth the time!

## Pivoting data to wide format
Data comes in various shapes and sizes but almost never in that one you need right now. Pivoting in Postgres needs a bit of creativity.
```
-- EXECUTABLE --------------------
WITH
store_sales(product_id,store_id,sales) AS (
  -- sample data
  VALUES
     ('abc', 'a', 10)
    ,('abc', 'b', 2)
    ,('abc', 'c', 15)
    ,('abc', 'a', 8)
    ,('def', 'b', 7)
    ,('def', 'c', 3)
)
SELECT
  product_id
  ,SUM(sales) FILTER (WHERE store_id = 'a')   AS sales_a
  ,SUM(sales) FILTER (WHERE store_id = 'b')   AS sales_b
  ,SUM(sales) FILTER (WHERE store_id = 'c')   AS sales_c
FROM store_sales
GROUP BY
  product_id
```
If your database does not support the FILTER clause, you can use the following.
```
-- EXECUTABLE --------------------
WITH
store_sales(product_id,store_id,sales) AS (
  -- sample data
  VALUES
     ('abc', 'a', 10)
    ,('abc', 'b', 2)
    ,('abc', 'c', 15)
    ,('abc', 'a', 8)
    ,('def', 'b', 7)
    ,('def', 'c', 3)
)
SELECT
    product_id
    ,SUM(CASE WHEN store_id='a' THEN sales END) AS sales_a
    ,SUM(CASE WHEN store_id='b' THEN sales END) AS sales_b
    ,SUM(CASE WHEN store_id='c' THEN sales END) AS sales_c
FROM store_sales
GROUP BY
    product_id
```
If you are using MS-SQL, there is a specific PIVOT statement available for you.

## Unpivoting data to long format
As mentioned before, data comes in various shapes and sizes…
```
-- EXECUTABLE --------------------
WITH
tb_summary(product_id,sales_a,sales_b,sales_c) AS (
  -- sample data
  VALUES
    ('abc', 10, 2, 15),
    ('def', 8, 7, 3),
    ('ghi', 3, 2, 6)
)
SELECT
  product_id
  ,UNNEST(ARRAY[
     'a'
    ,'b'
    ,'c'
    ]) AS store_id
  ,UNNEST(ARRAY[
     sales_a
    ,sales_b
    ,sales_c
    ]) AS sales
FROM tb_summary
```
Note that both arrays are of the same length and therefore will return index-wise pairs of the array elements.

If you are using MS-SQL, there is a specific UNPIVOT statement to do this.

## Analyzing duplicates
Duplicate values might appear due to data quality issues. Quite often though, we might only not understand enough the context of the data and overlook some valuable or even critical information, such as active/inactive record flags or that the “duplicate” values might represent different time periods. Checking for duplicates is an important practice for understanding our data and its quality.

There are various ways to check for duplicates but they typically fall short right after we find out that “yes, there are duplicates”. While it is nice to know there are duplicates, even nicer would be to know what is causing it.
```
-- MOCKUP ------------------------
WITH
duplicate_counts AS (
  -- partition by key(s) you expect to be unique
  SELECT *, COUNT(1) OVER (PARTITION BY key_1, key_2) AS rc
  FROM source_table
)
,total AS (
  -- counts total number of rows and number of duplicate values
  SELECT 
    COUNT(1) AS count_total
    ,COUNT(1) FILTER (WHERE rc > 1) AS count_duplicates
  FROM duplicate_counts
)
,sample AS (
  -- returns sample of duplicated values
  SELECT *
  FROM duplicate_counts
  WHERE rc > 1
  LIMIT 100
)
-- comment/uncomment to get the information you need
SELECT * FROM total;
-- SELECT * FROM sample;
```
The query above might seem unnecessarily complicated at first but let me explain. The reason is to keep the query generic, reusable and pragmatic. You only need to modify the CTE “duplicate_counts” with your table and keys to get information about the total count of rows and the count of duplicates with respect to those keys. Right after that, if duplicates are found, you can quickly extract a sample of duplicate records. Finding the reason for duplication is then quite easy.

## Database information schema
Documentation and data dictionaries are always missing when they are needed the most. In such cases, the information_schema of our database can come handy. It might not tell you exactly what a particular data point represents, but you might be able to understand some obscured information and if you love challenges, even do some reverse engineering.

Do you want to get some useful information about columns in a table/view?
```
-- MOCKUP ------------------------
SELECT
  -- concatenates schema and table name to full address
  (table_schema || '.' || table_name) AS address
  ,column_name
  ,data_type
  ,ordinal_position
  ,character_maximum_length
  ,column_default
FROM information_schema.columns
WHERE 1=1
  AND table_schema = 'schema_name'
  AND table_name = 'table_name'
ORDER BY table_name, ordinal_position
```
Do you need a list of tables and views that follow some particular naming patterns?
```
-- MOCKUP ------------------------
SELECT DISTINCT
  -- concatenates schema and table name to full address
  (table_schema || '.' || table_name) AS address
FROM information_schema.columns
WHERE 1=1
  AND table_schema in ('schema_a','schema_b')
  AND table_name ilike ANY(ARRAY['%peekaboo%','%peek-a-boo%','%pkb%'])
```
Do you need to understand how a particular UDF (user defined function) works? Let’s check its definition!
```
-- MOCKUP ------------------------
SELECT
    r.routine_schema
    ,r.routine_name
    -- concatenate input arguments to a single string
    ,string_agg(
        p.parameter_name
        || '(' || p.data_type || ')',
        ',' ORDER BY p.ordinal_position
    ) AS input_arguments
    ,r.data_type AS output_data_type
    ,r.routine_definition

FROM information_schema.routines AS r
LEFT JOIN information_schema.parameters AS p
    ON r.specific_name = p.specific_name
    AND r.specific_schema = p.specific_schema
WHERE 1=1
    AND r.specific_schema = 'schema_a'
    AND r.routine_name ilike 'function_a'
GROUP BY
    r.routine_schema
    ,r.routine_name
    ,r.data_type
    ,r.routine_definition
```
Or a view definition?
```
-- MOCKUP ------------------------
SELECT
    table_schema
    ,table_name
    ,view_definition
FROM information_schema.views
WHERE 1=1
    AND table_schema = 'schema_a'
    AND table_name = 'view_a';
```
Be aware though that some information might be restricted to you without proper access grants.

## Naming conventions
Now that you are aware you can use the information_schema to your advantage, I guess you might understand a bit better why some people might request to follow some particular naming conventions when creating database objects like tables, views or functions.

Using naming conventions can help you identify database objects that belong to a particular context, project or workstream. Also, if naming conventions are used and you or your team follows some standards, you gain the ability to automate a lot of dull tasks with python or some other programming language. Or even with Excel if you get creative.

Again, no need to overkill it with rules. Just try to be consistent.

## Timestamps
Are you creating a table or preparing some analysis? Keep track of timestamps. Timestamps store information on when a certain event occurred, such as the time when you generated an analysis or when some data points were ingested or modified.

Maybe you have already been in a situation where your stakeholder complained about the quality of data you provided. And after a couple of days frantically trying to figure out where the problem is, it became clear that they were comparing a report from last week with data from a month ago. If you would have timestamps in both datasets, the issue would be resolved in minutes. Without it, you can lose the argument even if your data were correct.
```
-- EXECUTABLE --------------------
WITH
sample_data(product_id, quantity, revenue) AS (
  VALUES
     ('a', 12, 18.6)
    ,('b', 6, 21.3)
    ,('c', 7, 9.1)
    ,('a', 9, 13.5)
    ,('c', 13, 20.2)
)
SELECT
  product_id
  ,SUM(quantity)         AS product_quantity
  ,SUM(revenue)          AS product_revenue
  ,current_timestamp     AS capture_timestamp
FROM sample_data
GROUP BY
  product_id
```
When creating new tables, you can include a timestamp column that automatically tracks the timestamp when data is inserted.
```
-- MOCKUP ------------------------
CREATE TABLE tb_{TAG}_{OBJECT_NAME}(
    id VARCHAR(32) NOT NULL
    ...
    ,insert_timestamp TIMESTAMPTZ DEFAULT current_timestamp
)
```
## Script templates and code snippets
As you might already understand, keeping a collection of useful code snippets can be a good idea. I strongly recommend doing so and generalizing the code continuously. I do enjoy that exercise and it saved me a ton of time coding and debugging, helped me to provide useful information and insights quickly and build solutions in almost no time.

Collect your snippets, keep them handy, generalize them and create script templates for table/view definitions, functions and procedures. Your team, colleagues and collaborators will benefit from it as well.

Thank you very much for your attention! I hope I did not waste your time and you found something helpful here. If you would like to share some pragmatic concepts you use in your practice or give me some tips on how to improve the presentation, please do so, I would love to hear from you!

In the next installment of Pragmatic SQL, I plan to talk about JSON/JSONB data types and how they can be leveraged to make your SQL practice more agile.

Happy coding!