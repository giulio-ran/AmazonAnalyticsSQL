# # Amazon Analytics SQL Project

## Abstract and research questions






## Project structure

### 1. Database setup
- **Database creation**: The project starts by creating a database named 'amazonsalesanalytics'
- **Landing table creation**: A landing table is created to tsore the raw sales data
- **Star Schema setup**: various tables have been created, each one describinig a different dimensionof the data; each of these 'descriptive' tables can be linked to the principal 'fact' table through a single join.

```sql

/* Create a dedicated database for Amazon-style analytics */
CREATE DATABASE amazonsalesanalytics;
USE amazonsalesanalytics;

/* Create the main sales table with appropriate data types */
CREATE TABLE SalesTransactions (
    OrderID VARCHAR(50) PRIMARY KEY,
    OrderDate DATE,
    CustomerID VARCHAR(50),
    CustomerName VARCHAR(255),
    ProductID VARCHAR(50),
    ProductName VARCHAR(255),
    Category VARCHAR(100),
    Brand VARCHAR(100),
    Quantity INT,
    UnitPrice DECIMAL(10, 2),
    Discount DECIMAL(10, 2),
    Tax DECIMAL(10, 2),
    ShippingCost DECIMAL(10, 2),
    TotalAmount DECIMAL(10, 2),
    PaymentMethod VARCHAR(50),
    OrderStatus Varchar(100),
    City VARCHAR(100),
    State VARCHAR(100),
    Country VARCHAR(100),
    SellerID VARCHAR(50) );

/* Loading data from the original CSV dataset */
LOAD DATA INFILE '/Amazon.csv' 
INTO TABLE SalesTransactions 
FIELDS TERMINATED BY ',' 
ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 LINES 
(OrderID, OrderDate, CustomerID, CustomerName, ProductID, ProductName, 
 Category, Brand, Quantity, UnitPrice, Discount, Tax, ShippingCost, 
 TotalAmount, PaymentMethod, OrderStatus, City, State, Country, SellerID);

/* Create the table for the storage of customers' information */
CREATE TABLE customers (
    CustomerID VARCHAR(50) PRIMARY KEY,
    CustomerName VARCHAR(255)
);

INSERT into  customers (CustomerID, CustomerName)
SELECT 
    CustomerID, 
    MAX(CustomerName)
FROM SalesTransactions
GROUP BY CustomerID;


/* Creation of the table for the storage of unique sales */
CREATE TABLE Sales (
    OrderID VARCHAR(50) PRIMARY KEY,
    CustomerID VARCHAR(50),
    OrderDate DATE,
    ProductID VARCHAR(50),
    Quantity INT,
    UnitPrice DECIMAL(10, 2),
    Discount DECIMAL(10, 2),
    Tax DECIMAL(10, 2),
    ShippingCost DECIMAL(10, 2),
    TotalAmount DECIMAL(10, 2),
    PaymentMethod VARCHAR(50),
    OrderStatus VARCHAR(100),
    FOREIGN KEY (CustomerID) REFERENCES Customers(CustomerID)
);


INSERT INTO Sales (
    OrderID, CustomerID, OrderDate, ProductID, 
    Quantity, UnitPrice, Discount, Tax, 
    ShippingCost, TotalAmount, PaymentMethod, OrderStatus
)
SELECT 
    OrderID, CustomerID, OrderDate, ProductID, 
    Quantity, UnitPrice, Discount, Tax, 
    ShippingCost, TotalAmount, PaymentMethod, OrderStatus
FROM SalesTransactions;

drop table products;

/* Creation of the table for the storage of the product information */
CREATE TABLE Products (
    ProductID VARCHAR(100) PRIMARY KEY,
    ProductName VARCHAR(255),
    Category VARCHAR(100),
    Brand VARCHAR(100)
);


INSERT INTO Products (ProductID, ProductName, Category, Brand)
SELECT 
    CONCAT('P', DENSE_RANK() OVER (ORDER BY ProductName, Category, Brand)) AS ProductID,
    ProductName,
    Category,
    Brand
FROM (
    SELECT DISTINCT ProductName, Category, Brand 
    FROM SalesTransactions
    WHERE ProductName IS NOT NULL
) AS ActualProducts;

SELECT * FROM Products
ORDER BY CAST(SUBSTR(ProductID, 2) AS UNSIGNED);

CREATE INDEX idx_prod_lookup ON Products(ProductName(100), Category(50), Brand(50));

CREATE INDEX idx_sales_lookup ON SalesTransactions(ProductName(100), Category(50), Brand(50));

UPDATE SalesTransactions t
JOIN Products p ON 
    t.ProductName = p.ProductName AND 
    t.Category = p.Category AND 
    t.Brand = p.Brand
SET t.ProductID = p.ProductID;


/* Geography dimension creation */
CREATE TABLE Geography (
    GeographyID INT AUTO_INCREMENT PRIMARY KEY,
    City VARCHAR(100),
    State VARCHAR(100),
    Country VARCHAR(100)
);

INSERT INTO Geography (City, State, Country)
SELECT DISTINCT City, State, Country 
FROM SalesTransactions
WHERE City IS NOT NULL;

/* Substituting old productID with the updated ones. */
TRUNCATE TABLE Sales;

INSERT INTO Sales (
    OrderID, CustomerID, OrderDate, ProductID, 
    Quantity, UnitPrice, Discount, Tax, 
    ShippingCost, TotalAmount, PaymentMethod, OrderStatus
)
SELECT 
    OrderID, CustomerID, OrderDate, ProductID, 
    Quantity, UnitPrice, Discount, Tax, 
    ShippingCost, TotalAmount, PaymentMethod, OrderStatus
FROM SalesTransactions;
```

### 2. Business Analysis
- **Top-10 big spenders**: In this section, I focused on retreiving the top-10 'big spenders', i.e. the customers that spent more on total, also computing the average total spent by order.
- **Geographic analysis**: Here, I grouped sales by city, and determined the top-5 cities in revenue
- **Monthly orders**: At last, I retrieved the total orders by month, in order to determine a rank of most profitable months.

```sql
 /* Who are the top-10 'big spenders' and how much did they order on average*/
SELECT 
    c.CustomerName,
    COUNT(s.OrderID) AS NumberOfOrders,
    SUM(s.TotalAmount) AS TotalAmount,
    ROUND(AVG(s.TotalAmount), 2) AS AverageAmount
FROM Sales s
JOIN Customers c ON s.CustomerID = c.CustomerID
GROUP BY c.CustomerID, c.CustomerName
HAVING NumberOfOrders > 1
ORDER BY TotalAmount DESC
LIMIT 10;


/* Performance for Category/Brand 
 * Useful to understand which combination works best */
SELECT 
    p.Category,
    p.Brand,
    SUM(s.Quantity) AS SingleProductsSold,
    SUM(s.TotalAmount) AS TotalIncome
FROM Sales s
JOIN Products p ON s.ProductID = p.ProductID
GROUP BY p.Category, p.Brand
ORDER BY TotalIncome DESC;
/* The combination of category/brand which generated the highest total income is Toys & Games/CoreTech */

/* Geographic analysis:
 * In which city we have the maiority of sales */
SELECT 
    g.Country,
    g.City,
    COUNT(s.OrderID) AS TotalOfOrders,
    SUM(s.TotalAmount) AS TotalAmountPerCity
FROM Sales s
JOIN Geography g ON s.OrderID IN (SELECT OrderID FROM SalesTransactions WHERE City = g.City) 
GROUP BY g.Country, g.City
ORDER BY TotalAmountPerCity DESC
LIMIT 5;
/* The city of Charlotte generated the highest income */

/* Monthly trend of sales */
SELECT 
    DATE_FORMAT(OrderDate, '%Y-%m') AS Month,
    COUNT(OrderID) AS NumberOfOrders,
    SUM(TotalAmount) AS MonthlyIncome
FROM Sales
GROUP BY Month
ORDER BY Month;
/*The month with the highest number of orders is Genuary 2020 */
```


We can observe that, in the top-3 of the big spenders, the number of orders is quite high (10,9,6 in descending order), while the average amount per sale is relatively modest, meaning that generally quality customers generate a high revenue by placing several medium order, rather than few big orders.

