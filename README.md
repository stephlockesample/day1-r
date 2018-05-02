Our R project
================

Packages
--------

Packages we'll look at today:

-   odbc, readxl, readr, dbplyr for data access
-   tidyverse for data manipulation
-   DataExplorer for providing automated EDA of our data
-   modelr, rsamples for sampling
-   recipes for performing feature engineering
-   glmnet, h2o, FFTrees for building models
-   yardstick, broom for evaluation
-   rmarkdown for documentation

Working with databases
----------------------

We need a database connection before we can do anything with our database.

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

Now that we have a DB connection, we can write SQL in a code chunk.

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

We can use dbplyr to construct dplyr commands that work on the DB.

``` r
library(tidyverse)
```

    ## -- Attaching packages ---------------------------------------------------------- tidyverse 1.2.1 --

    ## v ggplot2 2.2.1     v purrr   0.2.4
    ## v tibble  1.4.2     v dplyr   0.7.4
    ## v tidyr   0.8.0     v stringr 1.3.0
    ## v readr   1.1.1     v forcats 0.2.0

    ## -- Conflicts ------------------------------------------------------------- tidyverse_conflicts() --
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

``` r
library(dbplyr)
```

    ## 
    ## Attaching package: 'dbplyr'

    ## The following objects are masked from 'package:dplyr':
    ## 
    ##     ident, sql

``` r
flights_tbl<-tbl(con, "flights")

flights_tbl %>% 
  filter(month<=6) %>% 
  group_by(origin) %>% 
  summarise(n = n(), 
            mean_dist= mean(distance)) %>% 
  show_query()
```

    ## <SQL>
    ## SELECT "origin", COUNT(*) AS "n", AVG("distance") AS "mean_dist"
    ## FROM "flights"
    ## WHERE ("month" <= 6.0)
    ## GROUP BY "origin"

We can also work with tables that aren't in the default schema.

``` r
purchaseorders_tbl<-tbl(con, in_schema("purchasing","purchaseorders"))

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

We can use the `Id()` function from DBI to work with schema more generically within a database. This means we aren't restricted to just SELECT statements.

``` r
# Create a schema to work in - errors if already exists
dbGetQuery(con,"CREATE SCHEMA DBIexample")
```

    ## Error: <SQL> 'CREATE SCHEMA DBIexample'
    ##   nanodbc/nanodbc.cpp:1587: 42S01: [Microsoft][ODBC SQL Server Driver][SQL Server]There is already an object named 'DBIexample' in the database.

``` r
# Write some data - drop & recreate the table if it exists already
dbWriteTable(con, "iris", iris, overwrite=TRUE) 
# Read from newly written table
head(dbReadTable(con, "iris"))
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
head(dbReadTable(con, Id(schema="20774A",table="CustomerTransactions")))
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
dbWriteTable(con, Id(schema="DBIexample", table="iris"), iris, overwrite=TRUE)
```

    ## Error in (function (classes, fdef, mtable) : unable to find an inherited method for function 'dbWriteTable' for signature '"Microsoft SQL Server", "SQL", "missing"'

Some of our code could fail in that section so we used `error=TRUE` to be able to carry on even if some of the code errored. Great for optional code or things with bad connections.

Exploratory Data Analysis
-------------------------

``` r
flights_tbl %>% 
  as_data_frame() %>% 
  DataExplorer::GenerateReport()
```

Questions arising from the basic report:

1.  Why is there a day with double the number of flights?
2.  Why is there negative correlation between `flight` (flight number) and `distance`?
3.  Do we need to anything about missings or can we just remove the rows?

Things to implement later in the workflow due to the EDA:

1.  We need to address the high correlation between time columns
2.  We need to group low frequency airline carriers
3.  Bivariate analysis

### Answering our questions

> Why is there a day with double the number of flights?

Are there duplicate rows?

``` r
flights_tbl %>% 
  filter(day==15) %>% 
  distinct() %>% 
  summarise(n()) %>% 
  as_data_frame() ->
  distinct_count

flights_tbl %>% 
  filter(day==15) %>% 
  summarise(n())%>% 
  as_data_frame() ->
  row_count

identical(row_count, distinct_count)
```

    ## [1] TRUE

But are the number of rows unusual?

``` r
library(ggplot2)
flights_tbl %>% 
  group_by(day) %>% 
  summarise(n=n(), n_unique=n_distinct(flight)) %>% 
  as_data_frame() %>% 
  ggplot(aes(x=day, y=n)) +
    geom_col()
```

![](README_files/figure-markdown_github/unnamed-chunk-8-1.png) Looks like the jump in the histogram is an artifact of binning the data. d'oh!

### Bivariate analysis

``` r
flights_tbl %>% 
  select_if(is.numeric) %>% 
  as_data_frame() %>% 
  gather(col, val, -dep_delay) %>% 
  filter(col!="arr_delay",
         dep_delay<500) %>% 
  ggplot(aes(x=val, y=dep_delay)) +
    geom_bin2d() +
    facet_wrap(~col, scales = "free")+
    scale_fill_gradientn(colours = viridisLite::viridis(256, option = "D"))
```

    ## Applying predicate on the first 100 rows

    ## Warning: Removed 1631 rows containing non-finite values (stat_bin2d).

    ## Warning: Computation failed in `stat_bin2d()`:
    ## 'from' must be a finite number

![](README_files/figure-markdown_github/unnamed-chunk-9-1.png)

Sampling
--------

### Theory / Info

Our options for sampling data with a large class imbalance are:

-   Downsampling takes as many majority rows and there are minority rows
    -   No overfit from individual rows
    -   Can drastically reduce training data size
-   Upsampling or over sampling repeats minority rows until they meet some defined class ratio
    -   Risks overfitting
    -   Doesn't reduce training data set
-   Synthesising data makes extra records that are like the minority class
    -   Doesn't reduce training set
    -   Avoids some of the overfit risk of upsampling
    -   Can weaken predictions if minority data is very similar to majority

We need to think about whether we need to k-fold cross-validation explicitly.

-   Run the same model and assess robustness of coefficients
-   We have an algorithm that needs explicit cross validation because it doesn't do it internally
-   When we're going to run lots of models with hyper-parameter tuning so the results are more consistent

We use bootstrapping when we want to fit a single model and ensure the results are robust. This will often do many more iterations than k-fold cross validation, making it better in cases where there's relatively small amounts of data.

Packages we can use for sampling include:

-   modelr which facilitates basic, bootstrap, and k-fold crossvalidation strategies
-   rsample allows us to bootstrap and perform a wide variety of crossvalidation tasks
-   recipes allows us to upsample and downsample
-   synthpop allows us to build synthesised samples

### Practical

First we need to split our data into test and train.

``` r
flights_tbl %>% 
  as_data_frame() ->
  flights

flights %>% 
  mutate(was_delayed= ifelse(arr_delay>5,"Delayed", "Not Delayed"),
         week = ifelse(day %/% 7 > 3, 3, day %/% 7 )) ->
  flights

flights %>%   
  modelr::resample_partition(c(train=0.7,test=0.3)) ->
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

During the investigation, we'll look at the impact of upsampling. We'll see it in action in a bit. First prepping our basic features!

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
basic_fe <- recipe(train_raw, was_delayed ~ .)

basic_fe %>% 
  step_rm(ends_with("time"), ends_with("delay"),tailnum, flight,
          minute, time_hour, day) %>% 
  step_naomit(all_predictors()) %>% 
  step_naomit(all_outcomes()) %>% 
  step_zv(all_predictors()) %>% 
  step_nzv(all_predictors()) %>% 
  step_other(all_nominal(),threshold = 0.03)  ->
  colscleaned_fe

colscleaned_fe <- prep(colscleaned_fe, verbose = TRUE)
```

    ## oper 1 step rm [training] 
    ## oper 2 step naomit [training] 
    ## oper 3 step naomit [training] 
    ## oper 4 step zv [training] 
    ## oper 5 step nzv [training] 
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
    ## Training data contained 235743 data points and 6599 incomplete rows. 
    ## 
    ## Operations:
    ## 
    ## Variables removed dep_time, sched_dep_time, arr_time, ... [trained]
    ## Removing rows with NA values in all_predictors()
    ## Removing rows with NA values in all_outcomes()
    ## Zero variance filter removed year [trained]
    ## Sparse, unbalanced variable filter removed no terms [trained]
    ## Collapsing factor levels for carrier, origin, dest, was_delayed [trained]

``` r
train_prep1<-bake(colscleaned_fe, train_raw)
```

Now we need to process our numeric variables.

``` r
colscleaned_fe  %>% 
  step_log(distance) %>% 
  step_num2factor(month, week, hour) %>% 
  step_rm(tailnum, dest) -> #hack!
  numscleaned_fe

numscleaned_fe <- prep(numscleaned_fe, verbose = TRUE)
```

    ## oper 1 step rm [pre-trained]
    ## oper 2 step naomit [pre-trained]
    ## oper 3 step naomit [pre-trained]
    ## oper 4 step zv [pre-trained]
    ## oper 5 step nzv [pre-trained]
    ## oper 6 step other [pre-trained]
    ## oper 7 step log [training] 
    ## oper 8 step num2factor [training] 
    ## oper 9 step rm [training]

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
    ## Training data contained 235743 data points and 6599 incomplete rows. 
    ## 
    ## Operations:
    ## 
    ## Variables removed dep_time, sched_dep_time, arr_time, ... [trained]
    ## Removing rows with NA values in all_predictors()
    ## Removing rows with NA values in all_outcomes()
    ## Zero variance filter removed year [trained]
    ## Sparse, unbalanced variable filter removed no terms [trained]
    ## Collapsing factor levels for carrier, origin, dest, was_delayed [trained]
    ## Log transformation on distance [trained]
    ## Factor variables from month, week, hour [trained]
    ## Variables removed tailnum, dest [trained]

``` r
train_prep1<-bake(numscleaned_fe, train_raw)
```

W00t it's upsampling time!

``` r
numscleaned_fe %>% 
  step_upsample(all_outcomes(), ratio=1) %>% 
  prep(retain=TRUE) %>% 
  juice() %>% 
  # hack because juice isn't reducing the column set
  bake(numscleaned_fe, .) ->
  train_prep2
```

Building models
---------------

Decide which types of models you want to consider -- perhaps using Microsoft's lovely [cheat sheet](https://docs.microsoft.com/en-gb/azure/machine-learning/studio/algorithm-cheat-sheet). Then determine if any need any special processing to the data beyond what you've done so far.

### A basic logistic regression

We can use generalised linear model functionality to construct a logistic regression.

``` r
glm_unbal<- glm(was_delayed ~ . -1 , "binomial", data = train_prep1)
glm_bal  <- glm(was_delayed ~ . -1 , "binomial", data = train_prep2)
```

Then we can see how these models are constructed and how they perform.

``` r
glm_unbal
```

    ## 
    ## Call:  glm(formula = was_delayed ~ . - 1, family = "binomial", data = train_prep1)
    ## 
    ## Coefficients:
    ##    month1    month10    month11    month12     month2     month3  
    ##   1.84970    2.21598    2.18136    1.36252    1.82903    1.94208  
    ##    month4     month5     month6     month7     month8     month9  
    ##   1.66679    2.03914    1.62157    1.55809    1.87549    2.62006  
    ## carrierAA  carrierB6  carrierDL  carrierEV  carrierMQ  carrierUA  
    ##   0.28168   -0.17574    0.32667   -0.39089   -0.27171    0.15833  
    ## carrierUS  carrierWN  originJFK  originLGA   distance     hour11  
    ##   0.08256   -0.17747    0.12322    0.01841   -0.10739   -0.00600  
    ##    hour12     hour13     hour14     hour15     hour16     hour17  
    ##  -0.14352   -0.22332   -0.39486   -0.59240   -0.56685   -0.70574  
    ##    hour18     hour19     hour20     hour21     hour22     hour23  
    ##  -0.73363   -0.75659   -0.68971   -0.73143   -0.59814   -0.42021  
    ##     hour5      hour6      hour7      hour8      hour9      week1  
    ##   0.44658    0.41576    0.37990    0.10514    0.05843   -0.43163  
    ##     week2      week3  
    ##  -0.16862   -0.22387  
    ## 
    ## Degrees of Freedom: 221765 Total (i.e. Null);  221721 Residual
    ##   (7379 observations deleted due to missingness)
    ## Null Deviance:       307400 
    ## Residual Deviance: 267600    AIC: 267700

Fit measures on our *training* data

``` r
library(broom)
glance(glm_unbal)
```

    ##   null.deviance df.null    logLik      AIC      BIC deviance df.residual
    ## 1      307431.6  221765 -133784.8 267657.7 268111.3 267569.7      221721

Get the coefficients

``` r
tidy(glm_unbal)
```

    ##         term     estimate   std.error   statistic       p.value
    ## 1     month1  1.849699189 0.059380343  31.1500252 5.068044e-213
    ## 2    month10  2.215979570 0.059965637  36.9541574 6.245544e-299
    ## 3    month11  2.181359326 0.060067287  36.3152629 9.291910e-289
    ## 4    month12  1.362518977 0.059528267  22.8886049 6.034077e-116
    ## 5     month2  1.829034157 0.059618121  30.6791646 1.079530e-206
    ## 6     month3  1.942083476 0.059594043  32.5885507 5.958489e-233
    ## 7     month4  1.666792196 0.059460029  28.0321459 6.594316e-173
    ## 8     month5  2.039143296 0.059687502  34.1636561 8.383578e-256
    ## 9     month6  1.621569766 0.059559061  27.2262478 3.176462e-163
    ## 10    month7  1.558089237 0.059506740  26.1834078 4.106439e-151
    ## 11    month8  1.875486283 0.059634318  31.4497815 4.226593e-217
    ## 12    month9  2.620062163 0.060763764  43.1188259  0.000000e+00
    ## 13 carrierAA  0.281680707 0.027162837  10.3700769  3.392518e-25
    ## 14 carrierB6 -0.175736079 0.023872939  -7.3613090  1.821154e-13
    ## 15 carrierDL  0.326665970 0.025272978  12.9255037  3.231731e-38
    ## 16 carrierEV -0.390886925 0.025915724 -15.0830021  2.095213e-51
    ## 17 carrierMQ -0.271712679 0.026707134 -10.1737863  2.596162e-24
    ## 18 carrierUA  0.158328682 0.027052122   5.8527268  4.835780e-09
    ## 19 carrierUS  0.082560502 0.029177457   2.8295989  4.660639e-03
    ## 20 carrierWN -0.177474196 0.033482186  -5.3005557  1.154507e-07
    ## 21 originJFK  0.123218702 0.015804737   7.7963147  6.374132e-15
    ## 22 originLGA  0.018413466 0.014299120   1.2877342  1.978385e-01
    ## 23  distance -0.107392318 0.007550445 -14.2233104  6.567135e-46
    ## 24    hour11 -0.005999922 0.031206673  -0.1922641  8.475354e-01
    ## 25    hour12 -0.143516107 0.029769114  -4.8209735  1.428594e-06
    ## 26    hour13 -0.223315006 0.029233663  -7.6389676  2.189705e-14
    ## 27    hour14 -0.394862809 0.028339123 -13.9334871  3.965136e-44
    ## 28    hour15 -0.592400092 0.027462471 -21.5712597 3.344611e-103
    ## 29    hour16 -0.566853689 0.028010645 -20.2370808  4.617200e-91
    ## 30    hour17 -0.705738121 0.027584043 -25.5850138 2.239980e-144
    ## 31    hour18 -0.733632035 0.028222823 -25.9942826 5.746969e-149
    ## 32    hour19 -0.756591701 0.028139604 -26.8870774 3.110391e-159
    ## 33    hour20 -0.689707095 0.029721943 -23.2053163 4.023903e-119
    ## 34    hour21 -0.731428319 0.032631044 -22.4151062 2.803717e-111
    ## 35    hour22 -0.598139749 0.055553426 -10.7669282  4.931818e-27
    ## 36    hour23 -0.420206184 0.080583310  -5.2145560  1.842580e-07
    ## 37     hour5  0.446578769 0.072934840   6.1229828  9.183964e-10
    ## 38     hour6  0.415759961 0.028968217  14.3522800  1.030915e-46
    ## 39     hour7  0.379900725 0.030321063  12.5292682  5.163692e-36
    ## 40     hour8  0.105143009 0.028020628   3.7523431  1.751895e-04
    ## 41     hour9  0.058434497 0.029918207   1.9531417  5.080281e-02
    ## 42     week1 -0.431626541 0.014445697 -29.8792473 3.661582e-196
    ## 43     week2 -0.168616175 0.014534422 -11.6011611  4.065249e-31
    ## 44     week3 -0.223874195 0.013449582 -16.6454394  3.265605e-62

Get the fitted data

``` r
head(augment(glm_unbal))
```

    ##   .rownames was_delayed month carrier origin distance hour week  .fitted
    ## 1         1     Delayed     1      UA    EWR 7.244228    5    0 1.676632
    ## 2         2     Delayed     1      UA    LGA 7.255591    5    0 1.693825
    ## 3         3 Not Delayed     1      B6    JFK 7.362645    5    0 1.453069
    ## 4         4     Delayed     1      UA    EWR 6.577861    5    0 1.748195
    ## 5         5     Delayed     1      B6    EWR 6.970730    6    0 1.341120
    ## 6         6 Not Delayed     1      EV    LGA 5.433722    6    0 1.309446
    ##      .se.fit     .resid         .hat   .sigma      .cooksd .std.resid
    ## 1 0.07225035 -1.9225315 0.0006928264 1.098533 8.431909e-05 -1.9231978
    ## 2 0.07336855 -1.9300609 0.0007060464 1.098533 8.742046e-05 -1.9307427
    ## 3 0.07272412  0.6482909 0.0008124028 1.098539 4.324788e-06  0.6485544
    ## 4 0.07245784 -1.9538098 0.0006630377 1.098533 8.667484e-05 -1.9544579
    ## 5 0.03098241 -1.7739579 0.0001577533 1.098534 1.371210e-05 -1.7740978
    ## 6 0.03085874  0.6913653 0.0001593987 1.098539 9.783304e-07  0.6914204

Plot predicted's vs actuals

``` r
glm_unbal %>% 
  augment() %>% 
  ggplot(aes(x=.fitted, group=was_delayed, fill=was_delayed)) +
  geom_density(alpha=.5) +
  geom_vline(aes(xintercept=0))
```

![](README_files/figure-markdown_github/unnamed-chunk-19-1.png)

#### Prep and predict on test data

``` r
test_raw %>% 
  bake(numscleaned_fe, .) %>% 
  modelr::add_predictions(glm_unbal,var = "glm_unbal")  ->
  test_scored
```

``` r
test_scored %>% 
  ggplot(aes(x=glm_unbal, group=was_delayed, fill=was_delayed)) +
  geom_density(alpha=.5) +
  geom_vline(aes(xintercept=0))
```

    ## Warning: Removed 3217 rows containing non-finite values (stat_density).

![](README_files/figure-markdown_github/unnamed-chunk-21-1.png)

But how many did we get right etc?

``` r
library(yardstick)
```

    ## 
    ## Attaching package: 'yardstick'

    ## The following object is masked from 'package:readr':
    ## 
    ##     spec

``` r
test_scored %>% 
  mutate(glm_unbal_class=as.factor(
      ifelse(glm_unbal<0, "Delayed", "Not Delayed"))) %>% 
  conf_mat(was_delayed, glm_unbal_class)
```

    ##              Truth
    ## Prediction    Delayed Not Delayed
    ##   Delayed        5629        4332
    ##   Not Delayed   26060       58964

``` r
test_scored %>% 
  mutate(glm_unbal_class=as.factor(
      ifelse(glm_unbal<0, "Delayed", "Not Delayed"))) %>% 
  accuracy(was_delayed, glm_unbal_class)
```

    ## [1] 0.6800337
