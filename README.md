# Customer-Segmentation-using-RFM-Analysis
This project demonstrates customer segmentation using the RFM (Recency, Frequency, Monetary) analysis technique. By analyzing customer transaction data, I segmented customers into categories such as "New Customers", "Loyal Customers", "At Risk Customers", and "Lost Customers", providing valuable insights for optimizing marketing strategies.

## SQL Script

```.sql
-- Inspecting Data 
SELECT * 
FROM [dbo].[sales_data_sample];

-- Checking Unique Values
SELECT DISTINCT status FROM [dbo].[sales_data_sample]; -- Nice to plot 
SELECT DISTINCT YEAR_ID FROM [dbo].[sales_data_sample];
SELECT DISTINCT PRODUCTLINE FROM [dbo].[sales_data_sample]; -- Nice to plot 
SELECT DISTINCT COUNTRY FROM [dbo].[sales_data_sample]; -- Nice to plot 
SELECT DISTINCT DEALSIZE FROM [dbo].[sales_data_sample]; -- Nice to plot 
SELECT DISTINCT TERRITORY FROM [dbo].[sales_data_sample]; -- Nice to plot 

-- Analysis 

ALTER TABLE sales_data_sample
ALTER COLUMN SALES int;

-- Let's start by grouping sales ProductLine - The datatype of the sales column is nvarchar and we need to get sum of sales
-- so that's why we use cast to convert the data type of sales column and we convert it to decimal.
SELECT PRODUCTLINE, SUM(CAST(sales AS decimal(18, 2))) AS Revenue
FROM [dbo].[sales_data_sample]
GROUP BY PRODUCTLINE
ORDER BY Revenue DESC;

-- Sales by Year 
SELECT YEAR_ID, SUM(CAST(Sales AS Decimal(18,2))) AS Revenue
FROM [dbo].[sales_data_sample]
GROUP BY YEAR_ID
ORDER BY Revenue DESC;

-- Sales by Size
SELECT DEALSIZE, SUM(CAST(Sales AS Decimal(18,2))) AS Revenue
FROM [dbo].[sales_data_sample]
GROUP BY DEALSIZE
ORDER BY Revenue DESC;

-- What was the best month for sales in a specific year & How much earned that month? 
SELECT Month_ID, SUM(CAST(Sales AS decimal(18,2))) AS Revenue, COUNT(OrderLINENumber) AS Frequency
FROM [dbo].[sales_data_sample]
-- WHERE YEAR_ID = 2004
GROUP BY MONTH_ID
ORDER BY Revenue DESC;

-- November seems to be the month; what product do they sell in November?
SELECT ProductLine, Month_ID, SUM(CAST(Sales AS Decimal(18,2))) AS Revenue, COUNT(ORDERLINENUMBER) AS Frequency
FROM [dbo].[sales_data_sample]
WHERE MONTH_ID = 11
GROUP BY PRODUCTLINE, MONTH_ID
ORDER BY Revenue DESC;

-- Who is our best customer (This could be best answered with RFM)?
SELECT CUSTOMERNAME, Year_ID, Month_ID, SUM(CAST(Sales AS Decimal(18,2))) AS Revenue, COUNT(ORDERLINENUMBER) AS Frequency 
FROM [dbo].[sales_data_sample]
WHERE YEAR_ID = 2003 -- AND MONTH_ID = 1
GROUP BY CUSTOMERNAME, YEAR_ID, Month_ID
ORDER BY Revenue DESC;

-- RFM Analysis (we'll create CTE in order to make this script short and easy to use for the next coming)
DROP TABLE IF EXISTS #RFM; 
WITH RFM AS
(	
    SELECT CustomerName, 
           SUM(CAST(SALES AS Decimal(18,2))) AS Monetary_Value,
           AVG(CAST(SALES AS Decimal(18,2))) AS AVG_Monetary_Value,
           COUNT(ORDERLINENUMBER) AS Frequency, 
           MAX(OrderDate) AS Recent_Order_Date,
           (SELECT MAX(OrderDate) FROM [dbo].[sales_data_sample]) AS Max_Order_Date,
           DATEDIFF(DD, MAX(OrderDate), (SELECT MAX(OrderDate) AS Max_Order_Date FROM [dbo].[sales_data_sample])) AS Recency
    FROM [dbo].[sales_data_sample]
    GROUP BY CUSTOMERNAME
),
RFM_Calc AS 
(
    SELECT CustomerName, Recency, Frequency, Monetary_Value, 
           NTILE(4) OVER (ORDER BY Recency DESC) AS R_Score,	
           NTILE(4) OVER (ORDER BY Frequency DESC) AS F_Score,
           NTILE(4) OVER (ORDER BY Monetary_Value DESC) AS M_Score
    FROM RFM
)

SELECT r.*, 
       CAST(R_Score AS varchar) + CAST(F_Score AS varchar) + CAST(M_Score AS varchar) AS RFM_View
INTO #RFM
FROM RFM_Calc r 
ORDER BY RFM_View ASC;

SELECT CustomerName, Recency, Frequency, Monetary_Value, R_Score, F_Score, M_Score, RFM_View,
       CASE 
           WHEN RFM_View IN (433, 434, 443, 444) THEN 'Champions'
           WHEN RFM_View IN (321, 322, 323, 331, 332, 333, 334, 343, 244, 344, 432) THEN 'Loyal'
           WHEN RFM_View IN (221, 222, 223, 231, 232, 233, 224, 234, 422, 122, 212, 243, 123) THEN 'At Risk Customers'
           WHEN RFM_View IN (411, 421, 111, 412, 413, 414, 113, 114, 311, 211, 321, 312, 144) THEN 'New Customers'
           WHEN RFM_View IN (122, 121, 112, 113, 114, 124, 134, 133, 144, 143) THEN 'Lost Customers'
       END AS RFM_Segment
FROM #RFM;


