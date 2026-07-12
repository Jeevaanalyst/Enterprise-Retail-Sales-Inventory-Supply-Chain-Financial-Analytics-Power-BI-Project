Dataset Description :

fact_sales.csv -  Order ID, Order Date, Product ID, Customer ID, Store ID, Employee ID, Quantity, Sales, Cost, Profit, Discount 

act_inventory.csv -  Product ID, Warehouse, Stock, Reorder Level, Safety Stock, Lead Time 

fact_shipment.csv-  Shipment ID, Supplier ID, Product ID, Delivery Date, Delay Days, Freight Cost 

dim_product.csv -  Product ID, Name, Category, Brand, Price 

dim_customer.csv -  Customer ID, Gender, Age, City, State, Segment 

dim_store.csv-  Store ID, Region, Manager, Store Type 

dim_employee.csv-  Employee ID, Salesperson, Department 

dim_supplier.csv-  Supplier ID, Name, Country 

dim_calendar.csv-  Date, Year, Quarter, Month, Week, Fiscal Year 

-----------------------------------------------------------------------------------------------------------------------------------------------------------------

Power BI Analytic Dashboards  - Enterprise Retail Sales, Inventory & Supply Chain , Financial Analytics Link :

https://app.powerbi.com/groups/me/reports/db180b8e-4217-4bab-91f3-e32ab1c6dd32/09e091f06e23b12e0234?experience=power-bi




         

EXECUTIVE DASHBOARD :  ( DAX )
--------------------


Total Sales = SUM(fact_sales[sales])


Total Profit = SUM(fact_sales[profit])


Total Cost = SUM(fact_sales[cost])


Total Orders = DISTINCTCOUNT(fact_sales[order_id])


Customer Count = DISTINCTCOUNT(fact_sales[customer_id])


Profit Margin % = DIVIDE([Total Profit],[Total Sales],0)*100


Average Order Value = DIVIDE([Total Sales],[Total Orders],0)

------------------------------------------------------------------------------------------------------------------------------------------------------------------

SALES ANALYSIS :
---------------

Total Units Sold =
SUM(fact_sales[quantity])


Total Discount Given =
SUMX(
    fact_sales,
    fact_sales[sales] / (1 - fact_sales[discount]) * fact_sales[discount]
)



Avg Discount % =
AVERAGE(fact_sales[discount]) * 100

Discount Rate % =
DIVIDE(
    [Total Discount Given],
    [Total Sales] + [Total Discount Given],
    0
) * 100


Sales per Store =
DIVIDE(
    [Total Sales],
    DISTINCTCOUNT(fact_sales[store_id]),
    0
)



Active Store Count =
CALCULATE(
    DISTINCTCOUNT(dim_store[store_id]),
    dim_store[is_active] = 1
)


Running Total Sales =
CALCULATE(
    [Total Sales],
    FILTER(
        ALLSELECTED(dim_calendar[date]),
        dim_calendar[date] <= MAX(dim_calendar[date])
    )
)

Sales Rank by Product =
RANKX(
    ALLSELECTED(dim_product[product_id]),
    [Total Sales],
    ,
    DESC,
    DENSE
)



Bottom 10 Products Flag =
IF(
    RANKX(ALL(dim_product[product_id]), [Total Sales], , ASC, DENSE) <= 10,
    1, 0
)

YTD Sales =
CALCULATE(
    [Total Sales],
    DATESYTD(dim_calendar[date])
)


Discount Band =
SWITCH(TRUE(),
    fact_sales[discount]=0, "0%",
    fact_sales[discount]<=0.1, "1-10%",
    fact_sales[discount]<=0.2, "11-20%",
    fact_sales[discount]<=0.3, "21-30%",
    "30%+"
)


---------------------------------------------------------------------------------------------------------------------------------------------------------------

CUSTOMER ANALYTICS :
--------------------

Repeat Customer % =
DIVIDE([Repeat Customer Count], [Customer Count], 0) * 100


Avg CLV =
AVERAGEX(
    VALUES(dim_customer[customer_id]),
    CALCULATE([Total Sales])
)


New Customers This Period =
CALCULATE(
    DISTINCTCOUNT(fact_sales[customer_id]),
    FILTER(
        fact_sales,
        CALCULATE(
            MIN(fact_sales[order_date]),
            ALLEXCEPT(fact_sales, fact_sales[customer_id])
        ) >= MIN(dim_calendar[date])
    )
)


Avg Orders Per Customer =
DIVIDE([Total Orders], [Customer Count], 0)


Premium Customer Revenue % =
DIVIDE(
    CALCULATE([Total Sales], dim_customer[customer_segment] = "Premium"),
    [Total Sales],
    0
) * 100


Repeat Customer % =
VAR repeat_custs =
    COUNTROWS(FILTER(
        SUMMARIZE(fact_sales, fact_sales[customer_id], "orders", DISTINCTCOUNT(fact_sales[order_id])),
        [orders]>=2
    ))
RETURN DIVIDE(repeat_custs, [Customer Count], 0)*100



Avg CLV = AVERAGEX(VALUES(dim_customer[customer_id]), CALCULATE([Total Sales]))

Monthly Retention % =
VAR repeat_in_period =
    CALCULATE(
        COUNTROWS(FILTER(
            SUMMARIZE(fact_sales, fact_sales[customer_id], "orders", DISTINCTCOUNT(fact_sales[order_id])),
            [orders]>=2
        ))
    )
RETURN DIVIDE(repeat_in_period, [Customer Count], 0)*100

---------------------------------------------------------------------------------------------------------------------------------------------------------------


INVENTORY DASHBOARD :
---------------------


Current Total Stock =
CALCULATE(
    SUM(fact_inventory[stock]),
    fact_inventory[snapshot_date] = MAX(fact_inventory[snapshot_date])
)

Dead Stock SKU Count =
COUNTROWS(
    FILTER(
        SUMMARIZE(
            fact_inventory,
            fact_inventory[product_id],
            "total_sold", SUM(fact_inventory[units_sold_this_month])
        ),
        [total_sold] = 0
    )
)

Below Safety Stock Count =
CALCULATE(
    COUNTROWS(fact_inventory),
    fact_inventory[stock] < fact_inventory[safety_stock]
)

Avg Lead Time Days =
AVERAGE(fact_inventory[lead_time_days])

Inventory Value =
SUMX(
    fact_inventory,
    fact_inventory[stock] *
        RELATED(dim_product[cost_price])
)

All Products Sales =
CALCULATE(
    SUM(fact_inventory[units_sold_this_month]),
    ALL(dim_product)
)

Warehouse Utilization % =
DIVIDE(
    AVERAGE(fact_inventory[stock]),
    500,  -- assume 500 units max capacity per slot
    0
) * 100


Product Sales Total =
CALCULATE(SUM(fact_inventory[units_sold_this_month]))

Inventory Turnover =
DIVIDE(SUM(fact_inventory[units_sold_this_month]), AVERAGE(fact_inventory[stock]), 0)


Reorder Alert Count =
CALCULATE(COUNTROWS(fact_inventory), fact_inventory[stock] < fact_inventory[reorder_level])


Dead Stock Flag =
IF(CALCULATE(SUM(fact_inventory[units_sold_this_month]))=0, "Dead Stock", "Active")


------------------------------------------------------------------------------------------------------------------------------------------------------------------

SUPPLY CHAIN DASHBOARD :
-------------------------

Total Shipments =
COUNTROWS(fact_shipment)

On-Time Deliveries =
CALCULATE(
    COUNTROWS(fact_shipment),
    fact_shipment[delay_days] <= 2
)

On-Time Delivery % =
DIVIDE([On-Time Deliveries], [Total Shipments], 0) * 100

Avg Delivery Delay =
AVERAGE(fact_shipment[delay_days])

Total Freight Cost =
SUM(fact_shipment[freight_cost])

On-Time Delivery % =
DIVIDE(CALCULATE(COUNTROWS(fact_shipment), fact_shipment[delay_days]<=2), COUNTROWS(fact_shipment), 0)*100


Avg Delivery Delay = AVERAGE(fact_shipment[delay_days])


Total Freight Cost = SUM(fact_shipment[freight_cost])

-----------------------------------------------------------------------------------------------------------------------------------------------------------------

FINANCIAL DASHBOARD (WITH WATERFALL) :
----------------------------------------

Gross Revenue =
[Total Sales]

Net Margin % =
DIVIDE([Net Profit], [Total Sales], 0) * 100

Net Profit =
[Total Profit] - [Total Freight Cost]

Net Revenue =
[Total Sales]

Gross Profit =
[Total Profit]


Total Discounts =
SUMX(
    fact_sales,
    fact_sales[sales] / (1 - fact_sales[discount]) * fact_sales[discount]
)



PnL Value =
SWITCH(
    SELECTEDVALUE(PnL_Stages[Stage]),
    "Revenue", [Total Sales],
    "Discounts", -[Total Sales]*0.12,
    "Net Revenue", [Total Sales]*0.88,
    "COGS", -[Total Cost],
    "Gross Profit", [Total Profit],
    "Freight", -[Total Freight Cost],
    "Net Profit", [Total Profit]-[Total Freight Cost]
)



----------------------------------------------------------------------------------------------------------------------------------------------------------------


Row-Level Security

Modeling - Manage roles - New role "Regional Manager"

dax[region] = USERPRINCIPALNAME()
