# Amazon_sales (Phân tích một phần về hoạt động kinh doanh của sàn thương mại điện tử Amazon) 💻🏷️

## Introduction (Giới thiệu)

Với mục chính của việc phân tích 📊 dữ liệu hoạt động của sản thương mại điện tử Amazon nhằm việc phân tích về hoạt động kinh doanh của sàn. Bên cạnh đó với mục đích khác nhằm rèn luyện, nâng cao những kỹ năng phân tích của tôi

## Problems (Vấn đề) ⁉️

### Những vấn đề cần 🔎 phân tích:

1. Truy vấn 10 sản phẩm hàng đầu theo tổng giá trị bán hàng. Bao gồm tên sản phẩm, tổng số lượng bán ra và tổng giá trị bán hàng
2. Tính tổng doanh thu tạo ra bởi mỗi danh mục sản phẩm. Bao gồm tỷ lệ phần trăm đóng góp của mỗi danh mục vào tổng doanh thu
3. Truy vấn tổng doanh số bán hàng trong năm qua. Hiển thị xu hương bán hàng, so sánh so với tháng liên trước trong cùng một năm
4. Xác định sản phẩm bán chạy nhất cho từng danh mục. Bao gồm tổng doanh số cho danh mục đó
5. Tính giá trị đơn hàng trung bình cho mỗi khách hàng. Chỉ bao gồm những khách hàng có hơn 20  đơn hàng. Và tìm giá trị trung bình của đơn hàng
6. Tìm 10 ID người bán hàng hàng đầu dựa trên tổng giá trị bán hàng. Và hiển thị phần trăm doanh số của họ trên tổng doanh số của Sales Rep
7. Xác định những sản phẩm có tăng trưởng doanh thu giảm (so sánh giữa năm 2019 và năm 2018). Giá trị trả về gồm tên sản phẩm, doanh thu năm 2018 và 2019 cùng với tỷ lệ thay đổi (tăng/giảm)

## Tools I Used (Những công cụ đã sử dụng) ⚙️🛠️

**- SQL Server:** Dùng để truy vấn

**- Power BI:** Để trực quan hóa dữ liệu dưới dạng biểu đồ, bảng biểu

**- Github:** Dùng để đăng bài phân tích lên nhằm mục đích chia sẻ cho mọi người cùng tham khảo và đánh giá


## The Analysis (Phân tích) 📉📈
1. Truy vấn 10 sản phẩm hàng đầu theo tổng giá trị bán hàng. Bao gồm tên sản phẩm, tổng số lượng bán ra và tổng giá trị bán hàng
```sql
SELECT TOP 10
  Item Product_Name, 
  FORMAT(SUM(Sales_Quantity), 'N0') Quantity,
  FORMAT(SUM((Sales_Quantity * Sales_Price)),'C0') Revenue
FROM Amazon
GROUP BY Item
ORDER BY SUM((Sales_Quantity * Sales_Price)) DESC
```
![Hình 1](Pictures/1.png)

2. Tính tổng doanh thu tạo ra bởi mỗi danh mục sản phẩm. Bao gồm tỷ lệ phần trăm đóng góp của mỗi danh mục vào tổng doanh thu
```sql
SELECT 
  Category, 
  FORMAT(Revenue, 'C0') Revenue, 
  FORMAT(Revenue/(SELECT SUM(Sales_Quantity * Sales_Price) Total_Revenue FROM Amazon), 'P3') 'Contribution(%)'
FROM 
  (SELECT 
    Line_Number Category, 
    SUM(Sales_Quantity * Sales_Price) Revenue
  FROM Amazon
  GROUP BY Line_Number) AS Table2
ORDER BY (Revenue/(SELECT SUM(Sales_Quantity * Sales_Price) Total_Revenue FROM Amazon)) DESC
```
![Hình 2](Pictures/2.png)

3. Truy vấn tổng doanh số bán hàng trong năm qua. Hiển thị xu hương bán hàng, so sánh so với tháng liền trước trong cùng một năm
```sql
SELECT
  Year, Month, FORMAT(Revenue, 'C0') Current_Revenue, 
  FORMAT(Previous_Month,'C0') Previous_Month,
  FORMAT(ISNULL((Revenue - Previous_Month)/Previous_Month,0), 'P') Sales_Trend
FROM 
  (SELECT
    Year, Month, Revenue,
    LAG(Revenue) OVER ( PARTITION BY Year ORDER BY Year ASC, Month ASC) Previous_Month
  FROM 
    (SELECT 
      YEAR(DateKey) Year, MONTH(DateKey) Month,
      SUM(Sales_Price * Sales_Quantity) Revenue
    FROM Amazon
    GROUP BY YEAR(DateKey), MONTH(DateKey)) AS Table1) AS Table2
```
![Hình 3](Pictures/3.png)

4. Xác định sản phẩm bán chạy nhất cho từng danh mục. Bao gồm tổng doanh số cho danh mục đó
```sql
WITH Table3 AS
(SELECT 
  Line_Number, FORMAT(SUM(Revenue),'C0') Total_Revenue
FROM
  (SELECT 
    Line_Number ,Item Product_name, SUM(Sales_Price * Sales_Quantity) Revenue  
  FROM Amazon
  GROUP BY Line_Number, Item) AS Table2
GROUP BY Line_Number),
Table4 AS
(SELECT
  Line_Number, Product_name, FORMAT(Revenue, 'C0') Revenue
FROM 
  (SELECT 
    Line_Number, Item Product_name, 
    SUM(Sales_Price * Sales_Quantity) Revenue,
    RANK() OVER (PARTITION BY Line_Number ORDER BY SUM(Sales_Price * Sales_Quantity) DESC) Rank
  FROM Amazon 
  GROUP BY Item, Line_Number, Sales_Price, Sales_Quantity) AS Table1
WHERE Rank = 1)
-------
SELECT
  T4.Line_Number, Product_name, Revenue, Total_Revenue
FROM Table3 as T3 
JOIN Table4 as T4 ON T3.Line_Number = T4.Line_Number
```
![Hình 4](Pictures/4.png)

5. Tính giá trị đơn hàng trung bình cho mỗi khách hàng. Chỉ bao gồm những khách hàng có hơn 20  đơn hàng. Và tìm giá trị trung bình của đơn hàng
```sql
WITH Table1 AS
(  
  SELECT 
    Custkey Customer
  FROM Amazon
  GROUP BY Custkey
  HAVING COUNT(Invoice_Number) > 20
),
Table2 AS  
(		
  SELECT  -- Create subtale to made index number column
    ROW_NUMBER() OVER (ORDER BY Avg_Order_value DESC) Row_num,
    Customer, Avg_Order_value
  FROM 
    (SELECT 
      Custkey Customer, 
      CAST(AVG(Sales_Price * Sales_Quantity) AS Decimal(10,2)) Avg_Order_value
    FROM Amazon
    WHERE Custkey in (SELECT * FROM table1)
    GROUP BY Custkey) AS Table3
),
Median_table AS
(
  SELECT 
    COUNT(*) Number_order,
    CASE
      WHEN COUNT(*) % 2 = 0 THEN COUNT(*)/2
      ELSE (COUNT(*)+1) / 2
    END Median_value1,
    CASE
      WHEN COUNT(*) % 2 = 0 THEN (COUNT(*)+2)/2
      ELSE (COUNT(*)+1) / 2
    END Median_value2
  FROM Table2
)
--- Find the median value
SELECT
  FORMAT(AVG(Avg_order_value), 'C2') Median_value_of_Order
FROM 
  (SELECT 
    Row_num, Customer,
    Avg_order_value
  FROM Table2 as t2 
  JOIN Median_table as mt ON mt.Median_value1 = t2.Row_num OR mt.Median_value2 = t2.Row_num) as Result
```
![Hình 5](Pictures/5.png)

6. Tìm 10 ID người bán hàng hàng đầu dựa trên tổng giá trị bán hàng. Và hiển thị phần trăm doanh số của họ trên tổng doanh số của Sales Rep
```sql
WITH Sale_Total AS
(SELECT 
  SUM(Sales_Price * Sales_Quantity) Total_Revenue
FROM Amazon),
Sales AS
(SELECT
    Sales_Rep Seller, 
    FORMAT(SUM(Sales_Price * Sales_Quantity),'C0') Revenue,
    SUM(Sales_Price * Sales_Quantity) Sales_Value
FROM Amazon
GROUP BY Sales_Rep)
-----
SELECT TOP 10
  Seller, Revenue, 
  FORMAT((Sales_Value/(SELECT*FROM Sale_Total)),'P2') '%Contribution'
FROM Sales
ORDER BY Sales_Value DESC
```
![Hình 6](Pictures/6.png)

7. Xác định những sản phẩm có tăng trưởng doanh thu giảm (so sánh giữa năm 2019 và năm 2018). Giá trị trả về gồm tên sản phẩm, doanh thu năm 2018 và 2019 cùng với tỷ lệ thay đổi (tăng/giảm)
```sql
WITH Table1 AS
(SELECT 
  YEAR(Datekey) Year, Item Product,
  SUM(Sales_Price * Sales_Quantity) Total_Revenue
FROM Amazon
WHERE YEAR(Datekey) BETWEEN 2018 and 2019
GROUP BY Item, YEAR(Datekey)
)
SELECT 
  Product, [Year-2019] , 
  FORMAT( [Revenue-2019], 'C0') '2019-Revenue', FORMAT([Revenue-2018], 'C0') '2018-Revenue',
  [%Change]
FROM 
  (SELECT 
    cur.Product Product,
    pre.Year 'Year-2018', pre.Total_Revenue 'Revenue-2018',
    cur.Year 'Year-2019', cur.Total_Revenue 'Revenue-2019',
    FORMAT( (cur.Total_Revenue - pre.Total_Revenue)/ CAST(pre.Total_Revenue as decimal(10,2)), 'P2') '%Change',
    ( cur.Total_Revenue - pre.Total_Revenue )/ CAST(pre.Total_Revenue as decimal(10,2)) To_order
  FROM Table1 as cur 
  JOIN Table1 as pre ON pre.Year = cur.Year - 1 AND pre.Product = cur.Product) AS Final_table
WHERE to_order < 0
ORDER BY To_order ASC, Product ASC
```
![Hình 7](Pictures/7.png)

## Conclusion (Kết luận) 📝



