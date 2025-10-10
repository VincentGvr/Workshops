### Diagram, Value Order, and Hierarchy

1. To get familiar with the data model, start by viewing the **Model** pane on the right. This helps you understand relationships between tables and the impact of upcoming filters or measures.
2. To simplify navigation, you can reorganize tables, expand/collapse them, or create a second diagram to include only the desired tables.  
    1. Example visuals:  
        - Adjusting table positions  
        - Filtering relationships  
        - Viewing tables more concisely  

3. **Value Display Order**:  
    1. Create a visual to display the `[SalesCount]` measure by month:
        - Use the Table visual: From the **Report** view, click the white page background, choose the Table visual, and select `Date\Month` and `DAX\SalesCount`.  
    2. Initially, sorting follows alphabetical order (e.g., January, February, etc., based on the first character).  
    3. To sort by `SalesCount`, click the table's ellipsis **(...)**, then **Sort By > Sales Count**.  
    4. If sorting by calendar month order is needed:
        - Go to the **Data** view, select the `Month` column in the Date table, and in the **Column Tools** tab, choose **Sort By Column**, then `MonthKey`.  
        - Note: Ensure both columns (`Month` and `MonthKey`) have the same cardinality (distinct values match row by row).  

5. **Creating Hierarchies**:
    1. Hierarchies simplify navigation, e.g., `Year > Quarter > Month > Day`.  
    2. Right-click a field (e.g., `Year`), select **Create Hierarchy**, and rename it accordingly.  
    3. To reorder hierarchy levels, use the **Model** view and adjust via the **Advanced** pane.  

---

### Understanding Row-by-Row Measures (`MeasureX`)  

In certain cases, results require row-by-row calculations:  
- Example: From the `SalesOrderDetail` table, calculate revenue using `[UnitPrice] * [OrderQty]`. Direct summation of the columns isn’t feasible due to row-specific variations.  

1. Use the `SUMX` function:  
    - Syntax: `SUMX(<table>, <expression>)`  
    - Table: `SalesOrderDetail`, Expression: `[UnitPrice] * [OrderQty]`  
    - Example:  
      ```  
      TotalSales = SUMX(SalesOrderDetail, SalesOrderDetail[UnitPrice] * SalesOrderDetail[OrderQty])  
      ```  
2. Display this measure in a Matrix visual, grouped by months.  

---

### Using Inactive Relationships in Calculations  

1. By default, measures aggregate data using active relationships.  
    - For example, `SalesOrderHeader` might aggregate by `DueDate`.  
    - To aggregate by another field (e.g., `ShipDate`), you can:  
      - Create another date table and establish a relationship.  
      - Use `USERELATIONSHIP` in a calculation to activate an existing inactive relationship.  
    - Example:  
      ```  
      TotalSalesShipDate = CALCULATE([TotalSales], USERELATIONSHIP('Date'[Date], SalesOrderHeader[ShipDate]))  
      ```  

---

### Measures Returning Tables vs. Scalar Values  

- Functions like `SUMMARIZE`, `FILTER`, and `CALCULATETABLE` return tables, while others like `SUM` or `AVERAGE` return scalar values.  
- Use cases for table-based calculations:  
  1. Create a table with only red products:  
     ```  
     RedProducts = CALCULATETABLE(SalesOrderDetail, Product[Color] = "Red")  
     ```  
  2. Aggregate quantity per order:  
     ```  
     QuantityPerOrders = SUMMARIZE(SalesOrderDetail, SalesOrderDetail[SalesOrderID], "SumQuantity", SUM(SalesOrderDetail[OrderQty]))  
     ```  
  3. Combine filters and aggregation:  
     ```  
     QuantityPerOrdersWithRedProducts = CALCULATETABLE(
         SUMMARIZE(SalesOrderDetail, SalesOrderDetail[SalesOrderID], "SumQuantity", SUM(SalesOrderDetail[OrderQty])), 
         Product[Color] = "Red"
     )  
     ```  

---

### Time Intelligence  

1. To enable **Time Intelligence**, mark a table as a **Date Table**:  
    - Right-click the date table > **Mark as Date Table**, selecting the finest-granularity field (`Date`).  
2. Examples:  
    1. Calculate Year-To-Date (YTD) sales:  
        ```  
        SalesCountYearToDate = CALCULATE([SalesCount], DATESYTD('Date'[Date]))  
        ```  
    2. Compare sales with the previous year:  
        ```  
        SalesCountLastPeriod = CALCULATE([SalesCount], SAMEPERIODLASTYEAR('Date'[Date]))  
        ```  
    3. Compute the performance difference:  
        ```  
        VersusLastPeriod = [SalesCount] - [SalesCountLastPeriod]  
        ```  

---

### Variables in Measures  

Using variables can optimize performance and enhance readability:  
1. Without variables:  
    ```  
    DummyTest = IF([VersusLastPeriod] > 0, [VersusLastPeriod], BLANK())  
    ```  
2. With variables:  
    ```  
    DummyTest =  
        VAR vTest = [VersusLastPeriod]  
    RETURN  
        IF(vTest > 0, vTest, BLANK())  
    ```  

--- 

### Calculation Groups  

Calculation Groups simplify measure management by organizing calculations like Time Intelligence functions into reusable groups. Details can be added upon request.  

--- 

### Row-Level Security (RLS)  

Row-Level Security controls data access at the user level. Additional guidance is available if needed.
