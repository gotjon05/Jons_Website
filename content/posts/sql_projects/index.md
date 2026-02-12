+++
date = '2026-02-11T12:42:04-05:00'
draft = false
title = 'Compilation of SQL Challenges'
+++

For this compilation, I’m going to show my thought process and attempts at solving SQL challenges that helped me build a deeper understanding of the language.

{{< fold title="Challenges from Microsoft DP-080 T-SQL Course" >}}
{{< fold title="Labs/03b-subqueries: Challenge" >}}
[Link to challenge](https://microsoftlearning.github.io/dp-080-Transact-SQL/Instructions/Labs/03b-subqueries.html)

Adventure Works products each have a standard cost price that indicates the cost of manufacturing the product, and a list price that indicates the recommended selling price for the product. This data is stored in the SalesLT.Product table. Whenever a product is ordered, the actual unit price at which it was sold is also recorded in the SalesLT.SalesOrderDetail table. You must use subqueries to compare the cost and list prices for each product with the unit prices charged in each sale.

1. Retrieve products whose list price is higher than the average unit price.
Retrieve the product ID, name, and list price for each product where the list price is higher than the average unit price for all products that have been sold.
Tip: Use the AVG function to retrieve an average value.

    **Requirements**

    - required columns: Product ID, name, list price
    - SELF JOIN SalesLT.Product with SalesLT.SalesOrderDetail to get unit price 
    - filter by list price > unit price
```sql
SELECT
    p.ProductID,
    p.Name
FROM 
    [adventureworks].[SalesLT].[Product] AS p
JOIN
    [adventureworks].[SalesLT].[SalesOrderDetail] AS o
ON
    p.ProductID = o.ProductID
WHERE 
    p.ListPrice > o.UnitPrice
```
{{< fold title="Solving this problem using Subquery" >}}
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
    AND  
        p.ListPrice > o.UnitPrice
)
```
{{< /fold >}}


{{< /fold >}}



{{< /fold >}}