# Data Camp Course: PostgreSQL summary statgs - window functions

## Chapter 1. Introduction to window functions

Window function:
- Perform an operation across a set of rows that are somehow related to the current row
- Similar to GROUP BY aggregate functions, but all rows remain in the output
- Usually used for
    - fetching values from preceding or following rows (e.g. to calculate growth over time)
    - Assigning ordinal rank (1st, 2nd, etc) to rows based on their values' positions in a sorted list
    - Running totals, Moving averages


#### Enter ROW_NUMBER
```sh
SELECT ROW_NUMBER() OVER() AS Row_N FROM Summer_Medals WHERE Medal = 'Gold';
```

Note that OVER() clause indicates that ROW_NUMBER() is a windows function. Anotamy of windows function is: FUNCTION_NAME() OVER (). Paranthesis after OVER() function can be empty, but can contain subclauses like ORDER BY, PARTITION BY, ROWS/RANGE PRECEDING/FOLLOWING/UNBOUNDED. These function changes the output of the windows function.

#### Assignment: Numbering Olympic games in ascending order
The Summer Olympics dataset contains the results of the games between 1896 and 2012. The first Summer Olympics were held in 1896, the second in 1900, and so on. What if you want to easily query the table to see in which year the 13th Summer Olympics were held? You'd need to number the rows for that.
```sh
SELECT
  Year,
  -- Assign numbers to each year
  ROW_NUMBER() OVER () AS Row_N
FROM (
  SELECT DISTINCT Year
  FROM Summer_Medals
  ORDER BY Year ASC
) AS Years
ORDER BY Year ASC;
```

#### ORDER BY
You can use ORDER BY subclause in the OVER() clause to make condition on the ROW_NUMBER().
Here is example of Ordering in- and outside OVER(). Notice that ORDER BY inside OVER takes effect before ORDER BY outside OVER.
```sh
SELECT
  Year, Event, Country
  ROW_NUMBER() OVER (ORDER BY Year DESC, Event ASC) AS Row_N
FROM Summer_Medals
WHERE Medal = 'Gold'
ORDER BY Country ASC, Row_N ASC;
```

Another example of numbering Olympic athletes by medals earned: Row numbering can also be used for ranking. For example, numbering rows and ordering by the count of medals each athlete earned in the OVER clause will assign 1 to the highest-earning medalist, 2 to the second highest-earning medalist, and so on.

```sh
WITH Athlete_Medals AS (
  SELECT
    -- Count the number of medals each athlete has earned
    Athlete,
    COUNT(*) AS Medals
  FROM Summer_Medals
  GROUP BY Athlete)

SELECT
  -- Number each athlete by how many medals they have earned
  Athlete,
  ROW_NUMBER() OVER (ORDER BY Medals DESC) AS Row_N
FROM Athlete_Medals
ORDER BY Medals DESC;
```

#### LAG
Example of Reigning Champion: A reigning champion is a champion who's won both the previous and current years' competitions. The previous and current year's champions need to be in the same row (in two different columns). How do we do that? Using LAG!

LAG(column, n) OVER(...) return column's value at the row n rows before the current row.

```sh
WITH Weightlifting_Gold AS (
  SELECT
    -- Return each year's champions' countries
    Year,
    Country AS champion
  FROM Summer_Medals
  WHERE
    Discipline = 'Weightlifting' AND
    Event = '69KG' AND
    Gender = 'Men' AND
    Medal = 'Gold')

SELECT
  Year, Champion,
  -- Fetch the previous year champion
  LAG(Champion) OVER
    (ORDER BY Year ASC) AS Last_Champion
FROM Weightlifting_Gold
ORDER BY Year ASC;
```

#### PARTITION BY
PARTITION BY splits the table into partitions based on a column's unique values. Unlike GROUP BY, the results are not rolled into one column. 
    
Partitions are operated separately by the window function. (1) ROW_NUMBER will reset for each partition. (2) LAG will only fetch a row previous value if its previous row is in the same partition. 

Example of Reigning Champion by gender: You've already fetched the previous year's champion for one event. However, if you have multiple events, genders, or other metrics as columns, you'll need to split your table into partitions to avoid having a champion from one event or gender appear as the previous champion of another event or gender.

```sh
WITH Tennis_Gold AS (
  SELECT DISTINCT
    Gender, Year, Country
  FROM Summer_Medals
  WHERE
    Year >= 2000 AND
    Event = 'Javelin Throw' AND
    Medal = 'Gold')

SELECT
  Gender, Year,
  Country AS Champion,
  -- Fetch the previous year champion by gender
  LAG(Country) OVER (PARTITION BY Gender
                         ORDER BY Year ASC) AS Last_Champion
FROM Tennis_Gold
ORDER BY Gender ASC, Year ASC;
```

Adding complexity, Reigning champions by gender and event: In the previous exercise, you partitioned by gender to ensure that data about one gender doesn't get mixed into data about the other gender. If you have multiple columns, however, partitioning by only one of them will still mix the results of the other columns.
```sh
WITH Athletics_Gold AS (
  SELECT DISTINCT
    Gender, Year, Event, Country
  FROM Summer_Medals
  WHERE
    Year >= 2000 AND
    Discipline = 'Athletics' AND
    Event IN ('100M', '10000M') AND
    Medal = 'Gold')

SELECT
  Gender, Year, Event,
  Country AS Champion,
  -- Fetch the previous year champion by gender and event
  LAG(Country) OVER (PARTITION BY Gender, Event
                         ORDER BY Year ASC) AS Last_Champion
FROM Athletics_Gold
ORDER BY Event ASC, Gender ASC, Year ASC;
```

## Chapter 2. Fetching, Ranking, Paging

#### 4 Fetching functions: 

Relative fetching
* LAG(column, n) returns column value at the row n rows before the current row
* LEAD(column, n) returns column value at the row n rows after the current row

Absolute fetching
* FIRST_VALUE(column) returns the first value in the table or partition
* LAST_VALUE(column) returns the last value in the table or partition

We have used the LAG(...) clause before; similary LEAD(...) can be used. Fetching functions allow you to get values from different parts of the table into one row. If you have time-ordered data, you can "peek into the future" with the LEAD fetching function. This is especially useful if you want to compare a current value to a future value.
```sh
WITH Discus_Medalists AS (
  SELECT DISTINCT
    Year,
    Athlete
  FROM Summer_Medals
  WHERE Medal = 'Gold'
    AND Event = 'Discus Throw'
    AND Gender = 'Women'
    AND Year >= 2000)

SELECT
  -- For each year, fetch the current and future medalists
  Year,
  Athlete,
  LEAD(Athlete, 3) OVER (ORDER BY Year ASC) AS Future_Champion
FROM Discus_Medalists
ORDER BY Year ASC;
```


Absolute fetching is different from Relative fetching and need explanation using examples. 

Example of FIRST_VALUE(): first athlete by name
```sh
WITH All_Male_Medalists AS (
  SELECT DISTINCT
    Athlete
  FROM Summer_Medals
  WHERE Medal = 'Gold'
    AND Gender = 'Men')

SELECT
  -- Fetch all athletes and the first althete alphabetically
  Athlete,
  FIRST_VALUE(Athlete) OVER (
    ORDER BY Athlete ASC
  ) AS First_Athlete
FROM All_Male_Medalists;
```

Example of LAST_VALUE(): last country by name
```sh
WITH Hosts AS (
  SELECT DISTINCT Year, City
    FROM Summer_Medals)

SELECT
  Year,
  City,
  -- Get the last city in which the Olympic games were held
  LAST_VALUE(City) OVER (
   ORDER BY Year ASC
   RANGE BETWEEN
     UNBOUNDED PRECEDING AND
     UNBOUNDED FOLLOWING
  ) AS Last_City
FROM Hosts
ORDER BY Year ASC;
```

By default, a window starts at the beginning of the table or partition and ends at the current row. RANGE BETWEEN ... clause extends the window to the end of the table or partition. So last value can be fetched for LAST_VALUE() clause. If you don't use RANGE BETWEEN ... clause in above example then the Last_City column will be same as City column.

#### 3 Ranking function: 
Rank rows according to their values.
* ROW_NUMBER() always assigns unique numbers, even if two rows' values are the same
* RANK() assigns the same number to rows with identical values, skipping over the next numbers in such cases
* DENSE_RANK() also assigns the same number to rows with identical values, but doesn't skip over the next numbers



## Chapter 3. Aggregate window functions and frames













## Chapter 4. Beyond window functions


