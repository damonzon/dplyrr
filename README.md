# dplyrr - Utilities for comfortable use of dplyr with databases
Koji MAKIYAMA  



## 1. Overview

`dplyr` is the most powerful package for data handling in R, and it has also the ability of working with databases([See Vignette](http://cran.rstudio.com/web/packages/dplyr/vignettes/databases.html)).  
But the functionalities of dealing with databases in `dplyr` is developing yet.

Now, I'm trying to make `dplyr` with databases more comfortable by using some functions.  
For that purpose, I've created `dplyrr` package.

`dplyrr` has below functions:

- `load_tbls()` : Easy to load table objects for all tables in a database.
- `cut()` in `mutate()` : Easy to create a case statement by using the grammar like the `base::cut()`.
- `count_if` or `n_if()` in `summarise()` : Shortcut to count rows that a condition is satisfied.
- `filter()` : Improved `filter()` for `tbl_sql` which adds parentheses appropriately.
- `moving_mean()` in `mutate()` : Compute moving average for PostgreSQL.
- `moving_max()` in `mutate()` : Compute moving max for PostgreSQL.
- `moving_min()` in `mutate()` : Compute moving min for PostgreSQL.
- `moving_sum()` in `mutate()` : Compute moving sum for PostgreSQL.
- `first_value()` in `mutate()` : Compute first value for PostgreSQL.

## 2. How to install

The source code for `dplyrr` package is available on GitHub at

- https://github.com/hoxo-m/dplyrr.  

You can install the pakage from there.


```r
install.packages("devtools") # if you have not installed "devtools" package
devtools::install_github("hoxo-m/dplyrr")
```

## 3. Common functions for all databases

For illustration, we use a database file: "my_db.sqlite3".  
If you want to trace the codes below, you should first create the databese file.


```r
library(dplyrr)
library(nycflights13)

db <- src_sqlite("my_db.sqlite3", create = TRUE)
copy_nycflights13(db)
```

### 3-1. `load_tbls()`

Usually, when we use a database with `dplyr`, we first create database object, and we can see the tables in the databese by `show()`.


```r
library(dplyrr)
# Create database object
db <- src_sqlite("my_db.sqlite3")
show(db)
```

```
## src:  sqlite 3.8.6 [my_db.sqlite3]
## tbls: airlines, airports, flights, planes, sqlite_stat1, weather
```

Next, we create table objects for pulling data in some tables in the database.


```r
airlines_tbl <- tbl(db, "airlines")
airports_tbl <- tbl(db, "airports")
flights_tbl <- tbl(db, "flights")
planes_tbl <- tbl(db, "planes")
weather_tbl <- tbl(db, "weather")
```

Typing this code is really a bore!

If you want to create table objects for **all tables in the database**, you can use `load_tbls()`.


```r
load_tbls(db)
```

```
## Loading: airlines_tbl
## Loading: airports_tbl
## Loading: flights_tbl
## Loading: planes_tbl
## Loading: sqlite_stat1_tbl
## Loading: weather_tbl
```

Check the created table objects.


```r
ls(pattern = "_tbl$")
```

```
## [1] "airlines_tbl"     "airports_tbl"     "flights_tbl"     
## [4] "planes_tbl"       "sqlite_stat1_tbl" "weather_tbl"
```

```r
glimpse(airlines_tbl)
```

```
## Observations: 16
## Variables:
## $ carrier (chr) "9E", "AA", "AS", "B6", "DL", "EV", "F9", "FL", "HA", ...
## $ name    (chr) "Endeavor Air Inc.", "American Airlines Inc.", "Alaska...
```

### 3-2. `cut()` in `mutate()`

If you want to write case statement with like `base::cut()`, you can use `cut()` function in `mutate()`.

For example, there is `air_time` column in the database.


```r
db <- src_sqlite("my_db.sqlite3")
flights_tbl <- tbl(db, "flights")
q <- flights_tbl %>% select(air_time)
air_time <- q %>% collect
head(air_time, 3)
```

```
## Source: local data frame [3 x 1]
## 
##   air_time
## 1      227
## 2      227
## 3      160
```

If you want to group the `air_time` by break points `c(0, 80, 120, 190, 900)`, you think you must write the next code.


```r
q <- flights_tbl %>% 
  select(air_time) %>%
  mutate(air_time_cut = if(air_time > 0 && air_time <= 80) "(0,80]"
         else if(air_time > 80 && air_time <= 120) "(80,120]"
         else if(air_time > 120 && air_time <= 190) "(120,190]"
         else if(air_time > 190 && air_time <= 900) "(190,900]")
air_time_with_cut <- q %>% collect
head(air_time_with_cut, 3)
```

```
## Source: local data frame [3 x 2]
## 
##   air_time air_time_cut
## 1      227    (190,900]
## 2      227    (190,900]
## 3      160    (120,190]
```

When the break points increase, you are going to be tired to write more lines.

By using `cut()` function in `mutate()`, it can become easy.


```r
q <- flights_tbl %>% 
  select(air_time) %>%
  mutate(air_time_cut = cut(air_time, breaks=c(0, 80, 120, 190, 900)))
air_time_with_cut <- q %>% collect
head(air_time_with_cut, 3)
```

```
## Source: local data frame [3 x 2]
## 
##   air_time air_time_cut
## 1      227    (190,900]
## 2      227    (190,900]
## 3      160    (120,190]
```

The `cut()` in `mutate()` has more arguments such as `labels` coming from `base::cut()`.

- `cut(variable, breaks, labels, include.lowest, right, dig.lab)`

For integer break points, specially you can indicate `labels="-"`.


```r
q <- flights_tbl %>% 
  select(air_time) %>%
  mutate(air_time_cut = cut(air_time, breaks=c(0, 80, 120, 190, 900), labels="-"))
air_time_with_cut <- q %>% collect
head(air_time_with_cut, 3)
```

```
## Source: local data frame [3 x 2]
## 
##   air_time air_time_cut
## 1      227      191-900
## 2      227      191-900
## 3      160      121-190
```

### 3-3. `count_if()` or `n_if()` in `summarise()`

When we want to count rows that a condition is satisfied, we write like this.


```r
q <- flights_tbl %>% 
  select(air_time) %>%
  summarise(odd_airtime_rows = sum(if(air_time %% 2 == 1) 1L else 0L), 
            even_airtime_rows = sum(if(air_time %% 2 == 0) 1L else 0L), 
            total_rows=n())
q %>% collect
```

```
## Source: local data frame [1 x 3]
## 
##   odd_airtime_rows even_airtime_rows total_rows
## 1           164150            163196     336776
```

The `count_if()` and `n_if()` functions are a shortcut for it merely.

- `count_if(condition)`
- `n_if(condition)`


```r
q <- flights_tbl %>% 
  select(air_time) %>%
  summarise(odd_airtime_rows = count_if(air_time %% 2 == 1), 
            even_airtime_rows = n_if(air_time %% 2 == 0), 
            total_rows=n())
q %>% collect
```

```
## Source: local data frame [1 x 3]
## 
##   odd_airtime_rows even_airtime_rows total_rows
## 1           164150            163196     336776
```

Both functions do exactly the same thing.

### 3-4. Improved `filter()`



If you use `dplyr` with databases in pure mind, you can encounter the unintended action like below.


```r
library(dplyr)

db <- src_sqlite("my_db.sqlite3")
flights_tbl <- tbl(db, "flights")
q <- flights_tbl %>%
  select(month, air_time) %>%
  filter(month == 1) %>%
  filter(air_time > 200 || air_time < 100)
q$query
```

```
## <Query> SELECT "month" AS "month", "air_time" AS "air_time"
## FROM "flights"
## WHERE "month" = 1.0 AND "air_time" > 200.0 OR "air_time" < 100.0
## <SQLiteConnection>
```

Did you expect the where clause to be that?

If you use `dplyrr`, it becomes natural by adding parentheses.


```r
library(dplyrr)

db <- src_sqlite("my_db.sqlite3")
flights_tbl <- tbl(db, "flights")
q <- flights_tbl %>%
  select(month, air_time) %>%
  filter(month == 1) %>%
  filter(air_time > 200 || air_time < 100)
q$query
```

```
## <Query> SELECT "month" AS "month", "air_time" AS "air_time"
## FROM "flights"
## WHERE ("month" = 1.0) AND ("air_time" > 200.0 OR "air_time" < 100.0)
## <SQLiteConnection>
```

## 4. Functions for PostgreSQL



### 4-1. `moving_**()` in `mutate()`

`dplyrr` has four `moving_**()` functions that can use in `mutate()`.

- `moving_mean(variable, preceding, following)`
- `moving_max(variable, preceding, following)`
- `moving_min(variable, preceding, following)`
- `moving_sum(variable, preceding, following)`

When you want to set the same `preceding` and `following`, you can omit `following`.

For illustration, we use the test database that is PostgreSQL.


```r
srcs <- temp_srcs("postgres")
df <- data.frame(x = 1:5)
tbls <- dplyr:::temp_load(srcs, list(df = df))
temp_tbl <- tbls$postgres$df
head(temp_tbl)
```

```
##   x
## 1 1
## 2 2
## 3 3
## 4 4
## 5 5
```

Compute moving average with 1 preceding and 1 following.


```r
q <- temp_tbl %>%
  mutate(y = moving_mean(x, 1))
q %>% collect
```

```
## Source: local data frame [5 x 2]
## 
##   x   y
## 1 1 1.5
## 2 2 2.0
## 3 3 3.0
## 4 4 4.0
## 5 5 4.5
```

Comfirm query.


```r
q$query
```

```
## <Query> SELECT "x", "y"
## FROM (SELECT "x", avg("x") OVER (ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING) AS "y"
## FROM "tlsqbjsuou") AS "_W1"
## <PostgreSQLConnection:(10316,0)>
```

Compute moving mean with 1 preceding and 2 following.


```r
q <- temp_tbl %>%
  mutate(y = moving_mean(x, 1, 2))
q %>% collect
```

```
## Source: local data frame [5 x 2]
## 
##   x   y
## 1 1 2.0
## 2 2 2.5
## 3 3 3.5
## 4 4 4.0
## 5 5 4.5
```

Comfirm query.


```r
q$query
```

```
## <Query> SELECT "x", "y"
## FROM (SELECT "x", avg("x") OVER (ROWS BETWEEN 1 PRECEDING AND 2 FOLLOWING) AS "y"
## FROM "tlsqbjsuou") AS "_W2"
## <PostgreSQLConnection:(10316,0)>
```

Similary, you can use the other `moving_**()` functions.

### 4-2. `first_value()` in `mutate()`

`dplyrr` has `first_value()` function that can use in `mutate()`.

- `first_value(value, order_by)`

When you want to set the same `value` and `order_by`, you can omit `order_by`.

For illustration, we use the test database that is PostgreSQL.


```r
srcs <- temp_srcs("postgres")
df <- data.frame(class = c("A", "A", "B", "B", "C", "C"), x = 1:6, y = 6:1)
tbls <- dplyr:::temp_load(srcs, list(df=df))
temp_tbl <- tbls$postgres$df
head(temp_tbl)
```

```
##   class x y
## 1     A 1 6
## 2     A 2 5
## 3     B 3 4
## 4     B 4 3
## 5     C 5 2
## 6     C 6 1
```

Get the first values of x partitioned by class and ordered by x.


```r
q <- temp_tbl %>%
  group_by(class) %>%
  mutate(z = first_value(x))
q %>% collect
```

```
## Source: local data frame [6 x 4]
## Groups: class
## 
##   class x y z
## 1     A 1 6 1
## 2     A 2 5 1
## 3     B 3 4 3
## 4     B 4 3 3
## 5     C 5 2 5
## 6     C 6 1 5
```

See query.


```r
q$query
```

```
## <Query> SELECT "class", "x", "y", "z"
## FROM (SELECT "class", "x", "y", first_value("x") OVER (PARTITION BY "class" ORDER BY "x") AS "z"
## FROM "slrhxfdvrt") AS "_W3"
## <PostgreSQLConnection:(10316,0)>
```

Get the first values of x partitioned by class and ordered by y.


```r
q <- temp_tbl %>%
  group_by(class) %>%
  mutate(z = first_value(x, y))
q %>% collect
```

```
## Source: local data frame [6 x 4]
## Groups: class
## 
##   class x y z
## 1     A 2 5 2
## 2     A 1 6 2
## 3     B 4 3 4
## 4     B 3 4 4
## 5     C 6 1 6
## 6     C 5 2 6
```

See query.


```r
q$query
```

```
## <Query> SELECT "class", "x", "y", "z"
## FROM (SELECT "class", "x", "y", first_value("x") OVER (PARTITION BY "class" ORDER BY "y") AS "z"
## FROM "slrhxfdvrt") AS "_W4"
## <PostgreSQLConnection:(10316,0)>
```

Get the first values of x partitioned by class and ordered by descent of y.


```r
q <- temp_tbl %>%
  group_by(class) %>%
  mutate(z = first_value(x, desc(y)))
q %>% collect
```

```
## Source: local data frame [6 x 4]
## Groups: class
## 
##   class x y z
## 1     A 1 6 1
## 2     A 2 5 1
## 3     B 3 4 3
## 4     B 4 3 3
## 5     C 5 2 5
## 6     C 6 1 5
```

See query.


```r
q$query
```

```
## <Query> SELECT "class", "x", "y", "z"
## FROM (SELECT "class", "x", "y", first_value("x") OVER (PARTITION BY "class" ORDER BY "y" DESC) AS "z"
## FROM "slrhxfdvrt") AS "_W5"
## <PostgreSQLConnection:(10316,0)>
```

## 5. Miscellaneous

### `update_dplyrr()`

`update_dplyrr()` is a shortcut of 


```r
devtools::install_github("hoxo-m/dplyrr")
```

### `unload_dplyrr()`

`unload_dplyrr()` is a shortcut of 


```r
detach("package:dplyrr", unload = TRUE)
detach("package:dplyr", unload = TRUE)
```
