# Working with Data in the Tidyverse

<https://learn.datacamp.com/courses/working-with-data-in-the-tidyverse>

## Explore your data

Load the `readr` package for every session so things work properly:


```r
library(dplyr)
```

```
## 
## Attaching package: 'dplyr'
```

```
## The following objects are masked from 'package:stats':
## 
##     filter, lag
```

```
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

```r
library(readr)
election <- read_csv("http://faculty.baruch.cuny.edu/geoportal/data/county_election/elpo12p010g.csv")
```

```
## Rows: 3153 Columns: 14
```

```
## ── Column specification ────────────────────────────────────────────────────────
## Delimiter: ","
## chr (5): FIPS, STATE_FIPS, STATE, COUNTY, WINNER
## dbl (9): OBAMA, ROMNEY, OTHERS, TTL_VT, PCT_OBM, PCT_ROM, PCT_OTHR, PCT_WNR,...
## 
## ℹ Use `spec()` to retrieve the full column specification for this data.
## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.
```

```r
election
```

```
## # A tibble: 3,153 × 14
##    FIPS  STATE_FIPS STATE COUNTY   OBAMA ROMNEY OTHERS TTL_VT PCT_OBM PCT_ROM
##    <chr> <chr>      <chr> <chr>    <dbl>  <dbl>  <dbl>  <dbl>   <dbl>   <dbl>
##  1 01001 01         AL    Autauga   6363  17379    231  23973    26.5    72.5
##  2 01003 01         AL    Baldwin  18329  65772    887  84988    21.6    77.4
##  3 01005 01         AL    Barbour   5912   5550     55  11517    51.3    48.2
##  4 01007 01         AL    Bibb      2202   6132     86   8420    26.2    72.8
##  5 01009 01         AL    Blount    2970  20757    333  24060    12.3    86.3
##  6 01011 01         AL    Bullock   4061   1251     10   5322    76.3    23.5
##  7 01013 01         AL    Butler    4374   5087     41   9502    46.0    53.5
##  8 01015 01         AL    Calhoun  15500  30272    468  46240    33.5    65.5
##  9 01017 01         AL    Chambers  6871   7626    132  14629    47.0    52.1
## 10 01019 01         AL    Cherokee  2132   7506    154   9792    21.8    76.7
## # … with 3,143 more rows, and 4 more variables: PCT_OTHR <dbl>, WINNER <chr>,
## #   PCT_WNR <dbl>, group <dbl>
```

```r
#To skip lines from the top, start from the 2nd line for example, use "skip = 1" after comma in read_csv(...,)
```

`readr` is for rectangular data with extensions like: `.csv`, .tsv, `.fwf`, and `.log`

`read_csv()` function to read a csv file.

`readxl()` to read Microsoft Excel files.

**Assign Missing Values**

The `read_csv()` function also has an `na` argument, which allows you to specify value(s) that represent missing values in your data. 
The default values for the na argument are `c("", "NA")`, so both are recorded as missing (`NA`) in R. When you read in data, you can add additional values like the string `"UNKNOWN"` to a vector of missing values using the `c()` function to combine multiple values into a single vector.

The `is.na()` function is also helpful for identifying rows with missing values for a variable.

    read_csv("electiomn.csv", skip = 1, na = c("", "NA", "A new unspecified variable"))

    election %>% filter(is.na(variable you want to filter to find na))

The election data didn't have any observations classified as a `na` so the example above is for when assuming there's a missing value you want to filter/find. 

**How to list out all the variables of a dataset**

Use `glimpse()` to list out the hidden variables in the dataset:


```r
glimpse(election)
```

```
## Rows: 3,153
## Columns: 14
## $ FIPS       <chr> "01001", "01003", "01005", "01007", "01009", "01011", "0101…
## $ STATE_FIPS <chr> "01", "01", "01", "01", "01", "01", "01", "01", "01", "01",…
## $ STATE      <chr> "AL", "AL", "AL", "AL", "AL", "AL", "AL", "AL", "AL", "AL",…
## $ COUNTY     <chr> "Autauga", "Baldwin", "Barbour", "Bibb", "Blount", "Bullock…
## $ OBAMA      <dbl> 6363, 18329, 5912, 2202, 2970, 4061, 4374, 15500, 6871, 213…
## $ ROMNEY     <dbl> 17379, 65772, 5550, 6132, 20757, 1251, 5087, 30272, 7626, 7…
## $ OTHERS     <dbl> 231, 887, 55, 86, 333, 10, 41, 468, 132, 154, 156, 38, 56, …
## $ TTL_VT     <dbl> 23973, 84988, 11517, 8420, 24060, 5322, 9502, 46240, 14629,…
## $ PCT_OBM    <dbl> 26.54236, 21.56657, 51.33281, 26.15202, 12.34414, 76.30590,…
## $ PCT_ROM    <dbl> 72.49406, 77.38975, 48.18963, 72.82660, 86.27182, 23.50620,…
## $ PCT_OTHR   <dbl> 0.963584, 1.043677, 0.477555, 1.021378, 1.384040, 0.187899,…
## $ WINNER     <chr> "Romney", "Romney", "Obama", "Romney", "Romney", "Obama", "…
## $ PCT_WNR    <dbl> 72.49406, 77.38975, 51.33281, 72.82660, 86.27182, 76.30590,…
## $ group      <dbl> 24, 24, 12, 24, 25, 14, 22, 23, 22, 24, 24, 22, 22, 24, 25,…
```

Combine `glimpse()` with other functions in a sequence using the pipe (`%>%`) operator. For example, function like `arrange`:


```r
election %>% 
  arrange(OBAMA) %>% 
  glimpse() # no argument needed here
```

```
## Rows: 3,153
## Columns: 14
## $ FIPS       <chr> "48269", "48301", "31009", "31005", "31075", "48431", "4803…
## $ STATE_FIPS <chr> "48", "48", "31", "31", "31", "48", "48", "48", "51", "31",…
## $ STATE      <chr> "TX", "TX", "NE", "NE", "NE", "TX", "TX", "TX", "VA", "NE",…
## $ COUNTY     <chr> "King", "Loving", "Blaine", "Arthur", "Grant", "Sterling", …
## $ OBAMA      <dbl> 5, 9, 29, 30, 30, 31, 32, 33, 37, 41, 42, 44, 49, 51, 55, 5…
## $ ROMNEY     <dbl> 139, 54, 268, 227, 322, 459, 324, 468, 6463, 237, 360, 526,…
## $ OTHERS     <dbl> 1, 1, 6, 5, 11, 4, 7, 7, 222, 13, 6, 8, 9, 12, 10, 7, 6, 6,…
## $ TTL_VT     <dbl> 145, 64, 303, 262, 363, 494, 363, 508, 6722, 291, 408, 578,…
## $ PCT_OBM    <dbl> 3.448276, 14.062500, 9.570957, 11.450382, 8.264463, 6.27530…
## $ PCT_ROM    <dbl> 95.86207, 84.37500, 88.44885, 86.64122, 88.70523, 92.91498,…
## $ PCT_OTHR   <dbl> 0.689655, 1.562500, 1.980198, 1.908397, 3.030303, 0.809717,…
## $ WINNER     <chr> "Romney", "Romney", "Romney", "Romney", "Romney", "Romney",…
## $ PCT_WNR    <dbl> 95.86207, 84.37500, 88.44885, 86.64122, 88.70523, 92.91498,…
## $ group      <dbl> 25, 25, 25, 25, 25, 25, 25, 25, 25, 25, 25, 25, 25, 25, 25,…
```
Compare to the two code chunks, this arraged data shows the ascending order of `OBAMA`.

**How to get a list of summarized statistics of all the variables of a dataset**

Use `skim()` to list the standard statistics of the variables:


```r
library(skimr)
        
skim(election)
```


Table: (\#tab:unnamed-chunk-4)Data summary

|                         |         |
|:------------------------|:--------|
|Name                     |election |
|Number of rows           |3153     |
|Number of columns        |14       |
|_______________________  |         |
|Column type frequency:   |         |
|character                |5        |
|numeric                  |9        |
|________________________ |         |
|Group variables          |None     |


**Variable type: character**

|skim_variable | n_missing| complete_rate| min| max| empty| n_unique| whitespace|
|:-------------|---------:|-------------:|---:|---:|-----:|--------:|----------:|
|FIPS          |         0|             1|   5|   5|     0|     3153|          0|
|STATE_FIPS    |         0|             1|   2|   2|     0|       51|          0|
|STATE         |         0|             1|   2|   2|     0|       51|          0|
|COUNTY        |         0|             1|   2|  23|     0|     1862|          0|
|WINNER        |         0|             1|   5|   6|     0|        2|          0|


**Variable type: numeric**

|skim_variable | n_missing| complete_rate|     mean|        sd|    p0|     p25|      p50|      p75|       p100|hist  |
|:-------------|---------:|-------------:|--------:|---------:|-----:|-------:|--------:|--------:|----------:|:-----|
|OBAMA         |         0|             1| 20897.02|  73657.67|  5.00| 1572.00|  3959.00| 11259.00| 2216903.00|▇▁▁▁▁ |
|ROMNEY        |         0|             1| 19321.72|  44465.73| 54.00| 2926.00|  6294.00| 16032.00|  885333.00|▇▁▁▁▁ |
|OTHERS        |         0|             1|   683.27|   2262.16|  0.00|   72.00|   178.00|   485.00|   78831.00|▇▁▁▁▁ |
|TTL_VT        |         0|             1| 40902.02| 116118.03| 64.00| 4862.00| 10565.00| 28049.00| 3181067.00|▇▁▁▁▁ |
|PCT_OBM       |         0|             1|    38.54|     14.79|  0.55|   27.77|    37.28|    47.56|      93.39|▂▇▇▂▁ |
|PCT_ROM       |         0|             1|    59.63|     14.78|  5.98|   50.47|    60.72|    70.25|      96.15|▁▂▇▇▂ |
|PCT_OTHR      |         0|             1|     1.83|      1.00|  0.00|    1.17|     1.69|     2.35|      14.87|▇▁▁▁▁ |
|PCT_WNR       |         0|             1|    64.29|      9.97| 47.87|   56.12|    63.06|    71.31|      96.15|▇▇▆▂▁ |
|group         |         0|             1|    20.70|      4.52| 11.00|   22.00|    23.00|    24.00|      25.00|▃▁▁▃▇ |

You can combine `skim()` with other functions in a sequence using the pipe (%>%) operator. For example, use function `summary()` to find how many variables of each type are in the dataset:


```r
election %>% 
  skim() %>%  # no argument needed here
  summary() # no argument needed here
```

Table: (\#tab:unnamed-chunk-5)Data summary

|                         |           |
|:------------------------|:----------|
|Name                     |Piped data |
|Number of rows           |3153       |
|Number of columns        |14         |
|_______________________  |           |
|Column type frequency:   |           |
|character                |5          |
|numeric                  |9          |
|________________________ |           |
|Group variables          |None       |

**Count the data**

<center>Distinct()</center>

Use `distinct()` to find out how many different type of observations are there in one variable. To find out how many distinct states are there in the `STATE` variable:



```r
election %>%
  distinct(STATE)
```

```
## # A tibble: 51 × 1
##    STATE
##    <chr>
##  1 AL   
##  2 AZ   
##  3 AR   
##  4 CA   
##  5 CO   
##  6 CT   
##  7 DE   
##  8 DC   
##  9 FL   
## 10 GA   
## # … with 41 more rows
```

<center>Count()</center>

`count()` adds a new column named `n` to store the counts. this `count` function basically does the `group_by` and `summarize` for you. count the number of `counties` in each `state`:


```r
election %>%
  count(STATE)
```

```
## # A tibble: 51 × 2
##    STATE     n
##    <chr> <int>
##  1 AK       40
##  2 AL       67
##  3 AR       75
##  4 AZ       15
##  5 CA       58
##  6 CO       64
##  7 CT        8
##  8 DC        1
##  9 DE        3
## 10 FL       67
## # … with 41 more rows
```

Adapt the code to count by a logical condition instead:


```r
election %>%
  count(WINNER == "Obama")
```

```
## # A tibble: 2 × 2
##   `WINNER == "Obama"`     n
##   <lgl>               <int>
## 1 FALSE                2449
## 2 TRUE                  704
```

## Tame your data

**Cast Column Types**
To assign or change the type of the columns in the dataset.

<center>**Cast A Column To a Date**</center>

Use `parse_date("2012-14-08", format = "%Y-%d-%m")` then use `col_date(format = "%Y-%d-%m")` within cols() as the col_types argument of read_csv().

    parse_date("2012-14-08", format = "%Y-%d-%m")
    
    read_csv(...., col_types = cols( variale_name = col_date(format = "%Y-%d-%m")))

<center>**Cast A Column To a Number**</center>

    read_csv(...., variable_name = col_number())

Sometimes, there are `na` contained in the observations which would create errors while trying to cast. 

Diagnose parsing problems using a new `readr` function called `problems()`. Using `problems()` on a result of `read_csv()` will show you the rows and columns where parsing error occurred, what the parser expected to find (for example, a number), and the actual value that caused the parsing error.

<center>**Cast A Column As A Factor**</center>

Factors are categorical variables, where the possible values are a fixed and known set.

Use `parse_factor()` to parse variables and `col_factor()` to cast columns as categorical. Both functions have a `levels` argument that is used to specify the possible values for the factors. When `levels` is set to `NULL`, the possible values will be inferred from the unique values in the dataset. Alternatively, you can pass a list of possible values.

    read_csv(...., variable_name = col_factor(levels = NULL))
    
For more details, go to the [Cast A Factor And Examine Levels](https://econ380w21.github.io/bpAlNw1Ae7YwY9H3f/working-with-data-in-the-tidyverse.html#transform-your-data) section of this chapter.

**Recode Values**

Use `recode()` function in the `dplyr` package. The `recode` function is to re-name the observations into something easier to understand.

<center>**Recode A Character Variable**</center>


```r
election_CA <- election %>% filter(STATE == "CA")

election_CA %>% 
  mutate(STATE = recode(STATE, "CA" = "California"))
```

```
## # A tibble: 58 × 14
##    FIPS  STATE_FIPS STATE   COUNTY    OBAMA ROMNEY OTHERS TTL_VT PCT_OBM PCT_ROM
##    <chr> <chr>      <chr>   <chr>     <dbl>  <dbl>  <dbl>  <dbl>   <dbl>   <dbl>
##  1 06001 06         Califo… Alameda  469684 108182  17776 595642    78.9    18.2
##  2 06003 06         Califo… Alpine      389    236     28    653    59.6    36.1
##  3 06005 06         Califo… Amador     6830  10281    538  17649    38.7    58.3
##  4 06007 06         Califo… Butte     42669  44479   3604  90752    47.0    49.0
##  5 06009 06         Califo… Calaver…   8670  12365    751  21786    39.8    56.8
##  6 06011 06         Califo… Colusa     2314   3601    119   6034    38.3    59.7
##  7 06013 06         Califo… Contra … 290824 136517  10885 438226    66.4    31.2
##  8 06015 06         Califo… Del Nor…   3791   4614    365   8770    43.2    52.6
##  9 06017 06         Califo… El Dora…  35166  50973   2635  88774    39.6    57.4
## 10 06019 06         Califo… Fresno   129129 124490   5208 258827    49.9    48.1
## # … with 48 more rows, and 4 more variables: PCT_OTHR <dbl>, WINNER <chr>,
## #   PCT_WNR <dbl>, group <dbl>
```

<center>**Recode A Numeric Variable Into Factor **</center>

Dummy variables are often used in data analysis to bin a variable into one of two categories to indicate the absence or presence of something. Dummy variables take the value `0` or `1` to stand for, for example, `V_engine` or `S_engine`. 



```r
Car_engine <- mtcars[1:10,] %>%
  mutate(Engine_Type = recode(vs, "0" = "V_engine", .default =  "S_engine")) %>%
    select(vs, Engine_Type, everything(), -"carb") 

Car_engine
```

```
##                   vs Engine_Type  mpg cyl  disp  hp drat    wt  qsec am gear
## Mazda RX4          0    V_engine 21.0   6 160.0 110 3.90 2.620 16.46  1    4
## Mazda RX4 Wag      0    V_engine 21.0   6 160.0 110 3.90 2.875 17.02  1    4
## Datsun 710         1    S_engine 22.8   4 108.0  93 3.85 2.320 18.61  1    4
## Hornet 4 Drive     1    S_engine 21.4   6 258.0 110 3.08 3.215 19.44  0    3
## Hornet Sportabout  0    V_engine 18.7   8 360.0 175 3.15 3.440 17.02  0    3
## Valiant            1    S_engine 18.1   6 225.0 105 2.76 3.460 20.22  0    3
## Duster 360         0    V_engine 14.3   8 360.0 245 3.21 3.570 15.84  0    3
## Merc 240D          1    S_engine 24.4   4 146.7  62 3.69 3.190 20.00  0    4
## Merc 230           1    S_engine 22.8   4 140.8  95 3.92 3.150 22.90  0    4
## Merc 280           1    S_engine 19.2   6 167.6 123 3.92 3.440 18.30  0    4
```

```r
#Since there're only 2 distinct levels, the "1" = "S_engine" can be coded as ".default".
```

**Select And Reorder Variables**

Selecting a subset of columns to print can help check that a `mutate()` worked as expected, and rearranging columns next to each other can help spot obvious errors in data entry.

The `select()` **helpers** are functions that allow selection of variables based on their names:


```
##        Function                                   Usage
## 1 starts_with()                    starts with a prefix
## 2   ends_with()                      ends with a prefix
## 3    contains()               contains a literal string
## 4     matches()            matches a regular expression
## 5   num_range()   a numerical range like x01, x02, x03.
## 6      one_of()          variables in character vector.
## 7  everything()                          all variables.
## 8    last_col() last variable, possibly with an offset.
```


```r
# Move vs, and Engine_type to front and show only from mpg to drat:
Car_engine[1:10,] %>% 
   select(vs, Engine_Type, mpg:drat) 
```

```
##                   vs Engine_Type  mpg cyl  disp  hp drat
## Mazda RX4          0    V_engine 21.0   6 160.0 110 3.90
## Mazda RX4 Wag      0    V_engine 21.0   6 160.0 110 3.90
## Datsun 710         1    S_engine 22.8   4 108.0  93 3.85
## Hornet 4 Drive     1    S_engine 21.4   6 258.0 110 3.08
## Hornet Sportabout  0    V_engine 18.7   8 360.0 175 3.15
## Valiant            1    S_engine 18.1   6 225.0 105 2.76
## Duster 360         0    V_engine 14.3   8 360.0 245 3.21
## Merc 240D          1    S_engine 24.4   4 146.7  62 3.69
## Merc 230           1    S_engine 22.8   4 140.8  95 3.92
## Merc 280           1    S_engine 19.2   6 167.6 123 3.92
```

**Reformat Variable Names**

To change names **WITHOUT** changing the order of the variables, write `everything()` first in the `select()` function.
The function `clean_names()` takes an argument case that can be used to convert variable names to other cases, like `"upper_camel"` or `"all_caps"`.

use `clean_names()` from the `janitor` package to convert all variable names to snake_case.

    install.packages("janitor")


```r
library(janitor)
```

```
## 
## Attaching package: 'janitor'
```

```
## The following objects are masked from 'package:stats':
## 
##     chisq.test, fisher.test
```

```r
election[1:10,] %>% clean_names("snake")
```

```
## # A tibble: 10 × 14
##    fips  state_fips state county   obama romney others ttl_vt pct_obm pct_rom
##    <chr> <chr>      <chr> <chr>    <dbl>  <dbl>  <dbl>  <dbl>   <dbl>   <dbl>
##  1 01001 01         AL    Autauga   6363  17379    231  23973    26.5    72.5
##  2 01003 01         AL    Baldwin  18329  65772    887  84988    21.6    77.4
##  3 01005 01         AL    Barbour   5912   5550     55  11517    51.3    48.2
##  4 01007 01         AL    Bibb      2202   6132     86   8420    26.2    72.8
##  5 01009 01         AL    Blount    2970  20757    333  24060    12.3    86.3
##  6 01011 01         AL    Bullock   4061   1251     10   5322    76.3    23.5
##  7 01013 01         AL    Butler    4374   5087     41   9502    46.0    53.5
##  8 01015 01         AL    Calhoun  15500  30272    468  46240    33.5    65.5
##  9 01017 01         AL    Chambers  6871   7626    132  14629    47.0    52.1
## 10 01019 01         AL    Cherokee  2132   7506    154   9792    21.8    76.7
## # … with 4 more variables: pct_othr <dbl>, winner <chr>, pct_wnr <dbl>,
## #   group <dbl>
```

```r
#Notice how all the variables are lower_case now.
```

**How To Rename, Subset, And Reorder Variables At Once**

To rename, then subset (choose to show only selected) , and finally reorder variables in one code line, use `select()`.

    dataset_name %>% select( new_variable_name_ = starts_with("old_name") ) 

This code will find all variables in `dataset_name` whose names start with `old_name`, then rename each variable as `new_name_<N>`, where `N` is a number. If `dataset_name` has variables `oldname`, `oldname_v1`, `oldname3`, then the code will replace these names with `new_name_1`, `new_name_2`, `new_name_3`.


The arguments inputted into `select()` determines what R will show. And, the order of the arguments inputted will determine how the resulting order of the variables will be.


```r
election[1:10,] %>% 
  select(Precint_ = contains("PCT"), everything(), -"group")
```

```
## # A tibble: 10 × 13
##    Precint_1 Precint_2 Precint_3 Precint_4 FIPS  STATE_FIPS STATE COUNTY   OBAMA
##        <dbl>     <dbl>     <dbl>     <dbl> <chr> <chr>      <chr> <chr>    <dbl>
##  1      26.5      72.5     0.964      72.5 01001 01         AL    Autauga   6363
##  2      21.6      77.4     1.04       77.4 01003 01         AL    Baldwin  18329
##  3      51.3      48.2     0.478      51.3 01005 01         AL    Barbour   5912
##  4      26.2      72.8     1.02       72.8 01007 01         AL    Bibb      2202
##  5      12.3      86.3     1.38       86.3 01009 01         AL    Blount    2970
##  6      76.3      23.5     0.188      76.3 01011 01         AL    Bullock   4061
##  7      46.0      53.5     0.431      53.5 01013 01         AL    Butler    4374
##  8      33.5      65.5     1.01       65.5 01015 01         AL    Calhoun  15500
##  9      47.0      52.1     0.902      52.1 01017 01         AL    Chambers  6871
## 10      21.8      76.7     1.57       76.7 01019 01         AL    Cherokee  2132
## # … with 4 more variables: ROMNEY <dbl>, OTHERS <dbl>, TTL_VT <dbl>,
## #   WINNER <chr>
```
This example, the `PCT_name` has been replaced into `Precint_<N>`. The `Precint_` is first in the `select()` function so it will be shown first, then next is `everything()` so everything else will be shown the way it is, then exclude the `group` variable.

## Tidy your data

**gather()**

    ?gather

The `gather()` function collapses multiple columns into two columns. It reshapes the dataset from wide to long, it reduces the number of columns and increases the number of rows.


```r
library(tidyr)

Car_engine[1:2,] %>%
  gather(key = "measurements", value = "specs", mpg:hp) %>%
    select(measurements, specs)
```

```
##   measurements specs
## 1          mpg    21
## 2          mpg    21
## 3          cyl     6
## 4          cyl     6
## 5         disp   160
## 6         disp   160
## 7           hp   110
## 8           hp   110
```

**seperate()**

Within a tidy dataset, a column should only represent only one variable, but if observations from that variable contains two distinct type of info, we can seperate it with `seperate()`.

This is an example for the code before/after using the `seperate` funtion:

    week_ratings
    
       series episode viewers_7day
       <fct>  <chr>          <dbl>
     1 1      e1_7day         2.24
     2 2      e1_7day         3.1 
     3 3      e1_7day         3.85
     4 4      e1_7day         6.6 
     5 5      e1_7day         8.51
     6 6      e1_7day        11.6 
     7 7      e1_7day        13.6 
     8 8      e1_7day         9.46
     9 1      e2_7day         3   
    10 2      e2_7day         3.53
    
    week_ratings <- ratings2 %>% 
        select(series, ends_with("7day")) %>% 
        gather(episode, viewers_7day, ends_with("7day"), 
               na.rm = TRUE) %>%
    	# Edit to separate key column and drop extra
        separate(episode, into = c("episode","day")) 
        
    week_ratings
    
       series episode   day viewers_7day
       <fct>  <chr>   <chr>         <dbl>
     1 1      e1      7day           2.24
     2 2      e1      7day           3.1 
     3 3      e1      7day           3.85
     4 4      e1      7day           6.6 
     5 5      e1      7day           8.51
     6 6      e1      7day          11.6 
     7 7      e1      7day          13.6 
     8 8      e1      7day           9.46
     9 1      e2      7day           3   
    10 2      e2      7day           3.53


**Unite Columns**

In the `tidyr` package, the opposite of `separate()` is` unite()`. Sometimes you need to paste values from two or more columns together to tidy. Here is an example usage for `unite()`:

    data %>%
        unite(new_var, old_var1, old_var2)
To apply the function into the `election` data, merge the `COUNTY` into the `STATE`: 


```r
election_CA[1:5,] %>%
  unite(LOCATION, STATE, COUNTY, sep = ", ")
```

```
## # A tibble: 5 × 13
##   FIPS  STATE_FIPS LOCATION  OBAMA ROMNEY OTHERS TTL_VT PCT_OBM PCT_ROM PCT_OTHR
##   <chr> <chr>      <chr>     <dbl>  <dbl>  <dbl>  <dbl>   <dbl>   <dbl>    <dbl>
## 1 06001 06         CA, Ala… 469684 108182  17776 595642    78.9    18.2     2.98
## 2 06003 06         CA, Alp…    389    236     28    653    59.6    36.1     4.29
## 3 06005 06         CA, Ama…   6830  10281    538  17649    38.7    58.3     3.05
## 4 06007 06         CA, But…  42669  44479   3604  90752    47.0    49.0     3.97
## 5 06009 06         CA, Cal…   8670  12365    751  21786    39.8    56.8     3.45
## # … with 3 more variables: WINNER <chr>, PCT_WNR <dbl>, group <dbl>
```

**spread()**

Spreading reshapes the data from long to wide, adds columns and shrinks the rows.




```r
tidy_ratings_all[1:10,]
```

```
##    series episode days viewers
## 1       1       1    7    2.24
## 2       2       1    7    3.10
## 3       3       1    7    3.85
## 4       4       1    7    6.60
## 5       5       1    7    8.51
## 6       6       1    7   11.62
## 7       7       1    7   13.58
## 8       8       1    7    9.46
## 9       6       1   28   11.73
## 10      7       1   28   13.86
```


```r
tidy_ratings_all %>% 
	# Count viewers by series and days
    count(series, days, wt = viewers) %>%
	# Adapt to spread counted values
    spread(days, n, sep = "_")
```

```
##   series  days_7 days_28
## 1      1  16.620      NA
## 2      2  31.610      NA
## 3      3  50.010      NA
## 4      4  73.540      NA
## 5      5 100.393      NA
## 6      6 123.110  113.00
## 7      7 135.630  138.45
## 8      8  90.170   92.87
```

## Transform your data

**How To Create A Range-filtered Column**

Use case_when() to create a new column that represents a given range:




```r
bakers_sample <- bakers[10:17,] %>%
  select(baker, star_baker, technical_winner)

bakers_sample
```

```
##     baker star_baker technical_winner
## 10   Ruth          0                0
## 11    Ben          0                1
## 12  Holly          2                2
## 13    Ian          0                0
## 14  Janet          1                1
## 15  Jason          2                1
## 16 Joanne          1                3
## 17  Keith          0                0
```


```r
# Create skills variable with 4 levels
bakers_skill <- bakers_sample %>% 
  mutate(skill = case_when(
    star_baker > technical_winner ~ "super_star",
    star_baker < technical_winner ~ "high_tech",
    star_baker == 0 & technical_winner == 0 ~ NA_character_,
    star_baker == technical_winner  ~ "well_rounded"
  )) %>%
      drop_na(skill)

bakers_skill
```

```
##    baker star_baker technical_winner        skill
## 1    Ben          0                1    high_tech
## 2  Holly          2                2 well_rounded
## 3  Janet          1                1 well_rounded
## 4  Jason          2                1   super_star
## 5 Joanne          1                3    high_tech
```

**Cast A Factor And Examine Levels**

Cast `skill` as a factor:


```r
bakers_fct_skill <- bakers_skill %>% 
  mutate(skill = as.factor(skill))

# Examine levels
bakers_fct_skill %>% 
  pull(skill) %>% 
    levels()
```

```
## [1] "high_tech"    "super_star"   "well_rounded"
```

For more details, go to the [Cast A Column As A Factor](https://econ380w21.github.io/bpAlNw1Ae7YwY9H3f/working-with-data-in-the-tidyverse.html#tame-your-data) section of this chapter.

**Cast Characters As Dates**

Use `lubridate` to parse and cast a date variable within a `mutate()`.

    ?lubridate

```r
library(lubridate)
```

```
## 
## Attaching package: 'lubridate'
```

```
## The following objects are masked from 'package:base':
## 
##     date, intersect, setdiff, union
```

```r
bakers %>% 
  mutate(last_date_appeared = as.character(last_date_appeared))  %>%
  mutate(last_date = ymd(last_date_appeared),
         last_month = month(last_date_appeared, label = TRUE)) %>%
    select(baker, last_date_appeared, last_date, last_month)
```

```
##         baker last_date_appeared  last_date last_month
## 1     Annetha         2010-08-24 2010-08-24        Aug
## 2       David         2010-09-07 2010-09-07        Sep
## 3         Edd         2010-09-21 2010-09-21        Sep
## 4   Jasminder         2010-09-14 2010-09-14        Sep
## 5    Jonathan         2010-08-31 2010-08-31        Aug
## 6         Lea         2010-08-17 2010-08-17        Aug
## 7      Louise         2010-08-24 2010-08-24        Aug
## 8        Mark         2010-08-17 2010-08-17        Aug
## 9     Miranda         2010-09-21 2010-09-21        Sep
## 10       Ruth         2010-09-21 2010-09-21        Sep
## 11        Ben         2011-09-06 2011-09-06        Sep
## 12      Holly         2011-10-04 2011-10-04        Oct
## 13        Ian         2011-08-30 2011-08-30        Aug
## 14      Janet         2011-09-27 2011-09-27        Sep
## 15      Jason         2011-09-13 2011-09-13        Sep
## 16     Joanne         2011-10-04 2011-10-04        Oct
## 17      Keith         2011-08-16 2011-08-16        Aug
## 18  Mary-Anne         2011-10-04 2011-10-04        Oct
## 19     Robert         2011-09-13 2011-09-13        Sep
## 20      Simon         2011-08-23 2011-08-23        Aug
## 21    Urvashi         2011-08-30 2011-08-30        Aug
## 22     Yasmin         2011-09-20 2011-09-20        Sep
## 23    Brendan         2012-10-16 2012-10-16        Oct
## 24    Cathryn         2012-10-02 2012-10-02        Oct
## 25      Danny         2012-10-09 2012-10-09        Oct
## 26      James         2012-10-16 2012-10-16        Oct
## 27       John         2012-10-16 2012-10-16        Oct
## 28    Manisha         2012-09-11 2012-09-11        Sep
## 29    Natasha         2012-08-14 2012-08-14        Aug
## 30      Peter         2012-08-21 2012-08-21        Aug
## 31       Ryan         2012-09-25 2012-09-25        Sep
## 32 Sarah-Jane         2012-09-25 2012-09-25        Sep
## 33   Victoria         2012-08-28 2012-08-28        Aug
## 34     Stuart         2012-09-04 2012-09-04        Sep
## 35        Ali         2013-09-10 2013-09-10        Sep
## 36       Beca         2013-10-15 2013-10-15        Oct
## 37  Christine         2013-10-08 2013-10-08        Oct
## 38    Deborah         2013-09-03 2013-09-03        Sep
## 39    Frances         2013-10-22 2013-10-22        Oct
## 40      Glenn         2013-10-01 2013-10-01        Oct
## 41     Howard         2013-09-24 2013-09-24        Sep
## 42  Kimberley         2013-10-22 2013-10-22        Oct
## 43       Lucy         2013-08-27 2013-08-27        Aug
## 44       Mark         2013-09-03 2013-09-03        Sep
## 45     Robert         2013-09-17 2013-09-17        Sep
## 46       Ruby         2013-10-22 2013-10-22        Oct
## 47       Toby         2013-08-20 2013-08-20        Aug
## 48     Chetna         2014-10-01 2014-10-01        Oct
## 49     Claire         2014-08-06 2014-08-06        Aug
## 50      Diana         2014-09-03 2014-09-03        Sep
## 51    Enwezor         2014-08-13 2014-08-13        Aug
## 52       Iain         2014-08-27 2014-08-27        Aug
## 53     Jordan         2014-08-20 2014-08-20        Aug
## 54       Kate         2014-09-17 2014-09-17        Sep
## 55       Luis         2014-10-08 2014-10-08        Oct
## 56     Martha         2014-09-24 2014-09-24        Sep
## 57      Nancy         2014-10-08 2014-10-08        Oct
## 58     Norman         2014-09-03 2014-09-03        Sep
## 59    Richard         2014-10-08 2014-10-08        Oct
## 60      Alvin         2015-09-09 2015-09-09        Sep
## 61     Dorret         2015-08-19 2015-08-19        Aug
## 62      Flora         2015-09-30 2015-09-30        Sep
## 63        Ian         2015-10-07 2015-10-07        Oct
## 64      Marie         2015-08-12 2015-08-12        Aug
## 65        Mat         2015-09-16 2015-09-16        Sep
## 66     Nadiya         2015-10-07 2015-10-07        Oct
## 67       Paul         2015-09-23 2015-09-23        Sep
## 68      Sandy         2015-08-26 2015-08-26        Aug
## 69        Stu         2015-08-05 2015-08-05        Aug
## 70      Tamal         2015-10-07 2015-10-07        Oct
## 71       Ugnė         2015-09-02 2015-09-02        Sep
## 72     Andrew         2016-10-26 2016-10-26        Oct
## 73  Benjamina         2016-10-12 2016-10-12        Oct
## 74    Candice         2016-10-26 2016-10-26        Oct
## 75       Jane         2016-10-26 2016-10-26        Oct
## 76       Kate         2016-09-14 2016-09-14        Sep
## 77        Lee         2016-08-24 2016-08-24        Aug
## 78     Louise         2016-08-31 2016-08-31        Aug
## 79    Michael         2016-09-07 2016-09-07        Sep
## 80        Rav         2016-09-28 2016-09-28        Sep
## 81     Selasi         2016-10-19 2016-10-19        Oct
## 82        Tom         2016-10-05 2016-10-05        Oct
## 83        Val         2016-09-21 2016-09-21        Sep
## 84      Chris         2017-09-05 2017-09-05        Sep
## 85        Flo         2017-09-12 2017-09-12        Sep
## 86      James         2017-09-26 2017-09-26        Sep
## 87      Julia         2017-10-03 2017-10-03        Oct
## 88       Kate         2017-10-31 2017-10-31        Oct
## 89       Liam         2017-10-17 2017-10-17        Oct
## 90      Peter         2017-08-29 2017-08-29        Aug
## 91     Sophie         2017-10-31 2017-10-31        Oct
## 92     Stacey         2017-10-24 2017-10-24        Oct
## 93     Steven         2017-10-31 2017-10-31        Oct
## 94        Tom         2017-09-19 2017-09-19        Sep
## 95        Yan         2017-10-10 2017-10-10        Oct
```

In the example above, I turned `last_date_appeared` into a `character` type variable first since it was a `date` already. But the point of this is proving the function of `lubridate` in converting `character` into `date` variable.

**Calulate Timespans**

The first step to calculating a timespan in `lubridate` is to make an `interval`, `duration`, `period`, then use division to convert the units to what wanted (like `weeks(x)` or `months(x)`). The `x` refers to the number of time units to be included in the period:


```r
bakers_time <- bakers %>% 
    select(baker, first_date_appeared, last_date_appeared)

bakers_time[1:5,]  %>% 
  mutate(time_on_air = interval(first_date_appeared, last_date_appeared),
         weeks_on_air = time_on_air / weeks(1))
```

```
##       baker first_date_appeared last_date_appeared
## 1   Annetha          2010-08-17         2010-08-24
## 2     David          2010-08-17         2010-09-07
## 3       Edd          2010-08-17         2010-09-21
## 4 Jasminder          2010-08-17         2010-09-14
## 5  Jonathan          2010-08-17         2010-08-31
##                      time_on_air weeks_on_air
## 1 2010-08-17 UTC--2010-08-24 UTC            1
## 2 2010-08-17 UTC--2010-09-07 UTC            3
## 3 2010-08-17 UTC--2010-09-21 UTC            5
## 4 2010-08-17 UTC--2010-09-14 UTC            4
## 5 2010-08-17 UTC--2010-08-31 UTC            2
```

**String Wrangling**

Use `stringr` package to transform, detect, replace, and remove observations within a data frame:


```r
library(stringr)

election[1:7,] %>% 
  mutate(WINNER = str_to_upper(WINNER),
    WINNER = str_replace(WINNER, "MA", "mshazam"),
      WINNER = str_replace(WINNER, "RO", "Bolosho"),
      GROUP_24 = str_detect(group, "24")) %>%
        select(COUNTY, WINNER, GROUP_24)
```

```
## # A tibble: 7 × 3
##   COUNTY  WINNER      GROUP_24
##   <chr>   <chr>       <lgl>   
## 1 Autauga BoloshoMNEY TRUE    
## 2 Baldwin BoloshoMNEY TRUE    
## 3 Barbour OBAmshazam  FALSE   
## 4 Bibb    BoloshoMNEY TRUE    
## 5 Blount  BoloshoMNEY FALSE   
## 6 Bullock OBAmshazam  FALSE   
## 7 Butler  BoloshoMNEY FALSE
```
