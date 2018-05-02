Applied Data Science course material
================

Packages
--------

Packages we'll look at today:

-   odbc / readxl / readr / dbplyr for data access
-   tidyverse for data manipulation
-   DataExplorer for providing an overview of our data
-   modelr / rsamples for sampling strategy
-   recipes for performing feature engineering
-   glmnet / h2o / FFTrees for building models
-   yardstick / broom for evaluation
-   rmarkdown for documentation

Working with databases
----------------------

We need a database connection before we can do anything with our database. Add it by the below code or go o Connections &gt; New Connection. Inside the "New Connection" we can get the connection code.

``` r
library(DBI)
library(odbc)

Driver = "SQL Server"
Server = "fbmcsads.database.windows.net"
Database = "WideWorldImporters-Standard"
Uid = "adatumadmin"
Pwd = "Pa55w.rdPa55w.rd"


con <- dbConnect(odbc(),
                 driver=Driver,
                 server=Server,
                 database=Database,
                 pwd=Pwd,
                 uid=Uid
                 )
```

Now that we have a DB connection, we can write SQL in a code chunk. We use the above connection name "con".

``` sql
select top 5 * from flights
```

|  year|  month|  day|  dep\_time|  sched\_dep\_time|  dep\_delay|  arr\_time|  sched\_arr\_time|  arr\_delay| carrier |  flight| tailnum | origin | dest |  air\_time|  distance|  hour|  minute| time\_hour          |
|-----:|------:|----:|----------:|-----------------:|-----------:|----------:|-----------------:|-----------:|:--------|-------:|:--------|:-------|:-----|----------:|---------:|-----:|-------:|:--------------------|
|  2013|      1|    1|        517|               515|           2|        830|               819|          11| UA      |    1545| N14228  | EWR    | IAH  |        227|      1400|     5|      15| 2013-01-01 05:00:00 |
|  2013|      1|    1|        533|               529|           4|        850|               830|          20| UA      |    1714| N24211  | LGA    | IAH  |        227|      1416|     5|      29| 2013-01-01 05:00:00 |
|  2013|      1|    1|        542|               540|           2|        923|               850|          33| AA      |    1141| N619AA  | JFK    | MIA  |        160|      1089|     5|      40| 2013-01-01 05:00:00 |
|  2013|      1|    1|        544|               545|          -1|       1004|              1022|         -18| B6      |     725| N804JB  | JFK    | BQN  |        183|      1576|     5|      45| 2013-01-01 05:00:00 |
|  2013|      1|    1|        554|               600|          -6|        812|               837|         -25| DL      |     461| N668DN  | LGA    | ATL  |        116|       762|     6|       0| 2013-01-01 06:00:00 |

We can use bdplyr to construct dplyr commands that work on the DB.

``` r
library(tidyverse)
```

    ## -- Attaching packages -------------------------------------------------------------------------------- tidyverse 1.2.1 --

    ## v ggplot2 2.2.1     v purrr   0.2.4
    ## v tibble  1.4.2     v dplyr   0.7.4
    ## v tidyr   0.8.0     v stringr 1.3.0
    ## v readr   1.1.1     v forcats 0.2.0

    ## -- Conflicts ----------------------------------------------------------------------------------- tidyverse_conflicts() --
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

``` r
library(dbplyr) # tip: read the introduction to dbplyr package to know how to connect to db
```

    ## 
    ## Attaching package: 'dbplyr'

    ## The following objects are masked from 'package:dplyr':
    ## 
    ##     ident, sql

``` r
flights_tbl <- tbl(con, "flights")

flights_tbl %>% 
  filter(month<=6) %>% 
  group_by(origin) %>% 
  summarise(n = n(), 
            mean_dist= mean(distance)) %>% 
  show_query()
```

    ## Warning: Missing values are always removed in SQL.
    ## Use `AVG(x, na.rm = TRUE)` to silence this warning

    ## <SQL>
    ## SELECT "origin", COUNT(*) AS "n", AVG("distance") AS "mean_dist"
    ## FROM "flights"
    ## WHERE ("month" <= 6.0)
    ## GROUP BY "origin"

We can also work with tables that aren'nt in the default schema.

``` r
purchaseorders_tbl <- tbl(con, in_schema("purchasing", "purchaseorders")) # tip: see the documentation for in_schema in the dbplyr package to know hos to refer to a table in a schema. 

purchaseorders_tbl %>% 
  top_n(5)
```

    ## Selecting by LastEditedWhen

    ## # Source:   lazy query [?? x 12]
    ## # Database: Microsoft SQL Server
    ## #   12.00.0300[dbo@fbmcsads/WideWorldImporters-Standard]
    ##   PurchaseOrderID SupplierID OrderDate  DeliveryMethodID ContactPersonID
    ##             <int>      <int> <chr>                 <int>           <int>
    ## 1            2073          4 2016-05-31                7               2
    ## 2            2074          7 2016-05-31                2               2
    ## 3            2071          4 2016-05-30                7               2
    ## 4            2072          7 2016-05-30                2               2
    ## 5            2068          4 2016-05-27                7               2
    ## 6            2069          7 2016-05-27                2               2
    ## 7            2070          4 2016-05-28                7               2
    ## # ... with 7 more variables: ExpectedDeliveryDate <chr>,
    ## #   SupplierReference <chr>, IsOrderFinalized <lgl>, Comments <chr>,
    ## #   InternalComments <chr>, LastEditedBy <int>, LastEditedWhen <chr>

To write SQL updates, create views etc - it is better to use the DBI packages instead of dbplyr.

We can use the 'Id()' function from DBI to work with schema more generically within a database. This means we aren't restricted to just SELECT statements.

``` r
# Create a schema to work in - errors if already exists
dbGetQuery(con, "CREATE SCHEMA DBIexampleEVA") # creates my own schema in db
```

    ## Error: <SQL> 'CREATE SCHEMA DBIexampleEVA'
    ##   nanodbc/nanodbc.cpp:1587: 42S01: [Microsoft][ODBC SQL Server Driver][SQL Server]There is already an object named 'DBIexampleEVA' in the database.

``` r
# Write some data / drop & recreate the table if it exists already
dbWriteTable(con, "iris", iris, overwrite=TRUE) # overwrites the table
# Read from newly written table 
head(dbReadTable(con,"iris"))
```

    ##   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
    ## 1          5.1         3.5          1.4         0.2  setosa
    ## 2          4.9         3.0          1.4         0.2  setosa
    ## 3          4.7         3.2          1.3         0.2  setosa
    ## 4          4.6         3.1          1.5         0.2  setosa
    ## 5          5.0         3.6          1.4         0.2  setosa
    ## 6          5.4         3.9          1.7         0.4  setosa

``` r
# Read from a table in a schema
head(dbReadTable(con,Id(schema="20774A", table="CustomerTransactions")))
```

    ## Note: method with signature 'DBIConnection#SQL' chosen for function 'dbQuoteIdentifier',
    ##  target signature 'Microsoft SQL Server#SQL'.
    ##  "OdbcConnection#character" would also be valid

    ##                  CustomerName TransactionAmount OutstandingBalance
    ## 1             Aakriti Byrraju           2645.00                  0
    ## 2                  Bala Dixit            465.75                  0
    ## 3 Tailspin Toys (Head Office)            103.50                  0
    ## 4 Tailspin Toys (Head Office)            511.98                  0
    ## 5                Sara Huiting            809.60                  0
    ## 6                Alinne Matos            494.50                  0
    ##   TaxAmount PKIDDate TransactionDate
    ## 1    345.00 20130101      2013-01-01
    ## 2     60.75 20130101      2013-01-01
    ## 3     13.50 20130101      2013-01-01
    ## 4     66.78 20130101      2013-01-01
    ## 5    105.60 20130101      2013-01-01
    ## 6     64.50 20130101      2013-01-01

``` r
# If a write method is supported by the driver, this will work
dbWriteTable(con, Id( schema="DBIexampleEVA", table="iris", overwrite=TRUE))
```

    ## Error in (function (classes, fdef, mtable) : unable to find an inherited method for function 'dbWriteTable' for signature '"Microsoft SQL Server", "SQL", "missing"'

Some of our code could fail in that section so we used `error=TRUE` to be able to carry on even if some of the code errored. Great for optional cose or things with bad connection.

Explorative Data Analysis (EDA) using the DataExplorer package
--------------------------------------------------------------

High-level report to help us understand the data. Group discrete features into a "other" category. Make sure only to include the scheduled hour and not the

``` r
flights_tbl %>% 
  as_data_frame() %>% 
  DataExplorer::GenerateReport()
```

### Questions

Questions arising from the basic report:

1.  Why is there a day with double the number of flights?
2.  The date 31:st doesn't have as many flights as other days, do we need to adjust for this?
3.  Do we need to do anything about missings or can we just remove the rows?
4.  Why is there negative correlation between `flights` (flight number) and `distance`?

Things to implement later in the workflow due to the EDA: 1. We need to address the high correlation between time columns 2. We need to group low frequency airline carriers 3. Bivariate analysis

### Answering our questions

> Why is there a day with double the number of flights?

Are there duplicate rows?

``` r
flights_tbl %>% 
  filter(day==15) %>% 
  distinct() %>% 
  summarise(n()) %>% 
  as_data_frame()->
  distinct_count

flights_tbl %>% 
  filter(day==15) %>% 
  summarise(n())%>% 
  as_data_frame() ->
  row_count

identical(row_count,distinct_count)
```

If the identical() is TRUE, they are exactly equal, meaning that we not have any duplicate rows. But are the number of rows unusual?

``` r
library(ggplot2)
flights_tbl %>% 
  group_by(day) %>% 
  summarise(n=n(), n_distinct(flight)) %>% 
  as_data_frame() %>% 
  ggplot(aes(x=day, y=n)) + geom_col()
```

![](README_files/figure-markdown_github/unnamed-chunk-8-1.png) Looks like the jump in the histogram is an artifact with the data visualization binning data.

### Bivariate analysis

Next answer

> Bivariate analysis

``` r
flights_tbl %>% 
  select_if(is.numeric) %>% 
  as_data_frame() %>% 
  gather(col, val, -dep_delay) %>% # takes our wide data and turn it into long data, aka pivot all columns without the dep_delay.
  filter(col!="arr_delay",
         dep_delay<500) %>% 
  ggplot(aes(x=val, y=dep_delay)) +
  #  geom_point() +  # this will take long time since it is plotting row by row
  geom_bin2d() + 
   facet_wrap(~col, scales = "free") # take different parts of our data to produce them as charts
```

    ## Applying predicate on the first 100 rows

    ## Warning: Removed 1631 rows containing non-finite values (stat_bin2d).

    ## Warning: Computation failed in `stat_bin2d()`:
    ## 'from' must be a finite number

![](README_files/figure-markdown_github/unnamed-chunk-9-1.png) \#\# Sampling: Inbalanced data

### Theory/info

In our data model, 97% are delayed. This means that the model always will predict "delayed". We have some options to deal with this.

-   Undersampling (downsampling). Consequence: reducing data since we reduce the data to fit out imbalanced data.
-   Upsampling. Consequence: risks of overfitting. Doesn't reduce training data
-   Synthesising data makes extra records that are like the minority class (Doesn't reduce training set, Avoids some of the overfit risk of upsampling, Can weaken predictions if minority data is very similair to majority)

We need to think about whether we need to k-fold cross-validation explicitly.

-   Run the same model and assess robustness of coefficients
-   We have an algorithm that needs explicit cross validation because it doesn't do it internally
-   When we're going to run lots of models with hyper-parameter tuning do the results are more consistent

We use bootstrapping when we want t fit a single model and ensure the results are robust. This will often do many more iterations than k-fold cross validation, making it better in cases where there's relatively small amounts of data.

Packages we can use for sampling incude:

-   modelr which facilitates bootstrap and k-fold cross validation strategies
-   rsample allows us to bootstrap and perform a wide variety of cross validation tasks
-   recipes allows us to upsample and downsample
-   synthpop allows us to build synthesised samples

``` r
## The devtools are not updated in our cran verison, so we install them separate 
install.packages("devtools")
```

    ## Installing package into 'C:/Users/Admin/Documents/R/win-library/3.4'
    ## (as 'lib' is unspecified)

    ## package 'devtools' successfully unpacked and MD5 sums checked
    ## 
    ## The downloaded binary packages are in
    ##  C:\Users\Admin\AppData\Local\Temp\RtmpcnOiNn\downloaded_packages

``` r
devtools::install_github("topepo/recipes")
```

    ## Skipping install of 'recipes' from a github remote, the SHA1 (4fb3fe10) has not changed since last install.
    ##   Use `force = TRUE` to force installation

### Practical

``` r
flights_tbl %>% 
  as_data_frame() ->
  flights #insert our flights data to in-memory to be able to use the chosen packages

flights %>% 
  mutate(was_delayed=ifelse(arr_delay>5,"Delayed", "Not Delayed"),
         week= ifelse(day %/% 7 > 3, 3, day %/% 7 )) -> # here we adding columns that are not related to predicting variables, like changing time to hours 
  flights

flights %>% 
  modelr::resample_partition(c(train=0.7, test=0.3)) ->
  splits

splits %>% 
  pluck("train") %>% 
  as_data_frame()->
  train_raw

splits %>% 
  pluck("test") %>% 
  as_data_frame()->
  test_raw
```

During the investigation we will look at the impact of upsampling. We'll see it in action in a bit. First prepping our basic features!

#### Basic Feature Engineering

``` r
library(recipes)
```

    ## Loading required package: broom

    ## 
    ## Attaching package: 'recipes'

    ## The following object is masked from 'package:stringr':
    ## 
    ##     fixed

    ## The following object is masked from 'package:stats':
    ## 
    ##     step

``` r
basic_fe <- recipe(train_raw, was_delayed~. ) # we use was_delayed as predictor and everything else as variables

basic_fe %>% 
  step_rm(ends_with("time"), ends_with("delay"), tailnum, flight, minute, time_hour, day) %>%  # remove some variables
 # step_corr(all_predictors()) %>% # remove highly correlates with other variables
  step_zv(all_predictors()) %>% 
  step_nzv(all_predictors()) %>% 
  step_naomit(all_predictors()) %>% # remove rows that have missing values, since only 3 perc are missing
  step_naomit(all_outcomes()) %>% 
  step_other(all_nominal(), threshold = 0.03)-> # will pool infrequently occuring values into an "other category" 
  colscleaned_fe

colscleaned_fe
```

    ## Data Recipe
    ## 
    ## Inputs:
    ## 
    ##       role #variables
    ##    outcome          1
    ##  predictor         20
    ## 
    ## Operations:
    ## 
    ## Delete terms ends_with("time"), ends_with("delay"), ...
    ## Zero variance filter on all_predictors()
    ## Sparse, unbalanced variable filter on all_predictors()
    ## Removing rows with NA values in all_predictors()
    ## Removing rows with NA values in all_outcomes()
    ## Collapsing factor levels for all_nominal()

``` r
colscleaned_fe <- prep(colscleaned_fe, verbose = TRUE) # if error, type verbose=TRUE to print more error info
```

    ## oper 1 step rm [training] 
    ## oper 2 step zv [training] 
    ## oper 3 step nzv [training] 
    ## oper 4 step naomit [training] 
    ## oper 5 step naomit [training] 
    ## oper 6 step other [training]

``` r
colscleaned_fe
```

    ## Data Recipe
    ## 
    ## Inputs:
    ## 
    ##       role #variables
    ##    outcome          1
    ##  predictor         20
    ## 
    ## Training data contained 235743 data points and 6527 incomplete rows. 
    ## 
    ## Operations:
    ## 
    ## Variables removed dep_time, sched_dep_time, arr_time, ... [trained]
    ## Zero variance filter removed year [trained]
    ## Sparse, unbalanced variable filter removed no terms [trained]
    ## Removing rows with NA values in all_predictors()
    ## Removing rows with NA values in all_outcomes()
    ## Collapsing factor levels for carrier, origin, dest, was_delayed [trained]

``` r
train_prep1<-bake(colscleaned_fe, train_raw) # use the recipe to clean the train dataset
```

Now we need to process our numeric variables.

``` r
colscleaned_fe %>% 
  step_log(distance) %>% 
  step_num2factor(month, week, hour) ->
  numscleaned_fe

numscleaned_fe
```

    ## Data Recipe
    ## 
    ## Inputs:
    ## 
    ##       role #variables
    ##    outcome          1
    ##  predictor         20
    ## 
    ## Training data contained 235743 data points and 6527 incomplete rows. 
    ## 
    ## Operations:
    ## 
    ## Variables removed dep_time, sched_dep_time, arr_time, ... [trained]
    ## Zero variance filter removed year [trained]
    ## Sparse, unbalanced variable filter removed no terms [trained]
    ## Removing rows with NA values in all_predictors()
    ## Removing rows with NA values in all_outcomes()
    ## Collapsing factor levels for carrier, origin, dest, was_delayed [trained]
    ## Log transformation on distance
    ## Factor variables from month, week, hour

``` r
numscleaned_fe <- prep(numscleaned_fe, verbose = TRUE)
```

    ## oper 1 step rm [pre-trained]
    ## oper 2 step zv [pre-trained]
    ## oper 3 step nzv [pre-trained]
    ## oper 4 step naomit [pre-trained]
    ## oper 5 step naomit [pre-trained]
    ## oper 6 step other [pre-trained]
    ## oper 7 step log [training] 
    ## oper 8 step num2factor [training]

``` r
numscleaned_fe
```

    ## Data Recipe
    ## 
    ## Inputs:
    ## 
    ##       role #variables
    ##    outcome          1
    ##  predictor         20
    ## 
    ## Training data contained 235743 data points and 6527 incomplete rows. 
    ## 
    ## Operations:
    ## 
    ## Variables removed dep_time, sched_dep_time, arr_time, ... [trained]
    ## Zero variance filter removed year [trained]
    ## Sparse, unbalanced variable filter removed no terms [trained]
    ## Removing rows with NA values in all_predictors()
    ## Removing rows with NA values in all_outcomes()
    ## Collapsing factor levels for carrier, origin, dest, was_delayed [trained]
    ## Log transformation on distance [trained]
    ## Factor variables from month, week, hour [trained]

``` r
train_prep1<-bake(numscleaned_fe, train_raw)
```

#### Time for upsampling!

``` r
numscleaned_fe %>% 
  step_upsample(all_outcomes(), ratio = 1) %>% 
  prep(retain=TRUE) %>% 
  juice() %>% 
  bake(numscleaned_fe,.)-> # upsample to 50/50 by repeating the rows with delays 
  train_prep2
```

Building models
---------------
