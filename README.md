
# RFM Customer Sales Analysis 

### Here is the Sales Dashboard Screenshot  with link made in Tableau:-

Sales Dashboard 1 : - 
https://public.tableau.com/views/RFM_Segmentation_Sales_Analysis_Dasboard1/SalesDashboard?:language=en-US&publish=yes&:sid=&:display_count=n&:origin=viz_share_link

Sales Dashboard 2 : - 
https://public.tableau.com/views/RFM_Segmentation_Sales_Analysis_Dasboard2/SalesDashboard2?:language=en-US&publish=yes&:sid=&:display_count=n&:origin=viz_share_link


This project aims to analyze sales data from the customer sales_data_sample table to gain insights into product sales, customer behavior, and revenue trends.

## What is RFM ? 

RFM (Recency, Frequency, Monetary) analysis is a method used to analyze and categorize customers based on their purchasing behavior. It involves the following key components:

- **Recency (R)**: How recently a customer has made a purchase. It is calculated as the number of days since the customer's last purchase.

- **Frequency (F)**: How often a customer makes a purchase. It is calculated as the total number of purchases made by the customer over a certain period.

- **Monetary Value (M)**: How much money a customer has spent. It is calculated as the total monetary value of all purchases made by the customer over a certain period.

RFM analysis helps businesses identify different customer segments, such as:

- **Loyal Customers**: High Frequency, High Monetary Value, and recent purchases.
- **Potential Churners**: High Frequency and Monetary Value but not purchased recently.
- **New Customers**: Recent first purchase.
- **Lost Customers**: Low Frequency, Low Monetary Value, and not purchased recently.

By segmenting customers based on RFM, businesses can tailor their marketing strategies to better meet the needs of each customer segment and improve overall customer satisfaction and retention.


```sql
-- Inspecting the data

SELECT * FROM [dbo].sales_data_sample;

-- Checking unique values 

SELECT DISTINCT status FROM [dbo].sales_data_sample; -- Clean Data
SELECT DISTINCT YEAR_ID FROM [dbo].sales_data_sample; -- Clean Data
SELECT DISTINCT PRODUCTLINE FROM [dbo].sales_data_sample; -- Clean Data
SELECT DISTINCT COUNTRY FROM [dbo].sales_data_sample; -- Clean Data
SELECT DISTINCT DEALSIZE FROM [dbo].sales_data_sample; -- Clean Data
SELECT DISTINCT TERRITORY FROM [dbo].sales_data_sample; -- Clean Data

-- Analysis

-- Let's start by grouping sales by productline

SELECT
    PRODUCTLINE, SUM(sales) Revenue 
FROM [dbo].sales_data_sample
GROUP BY PRODUCTLINE 
ORDER BY 2 DESC;

SELECT
    YEAR_ID, SUM(sales) Revenue 
FROM [dbo].sales_data_sample
GROUP BY YEAR_ID 
ORDER BY 2 DESC;

SELECT
    DEALSIZE, SUM(sales) Revenue 
FROM [dbo].sales_data_sample
GROUP BY DEALSIZE 
ORDER BY 2 DESC;

-- What was the best month for the sales in a specific year ? How much was earned that month ? 

SELECT MONTH_ID, SUM(sales) Revenue, COUNT(Ordernumber) Frequency 
FROM sales_data_sample
WHERE YEAR_ID = 2003
GROUP BY MONTH_ID
ORDER BY 2 DESC;

-- From above Query we can see that November is the best month for sales, now finding out out what products were sold in November ?

SELECT MONTH_ID, Productline, SUM(sales) Revenue, COUNT(Ordernumber) Frequency 
FROM sales_data_sample
WHERE YEAR_ID = 2003 AND MONTH_ID = 11
GROUP BY MONTH_ID, PRODUCTLINE
ORDER BY 3 DESC;

-- Classic Cars is the highest sold product with the frequency of 114 in the month of November in the year 2003---

-- Finding out the best customers---

( Doing RFM - Recency Frequency Monetary )
-- Recency = Last Order Date
-- Frequency = Count Of the Total Orders
-- Monetary Value = Total Spend

DROP TABLE IF EXISTS #RFM; 
WITH RFM as (
SELECT Customername,
    sum(sales) Monetary_Value,
    avg(sales) Avg_Monetary_Value,
    count(ordernumber) Frequency, 
    max(orderdate) Last_Order_Date,
    (Select MAX(orderdate) from [dbo].sales_data_sample) Max_Order_Date,
    DATEDIFF(DD, max(orderdate) , (Select MAX(orderdate) from [dbo].sales_data_sample) ) Recency
    from [dbo].sales_data_sample
    group by customername ),

RFM_Calc as (
    select r.*, 
        NTILE(4) OVER (order by Recency desc) RFM_Recency,
        NTILE(4) OVER (order by Frequency desc) RFM_Frequency,
        NTILE(4) OVER (order by Frequency desc) RFM_Monetary
    from RFM r 
    )

Select C.* , RFM_Recency + RFM_Frequency + RFM_Monetary as RFM_Cell ,
    cast(RFM_Recency as varchar) + cast( RFM_Frequency as varchar) + cast(RFM_Monetary as varchar ) AS RFM_Cell_String
into #rfm
from RFM_Calc C

    Select Customername , RFM_Recency , RFM_Frequency , RFM_Monetary, 
    case 
        when rfm_cell_string in (111, 112 , 121, 122, 123, 132, 211, 212, 114, 141) then 'lost_customers'  --lost customers
        when rfm_cell_string in (133, 134, 143, 244, 334, 343, 344, 144) then 'slipping away, cannot lose' -- (Big spenders who haven’t purchased lately) slipping away
        when rfm_cell_string in (311, 411, 331) then 'new customers'
        when rfm_cell_string in (222, 223, 233, 322) then 'potential churners'
        when rfm_cell_string in (323, 333,321, 422, 332, 432) then 'active' --(Customers who buy often & recently, but at low price points)
        when rfm_cell_string in (433, 434, 443, 444) then 'loyal'
    end rfm_segment

from #rfm

-- What products are most often sold together ?

Select distinct OrderNumber,  stuff ( 
(Select ',' + Productcode 
from sales_data_sample p
where ORDERNUMBER in 
(
        Select ordernumber
        from (
            select ordernumber ,  count(*) rn from 
            sales_data_sample where status = 'shipped'
            group by ORDERNUMBER
        )m
        where rn = 3
    )
    and p.ORDERNUMBER = s.ORDERNUMBER
    for xml  path(''))
    ,1 ,1 ,'') Product_Code_Name

from sales_data_sample s
order by 2 desc

```
