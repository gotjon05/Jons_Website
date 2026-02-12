+++
date = '2026-02-11T12:42:04-05:00'
draft = false
title = 'Compilation of SQL Challenges'
+++

For this compilation, I’m going to show my thought process and attempts at solving SQL challenges that helped me build a deeper understanding of the language.

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





{{< /fold >}}
{{< /fold >}}