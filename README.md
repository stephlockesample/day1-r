Our R project
================

Packages
--------

Packages we'll look at today:

-   odbc, readxl, readr, dbplyr for data access
-   tidyverse for data manipulation
-   DataExplorer for prodiving of our data
-   modelr, rsamples for sampling
-   recipes for performing feature engineering
-   glmnet, h2o, FFTrees for building models
-   yardstick, broom for evaluation
-   rmarkdown for documentation

Working with databases
----------------------

``` r
library(DBI)
library(odbc)

driver = "SQL Server" 
server = "fbmcsads.database.windows.net"
database = "WideWorldImporters-Standard"
uid = "adatumadmin"
pwd = "Pa55w.rdPa55w.rd"

con<-dbConnect(odbc(),
               driver = driver, 
               server = server,
               database = database,
               uid = uid,
               pwd = pwd)
```
