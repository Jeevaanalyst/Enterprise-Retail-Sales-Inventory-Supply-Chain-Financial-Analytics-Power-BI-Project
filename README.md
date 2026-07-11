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

Power BI Analytic Dashboards  - Enterprise Retail Sales, Inventory & Supply Chain Analytics Link :

https://app.powerbi.com/groups/me/reports/9fae2459-e27d-430c-a9f4-42780d7045fd/09e091f06e23b12e0234?experience=power-bi

<img width="190" height="217" alt="image" src="https://github.com/user-attachments/assets/edaf2fff-165b-4e6f-b7a4-457d3bb773e7" />

        Scan to View 


         DAX 

EXECUTIVE DASHBOARD


Total Sales = SUM(fact_sales[sales])


Total Profit = SUM(fact_sales[profit])


Total Cost = SUM(fact_sales[cost])


Total Orders = DISTINCTCOUNT(fact_sales[order_id])


Customer Count = DISTINCTCOUNT(fact_sales[customer_id])


Profit Margin % = DIVIDE([Total Profit],[Total Sales],0)*100


Average Order Value = DIVIDE([Total Sales],[Total Orders],0)



SALES ANALYSIS

YoY Sales Growth % =
VAR current_year = [Total Sales]
VAR prior_year =
    CALCULATE(
        [Total Sales],
        SAMEPERIODLASTYEAR(dim_calendar[date])
    )
RETURN
DIVIDE(current_year - prior_year, prior_year, 0) * 100

YTD Sales Last Year =
CALCULATE(
    [Total Sales],
    DATESYTD(SAMEPERIODLASTYEAR(dim_calendar[date]))
)



Discount Band =
SWITCH(TRUE(),
    fact_sales[discount]=0, "0%",
    fact_sales[discount]<=0.1, "1-10%",
    fact_sales[discount]<=0.2, "11-20%",
    fact_sales[discount]<=0.3, "21-30%",
    "30%+"
)




CUSTOMER ANALYTICS



Age Band =
SWITCH(TRUE(),
    dim_customer[age]<25, "18-24",
    dim_customer[age]<35, "25-34",
    dim_customer[age]<45, "35-44",
    dim_customer[age]<55, "45-54",
    "55+"
)



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




INVENTORY DASHBOARD




Inventory Turnover =
DIVIDE(SUM(fact_inventory[units_sold_this_month]), AVERAGE(fact_inventory[stock]), 0)


Reorder Alert Count =
CALCULATE(COUNTROWS(fact_inventory), fact_inventory[stock] < fact_inventory[reorder_level])


Dead Stock Flag =
IF(CALCULATE(SUM(fact_inventory[units_sold_this_month]))=0, "Dead Stock", "Active")




SUPPLY CHAIN DASHBOARD



On-Time Delivery % =
DIVIDE(CALCULATE(COUNTROWS(fact_shipment), fact_shipment[delay_days]<=2), COUNTROWS(fact_shipment), 0)*100


Avg Delivery Delay = AVERAGE(fact_shipment[delay_days])


Total Freight Cost = SUM(fact_shipment[freight_cost])



FINANCIAL DASHBOARD (WITH WATERFALL)



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



3 Month Moving Avg =
CALCULATE(
    AVERAGEX(
        DATESINPERIOD(dim_calendar[date], LASTDATE(dim_calendar[date]), -3, MONTH),
        [Total Sales]
    )
)





Row-Level Security

Modeling → Manage roles → New role "Regional Manager"

dax[region] = USERPRINCIPALNAME()
