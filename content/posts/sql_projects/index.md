+++
date = '2026-02-11T12:42:04-05:00'
draft = false
title = 'Compilation of SQL Challenges'
+++

For this compilation, I’m going to show my thought process and attempts at solving SQL challenges that helped me wrap my head around important concepts.

{{< fold title="Challenges from Microsoft DP-080 T-SQL Course" >}}
{{< fold title="Labs/03b-subqueries: Challenge" >}}
[Link to challenge](https://microsoftlearning.github.io/dp-080-Transact-SQL/Instructions/Labs/03b-subqueries.html)

**Challenge 1: Retrieve product price information**

Adventure Works products each have a standard cost price that indicates the cost of manufacturing the product, and a list price that indicates the recommended selling price for the product. This data is stored in the SalesLT.Product table. Whenever a product is ordered, the actual unit price at which it was sold is also recorded in the SalesLT.SalesOrderDetail table. You must use subqueries to compare the cost and list prices for each product with the unit prices charged in each sale.

1. Retrieve products whose list price is higher than the average unit price.
Retrieve the product ID, name, and list price for each product where the list price is higher than the average unit price for all products that have been sold.
Tip: Use the AVG function to retrieve an average value.

**Requirements**

- required columns: Product ID, name, list price
- SELF JOIN SalesLT.Product with SalesLT.SalesOrderDetail to get unit price 
- filter by list price > AVG(unit price)
- Required to use a subquery instead of just Joining tables

For each product, I’m checking whether there is at least one related order-detail row where p.ProductID = o.ProductID and p.ListPrice > o.UnitPrice.

```sql
SELECT
    p.ProductID,
    p.Name,
    p.ListPrice
FROM 
    [adventureworks].[SalesLT].[Product] AS p
WHERE EXISTS(
    SELECT 1
    FROM
        [adventureworks].[SalesLT].[SalesOrderDetail] AS o
    WHERE
        p.ProductID = o.ProductID
    HAVING  
        p.ListPrice > AVG(o.UnitPrice)
)

```

2. Retrieve Products with a list price of 100 or more that have been sold for less than 100.
Retrieve the product ID, name, and list price for each product where the list price is 100 or more, and the product has been sold for less than 100.

**Requirements**
- required columns: ProductID, Name, ListPrice
- filter product by list price >= 100 AND unit price < 100 (language ambigious for sold for, being unit price)

ListPrice is in [SalesLT].[Product], the outer query so I filtered for ListPrice there. 
And I'm checking to see if theres at least one row of salesorderdetail for the same product where  o.UnitPrice < 100


```sql
SELECT
    p.ProductID,
    p.Name,
    p.ListPrice
FROM 
    [adventureworks].[SalesLT].[Product] AS p
WHERE 
    p.ListPrice >= 100
AND
EXISTS(
    SELECT 1
    FROM
        [adventureworks].[SalesLT].[SalesOrderDetail] AS o
    WHERE
        p.ProductID = o.ProductID
    AND
        o.UnitPrice < 100
)
```
**Challenge 2: Analyze profitability**

The standard cost of a product and the unit price at which it is sold determine its profitability. You must use correlated subqueries to compare the cost and average selling price for each product.

1. Retrieve the product ID, name, cost, and list price for each product along with the average unit price for which that product has been sold.

**Requirements**

- Columns required: ProductID, Name, Cost, ListPrice, avg unit price
- correlated subqueries

I dont want an aggregate of ProductID, Name, Cost, ListPrice with avg unit price. I want the Unit price avg for ProductID only.

Going to create a subquery in SELECT to find this avg with ProductID and ListPrice only. 

This is a scalar subquery, my subquery needs to only return one value, the avg of unitprice 

Attempt 1: 
```sql
SELECT
    p.ProductID,
    p.Name,
    p.StandardCost,
    p.ListPrice,
    (SELECT AVG(o.UnitPrice) AS avg_UnitPrice
    FROM [adventureworks].[SalesLT].[Product] AS p
    JOIN [adventureworks].[SalesLT].[SalesOrderDetail] AS o
        ON p.ProductID = o.ProductID
    GROUP BY p.ProductID
)
FROM [adventureworks].[SalesLT].[Product] AS p;
```
I made a mistake using GROUP BY in the subquery. I got the error: Each GROUP BY expression must contain at least one column that is not an outer reference. I just had to reference the inner column with ```GROUP BY o.ProductID```
 
But in the process, I realized I don’t need GROUP BY at all. I could remove it and get the same outcome because this query is correlated

The subquery is correlated because it references the outer query via o.ProductID = p.ProductID, so it behaves by running the subquery once per outer query row. 

And AVG collapses the matching inner query rows into one into one scalar value, the average UnitPrice for each outer query ProductID


Answer:
```sql
SELECT
    p.ProductID,
    p.Name,
    p.StandardCost,
    p.ListPrice,
    (SELECT AVG(o.UnitPrice)
    FROM [adventureworks].[SalesLT].[SalesOrderDetail] AS o
    WHERE o.ProductID = p.ProductID
    )
FROM 
    [adventureworks].[SalesLT].[Product] AS p
```

2. Retrieve products that have an average selling price that is lower than the cost. Filter your previous query to include only products where the cost price is higher than the average selling price.

Requirements:

- Columns required: ProductID, Name, Cost, ListPrice, avg unit price
- correlated subqueries
- cost_price is > AVG(unit_price)
- unclear if costprice is StandardCost, very annoying

My Plan:

I need to create a subquery in the WHERE statement to provide this filter of StandardCost is > AVG(unit_price) for the entire query. Going to compare cost_price with the subquery that will provide the cost_price for each row of the outer query. 

Solution:

```sql 
SELECT
    p.ProductID,
    p.Name,
    p.StandardCost,
    p.ListPrice,
    (SELECT AVG(o.UnitPrice) AS avg_UnitPrice
    FROM [adventureworks].[SalesLT].[SalesOrderDetail] AS o
    WHERE p.ProductID = o.ProductID
)
FROM 
    [adventureworks].[SalesLT].[Product] AS p;
WHERE p.StandardCost > (
        SELECT AVG(UnitPrice)
        FROM [adventureworks].[SalesLT].[SalesOrderDetail] AS o
        WHERE o.ProductID = p.ProductID
    )

```

{{< /fold >}}
{{< /fold >}}