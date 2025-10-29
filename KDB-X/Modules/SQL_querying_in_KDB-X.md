# Using SQL in KDB-X

SQL provides a familiar relational interface for querying and managing data in KDB-X, allowing users to leverage standard SQL syntax within the q environment. 
The tutorial will walk through some examples of how to use SQL to interact with and manipulate q objects, and then show how to utilise the power of q functionality through SQL. 

Note: SQL is currently offered as part of core but may be embedded as a module in the future.

## 1. Prerequisites

View the [KDB-X Docs](https://docs.kx.com/public-preview/kdb-x/home.htm) for full details on KDB-X and KDB-X Python.

1. Requires KDB-X to be installed, you can sign up [here](https://developer.kx.com/products/kdb-x/install). For full install instructions see: [KDB-X Install](https://docs.kx.com/public-preview/kdb-x/Get_Started/kdb-x-install.htm).
2. Ensure you have the necessary dataset

## 2. Loading and Preparing the Data

Launch a q session by running `q`:
```
Welcome to KDB-X Community Edition!
For Community support, please visit https://kx.com/slack
Tutorials can be found at https://github.com/KxSystems/tutorials
Ready to go beyond the Community Edition? Email preview@kx.com
q)
```
Load the dataset into a q table:
```
// The \z system command sets the format for date parsing 
\z 1

fvprices:("SSSDSF"; enlist ",") 0: `$":/src/wholesaleproduceprices.csv"
```
In the above:
•	0: is used to read the csv file
•	The schema " SSSDSF" specifies the datatypes for each column
•	The "," ensures the CSV is parsed using commas as delimiters.
Now the dataset has been loaded, we can inspect the table using first to see what the first row of the table looks like:
```
first fvprices;

category| `fruit
item    | `apples
variety | `bramleys_seedling
date    | 2025.10.13
unit    | `kg
price   | 1.27
```

The table above is a simple table giving information on average wholesale prices of selected home-grown horticultural produce in England and Wales. It is updated every fortnight.
Source: https://www.gov.uk/government/statistical-data-sets/wholesale-fruit-and-vegetable-prices-weekly-average.

In this tutorial, we are going to query a very simple dataset using both q and SQL.  
This is particularly useful if you have a q object that you need to interrogate, but your skillset is more suited to SQL than the q language.

## 3. Use SQL to interrogate a q table

### 3.1 Common query types in SQL 

There are a few different ways to query using SQL within a q session.Use s) and then standard SQL code to return all rows of the table 
```
s) SELECT * FROM fvprices
```
```
category  item                 variety                date       unit price
---------------------------------------------------------------------------
fruit     apples               bramleys_seedling      2025.10.13 kg   1.27
fruit     apples               coxs_orange_group      2025.10.13 kg   1.22
fruit     apples               egremont_russet        2025.10.13 kg   1.46
fruit     apples               braeburn               2025.10.13 kg   1.38
fruit     apples               gala                   2025.10.13 kg   1.23
fruit     blackberries         blackberries           2025.10.13 kg   15.35
fruit     blueberries          blueberries            2025.10.13 kg   12.19
fruit     pears                conference             2025.10.13 kg   1.22
fruit     pears                doyenne_du_comice      2025.10.13 kg   1.17
fruit     plums                all_other              2025.10.13 kg   1.52
fruit     raspberries          raspberries            2025.10.13 kg   13.16
fruit     strawberries         strawberries           2025.10.13 kg   4.08
vegetable beans                dwarf_french_or_kidney 2025.10.13 kg   1.87
vegetable beans                runner_climbing        2025.10.13 kg   3.92
vegetable beetroot             beetroot               2025.10.13 kg   0.7
vegetable brussels_sprouts     brussels_sprouts       2025.10.13 kg   1
..
```

However, you don’t have to capitalise key words like SELECT and FROM
```
s) select * from fvprices where category = 'vegetable' 
```
```
category  item                 variety                date       unit price
---------------------------------------------------------------------------
vegetable beans                dwarf_french_or_kidney 2025.10.13 kg   1.87
vegetable beans                runner_climbing        2025.10.13 kg   3.92
vegetable beetroot             beetroot               2025.10.13 kg   0.7
vegetable brussels_sprouts     brussels_sprouts       2025.10.13 kg   1
vegetable pak_choi             pak_choi               2025.10.13 kg   3.5
vegetable curly_kale           curly_kale             2025.10.13 kg   4.19
vegetable cabbage              red                    2025.10.13 kg   0.54
vegetable cabbage              savoy                  2025.10.13 head 0.64
vegetable spring_greens        prepacked              2025.10.13 kg   1.39
vegetable cabbage              summer_autumn_pointed  2025.10.13 kg   0.79
vegetable cabbage              white                  2025.10.13 kg   0.52
vegetable cabbage              round_green_other      2025.10.13 head 0.62
..
```

Another way to query using SQL is to use .s.e and wrap your SQL code within quotation marks using `.s.e`

```
.s.e "SELECT category, COUNT(item) AS count_per_category FROM fvprices GROUP BY category"
```
```
category    count_per_category
------------------------------
cut_flowers 423
fruit       3521
pot_plants  49
vegetable   12673
```

As you can see, common SQL aggregations and calculations are supported in KDB-X.

Find the avg price of each type of item:
```
.s.e "SELECT category,item,unit,avg(price) AS avg_price FROM fvprices GROUP BY category,item ORDER BY avg_price DESC"
```
```
category    item                 unit avg_price
-----------------------------------------------
vegetable   asparagus            kg   10.35726
vegetable   watercress           kg   9.374348
fruit       raspberries          kg   8.969415
fruit       blueberries          kg   8.801304
fruit       blackberries         kg   8.368394
fruit       currants             kg   8.165982
fruit       gooseberries         kg   6.566883
fruit       cherries             kg   5.314222
vegetable   rocket               kg   5.295729
vegetable   mixed_babyleaf_salad kg   5.292558
..
```

### 3.2 Defining functions

Functions can be used in a number of different ways using both SQL and q. 

You can define a function to return a table in SQL and then query the result of that function:
```
x:.s.e"SELECT item, variety, unit, avg(price) as avg_price FROM fvprices WHERE price>15.00 GROUP BY variety";

.s.e "select item, avg_price from x";
```
```
item         avg_price
----------------------
cherries     16.08
asparagus    17.20381
currants     17.16
blackberries 16.342
gooseberries 16.7075
raspberries  16.57
currants     15.82
```
### 3.3 Using parameters
To parameterize your function, you can use `.s.sp`. The function expects a list of parameters so in the case where you only need one parameter, you can use `enlist`.
```
.s.sp["select item,variety, price from fvprices where category=$1"](enlist `fruit);
```
```
.s.sp["select item,variety, price from fvprices where category=$1 and price<$2"](`fruit;2.00);
```

To define a function that you will need to use repeatedly with different parameters, you can use `.s.sq` to define the function, giving null values for the parameters:
```
query:.s.sq["select * from fvprices where item=$1 and price<$2"](`;0n);
```

And then use `.s.sx` to call that function with whichever parameters you need:
```
.s.sx[query](`apples;0.6)
```
```
.s.sx[query](`beans;1.4)
```

## 4. Integrate with q

Sometimes, it may be easier to use a function that exists in the q language. 

In the code below, we are looking at an example of creating a pivot table – something that is relatively straightforward and simple to do in just a few lines of q code. 
```
pivotprices:{[]
//Remove the “.” from the column names (dates) because SQL does not like dots in column names
pp:update date:`$"_" sv '"." vs' string[date] from select variety, price, date from fvprices; 
//Pull out the date column into distinct values that will become individual columns in the pivot table
D:asc exec distinct date from pp;
//Create the dictionary mappings
B: exec D#(date!price) by variety:variety from pp;
//Add the remaining columns back into the table
A:select variety, category,item,unit from fvprices; 
A lj B
};
pivotprices[]
```
We can then call that q function from within the SQL code
```
s) select * from qt('{pivotprices[]}[]')
```
