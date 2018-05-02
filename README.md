Applied Data science course material
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

``` r
library(DBI)
library(odbc)

driver = "SQL Server"
server = "fbmcsads.database.windows.net"
Database = "WideWorldImporters-Standard"
uid = "adatumadmin"
Pwd = "Pa55w.rdPa55w.rd"


con <- dbConnect(odbc(),
                 driver=driver,
                 server=server,
                 database=Database,
                 pwd=Pwd,
                 uid=uid
                 )
```
