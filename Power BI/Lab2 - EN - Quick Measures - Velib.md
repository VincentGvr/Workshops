# Create a Measure Table  

1. From the portal, add a table: _Home > Enter Data_  
2. Add a column with a single row (no specific value). Name the table _Calculations_ and validate.  
3. In this table, create your first measure: Right-click on the _Calculations_ table > New Measure > In the formula bar, enter _Today = TODAY()_ > Press Enter.  
4. Delete the column in the _Calculations_ table. The table should appear at the top of the list of tables, and its icon should change to a calculator.  

# Create Your First Measures  

1. Number of available bikes
2. Number of available mechanical bikes
3. Average number of bikes per station
4. Number of stations
5. Boolean (True/False) indicating whether electric bikes are available (True) or not (False) at the station
6. Number of empty stations
7. Number of stations with large capacity (more than 35)
8. Number of full stations
9. Percentage of full stations
   - To perform a division, use the `DIVIDE(A, B)` function instead of `/`, as `DIVIDE` handles division by zero and reduces latencies (CallBack).  
   - To display the rate as a percentage, click on the measure and, in the _Measure Tools_ tab, click "%".  
   - ![image](https://github.com/user-attachments/assets/c2d915fb-0721-4446-86c3-42540c9fe64c)
10. Station fill rate (number of available bikes compared to capacity) 
11. Percentage of the station's capacity compared to the total available capacity  
12. Station capacity rank

# Answers  

1. ```Number of available bikes = SUM(Etats[Total Available Bikes])```
2. ```Number of available mechanical bikes = SUM(Etats[Mechanical Bikes Available])```
3. ```Average bikes per station = AVERAGE(Etats[Total Available Bikes])```
4. ```Number of stations = DISTINCTCOUNT(Etats[Station Code])```
5. ```Mechanical bikes available tag = IF([Number of available mechanical bikes] > 0, true, false)```
6. ```  
   Number of empty stations = CALCULATE(  
       [Number of stations],  
       'Etats'[Total Available Bikes] = 0  
   )  
   ```
7. ```
    Number of stations with large capacity = CALCULATE(  
       [Number of stations],  
       Stations[Capacity] > 35
    )  
   ```  
8. ```  
   Number of full stations = CALCULATE(  
       [Number of stations],  
       'Etats'[Total Available Docks] = 0,  
       'Etats'[Total Available Bikes] <> 0  
   )  
   ```
9. ```  
   Percentage of full stations = DIVIDE(  
       [Number of full stations],  
       [Number of stations]  
   )  
   ```  
10. ```
    Station fill rate = DIVIDE(  
       [Number of available bikes],  
       SUM(Stations[Capacity])
    )  
   ```
11. ```
   Capacity % of total = DIVIDE(  
      SUM(Stations[Capacity]),  
      CALCULATE(SUM(Stations[Capacity]), ALL(Stations))  
    )  
    ```
12. ```
    Station Capacity = MAX(Stations[Capacity])
    Station Capacity Rank = RANKX(ALL(Stations), [Station Capacity], , , DENSE)  
   ```

# What-If Scenario  

1. Create a table with a list of values from 0 to 10.  
2. This table, without any relationship, is referred to as a "disconnected" table.  
3. Create a measure `AvailabilityScenario` that retrieves the value selected by the user:  
   1. Use the `MIN()`, `MAX()`, or `SELECTEDVALUE()` function.  
4. Create a measure that subtracts the `AvailabilityScenario` value from the number of available bikes.  
5. Create a slicer with the column from the disconnected table as the choice.  
6. In a matrix, display the station names, the number of available bikes, and the new measure created in step 4. Play with the slicer to test the measure.  
