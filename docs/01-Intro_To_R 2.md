# (PART) DataCamp {-} 

# Introduction to R

<https://learn.datacamp.com/courses/free-introduction-to-r>

## Intro to basics

**Arithmetic**

Basic codes to perform calculation:
<center>

```
##               Name    Code Output
## 1      An addition     5+5     10
## 2    A subtraction     5-5      0
## 3 A multiplication     3*5     15
## 4       A division (5+5)/2      5
## 5    Exonentiation     2^5     32
## 6           Modulo   28%%6      4
```
</center>


**Variable Assignments**

A `variable` can store a ```value``` (e.g. 4) or ```an object``` (e.g. a function description) in R:


```r
my_socks <- 2
```

**Basic Data Types**

Here are some of the basic data types in R:

```r
    my_numeric <- 5
    my_character <- "abcdefg"
    my_logical <- FALSE
```
To check the data type of a variable, use the `class()` function as follow:


```r
class(my_numeric)
```

```
## [1] "numeric"
```

```r
class(my_character)
```

```
## [1] "character"
```

```r
class(my_logical)
```

```
## [1] "logical"
```

## Vectors

**Create A Vector**

Create a vector with the function `c()`:


```r
poker_vector <- c(140, -50, 20, -120, 240)
roulette_vector <- c(-24, -50, 100, -350, 10)
```
    
**Naming a Vector**

To assign names to the elements of a vector, use the `names()` function:


```r
names(poker_vector)    <- c("Monday", "Tuesday", "Wednesday", "Thursday", "Friday")
names(roulette_vector) <- c("Monday", "Tuesday", "Wednesday", "Thursday", "Friday")
```

Typing the days multiple times is too much work, so to save time, assign the days of the week vector to a variable: `days_vector`, and then assign that to the other vectors with the `class()` function like below:


```r
days_vector <- c("Monday", "Tuesday", "Wednesday", "Thursday", "Friday")
names(poker_vector) <- c(days_vector)   
names(roulette_vector) <- c(days_vector)  
```

**Calculating the winnings**

This code below calculates the total profit from poker and roulette each day:

```r
total_daily <- poker_vector + roulette_vector
total_daily
```

```
##    Monday   Tuesday Wednesday  Thursday    Friday 
##       116      -100       120      -470       250
```
      
To calculate the sum of all elements of a vector, use the function `sum()`.


```r
total_poker <- sum(poker_vector)
total_roulette <- sum(roulette_vector)
total_week <- total_poker + total_roulette
total_week
```

```
## [1] -84
```

**Vector Selection**

To select elements of a vector, use square brackets `[]`:


```r
poker_wednesday <- poker_vector[3]
poker_wednesday
```

```
## Wednesday 
##        20
```

Two ways to select multiple elements from a vector:


```r
poker_midweek <- poker_vector[c(2,3,4)]
poker_midweek <- poker_vector[2:4]
```

Another way to select is by using the assigned names of the vector elements above (Monday, Tuesday, …) instead of their numeric positions:



```r
poker_vector["Monday"]
```

```
## Monday 
##    140
```

```r
poker_vector[c("Monday","Tuesday")]
```

```
##  Monday Tuesday 
##     140     -50
```

**Selection By Comparison**

The (logical) comparison operators known to R are:

    "<" for less than
    ">" for greater than
    "<=" for less than or equal to
    ">=" for greater than or equal to
    "==" for equal to each other
    "!=" not equal to each other
    
Application of these comparison operators on vectors:


```r
c(4, 5, 6) > 5
```

```
## [1] FALSE FALSE  TRUE
```
    
To find the days that had a positive poker return:


```r
poker_vector > 0
```

```
##    Monday   Tuesday Wednesday  Thursday    Friday 
##      TRUE     FALSE      TRUE     FALSE      TRUE
```

To know not only the winning days, but also how much was made on those days:


```r
selection_vector <- poker_vector > 0
poker_winning_days <- poker_vector[selection_vector]
poker_winning_days
```

```
##    Monday Wednesday    Friday 
##       140        20       240
```

## Matrices

**A Matrix**

To construct a matrix in R, use the matrix() function:


```r
matrix(1:9, byrow = TRUE, nrow = 3)
```

```
##      [,1] [,2] [,3]
## [1,]    1    2    3
## [2,]    4    5    6
## [3,]    7    8    9
```

The `1:9` is the data that's written into the matrix. Here,` 1:9` is a shortcut for `c(1, 2, 3, 4, 5, 6, 7, 8, 9)`. The part `byrow` tells the matrix that the `1:9` is filled by the rows (left to right of each row from the top). To fill by the columns, write `byrow = FALSE`. The third part `nrow` tells the matrix to create 3 rows.

Example of a data-set used to create a matrix:


```r
#Box office Star Wars (in millions)
new_hope <- c(460.998, 314.4)
empire_strikes <- c(290.475, 247.900)
return_jedi <- c(309.306, 165.8)

#Create box_office (a vector)
box_office <- c(new_hope, empire_strikes, return_jedi)

#Construct star_wars_matrix
star_wars_matrix <- matrix(box_office, byrow = TRUE, nrow = 3)
star_wars_matrix
```

```
##         [,1]  [,2]
## [1,] 460.998 314.4
## [2,] 290.475 247.9
## [3,] 309.306 165.8
```

**Naming A Matrix**

To add names ro the rows and columns of a matrix:


```r
region <- c("US", "non-US")
titles <- c("A New Hope", "The Empire Strikes Back", "Return of the Jedi")

colnames(star_wars_matrix) <- region
rownames(star_wars_matrix) <- titles
```

**Calculating Rows' or Columns' Sum**

Function `rowSums()` calculates the sum of each row of a matrix. This would would give the total revenue of each movie from US + Non-US:


```r
rowSums(star_wars_matrix)
```

```
##              A New Hope The Empire Strikes Back      Return of the Jedi 
##                 775.398                 538.375                 475.106
```

To calculate the total revenue of each region (US and Non-US) from all movies:


```r
colSums(star_wars_matrix)
```

```
##       US   non-US 
## 1060.779  728.100
```

**Adding Row or Column To The Matrix**

Add a column(s) to a matrix with the `cbind()` function, which merge matrices and/or vectors together by column. For example:


```r
worldwide_vector <- rowSums(star_wars_matrix)
all_wars_matrix <- cbind(star_wars_matrix, worldwide_vector)
all_wars_matrix
```

```
##                              US non-US worldwide_vector
## A New Hope              460.998  314.4          775.398
## The Empire Strikes Back 290.475  247.9          538.375
## Return of the Jedi      309.306  165.8          475.106
```
    
**Selection of matrix elements**

Use the square brackets `[ ]` to select element(s) from a matrix. Vectors contain one type of data (like only x-axis) and matrices contain two types (like x and y-axis). So, use a comma to separate the rows, columns to be selected. For example:

    my_matrix[1,2] selects the element at the first row and second column.
    my_matrix[1:3,2:4] results in a matrix with the data on the rows 1, 2, 3 and columns 2, 3, 4.

To select all elements of a row or a column, no number is needed before or after the comma:

    my_matrix[,1] selects all elements of the first column.
    my_matrix[1,] selects all elements of the first row.
    
**A little arithmetic with matrices**

`2 * matrix` would multiply each element of my_matrix by two.
    
`matrix1 * matrix2` would create a new matrix where each element is the product of the corresponding elements in `matrix1` and `matrix2`. 

    -Estimated number of visitors
    visitors <- all_wars_matrix/ticket_prices_matrix

## Factors

There are two types of categorical variables: **nominal categorical** variable and **ordinal categorical** variable.

A **nominal variable** is a categorical variable *without* an implied order. For example, the categorical variable items_vector containing `"Car", "Key", "Phone" and "Shoe"`. It's hard to compare these elements using a standard.

**Ordinal variables** have *natural ordering*. For example, the categorical variable `temperature_vector` with the categories: `"Low", "Medium" and "High"`. The temps are comparable using a standard.

The function `factor()` point out the different element groups in the data-set:


```r
survey_vector <- c("M", "F", "F", "M", "M")
factor_survey_vector <- factor(survey_vector)
factor_survey_vector
```

```
## [1] M F F M M
## Levels: F M
```

**Factor Levels(Naming Elements)**

To change the factor levels "M" and "F" to "Male" and "Female":


```r
levels(factor_survey_vector) <- c("Female", "Male")
factor_survey_vector
```

```
## [1] Male   Female Female Male   Male  
## Levels: Female Male
```

**Summarizing a factor**

To know how many `"Male"` responses are in the study, and how many `"Female"` responses. The `summary()` function gives the answer to this question:


```r
summary(factor_survey_vector)
```

```
## Female   Male 
##      2      3
```

**Ordered factors (Comparing elements)**

The function `factor()` convert `speed_vector` into *unordered* factor. To create an *ordered* factor, add two additional arguments: `ordered` and `levels`:


```r
speed_vector <- c("medium", "slow", "slow", "medium", "fast")
factor_speed_vector <- factor(speed_vector, ordered = TRUE, levels = c("slow", "medium", "fast"))
factor_speed_vector
```

```
## [1] medium slow   slow   medium fast  
## Levels: slow < medium < fast
```

The `ordered = TRUE` is saying that the elements can be compared with a standard. The `levels` state the elements of the factor in ranks.

**Comparing ordered factors**

The ordered `factor_speed_vector` enables the comparison of different elements:


```r
data_analyst2 <- factor_speed_vector[2]
data_analyst5 <- factor_speed_vector[5]

data_analyst2 > data_analyst5
```

```
## [1] FALSE
```

## Data frames

Use function `data.frame()` to create a data frame. Example from Datacamp below:


```r
#Definition of vectors
name       <- c("Mercury", "Venus", "Earth", "Mars", "Jupiter", "Saturn", "Uranus", "Neptune")
type       <- c("Terrestrial planet", "Terrestrial planet", "Terrestrial planet", "Terrestrial planet", "Gas giant", "Gas giant", "Gas giant", "Gas giant")
diameter   <- c(0.382, 0.949, 1, 0.532, 11.209, 9.449, 4.007, 3.883)
rotation   <- c(58.64, -243.02, 1, 1.03, 0.41, 0.43, -0.72, 0.67)
rings      <- c(FALSE, FALSE, FALSE, FALSE, TRUE, TRUE, TRUE, TRUE)

#Create a data frame from the vectors
planets_df <- data.frame(name, type, diameter, rotation, rings)
planets_df
```

```
##      name               type diameter rotation rings
## 1 Mercury Terrestrial planet    0.382    58.64 FALSE
## 2   Venus Terrestrial planet    0.949  -243.02 FALSE
## 3   Earth Terrestrial planet    1.000     1.00 FALSE
## 4    Mars Terrestrial planet    0.532     1.03 FALSE
## 5 Jupiter          Gas giant   11.209     0.41  TRUE
## 6  Saturn          Gas giant    9.449     0.43  TRUE
## 7  Uranus          Gas giant    4.007    -0.72  TRUE
## 8 Neptune          Gas giant    3.883     0.67  TRUE
```

**Analyzation Tools**

<center>**Head and Tail**</center>

The function  `head()` and `tail()` show show the first few and last few observations of a data frame.

<center>**Structure**</center>

The function `str()` helps investigate the structure of the `planets_df` data frame:


```r
str(planets_df)
```

```
## 'data.frame':	8 obs. of  5 variables:
##  $ name    : chr  "Mercury" "Venus" "Earth" "Mars" ...
##  $ type    : chr  "Terrestrial planet" "Terrestrial planet" "Terrestrial planet" "Terrestrial planet" ...
##  $ diameter: num  0.382 0.949 1 0.532 11.209 ...
##  $ rotation: num  58.64 -243.02 1 1.03 0.41 ...
##  $ rings   : logi  FALSE FALSE FALSE FALSE TRUE TRUE ...
```

**Selection of data frame elements**

Use `[]` to select specific elements from a data frame. Examples from Datacamp below:


```r
#Print out data for Mars (entire fourth row)
planets_df[4,]
```

```
##   name               type diameter rotation rings
## 4 Mars Terrestrial planet    0.532     1.03 FALSE
```

```r
#Select first 5 values of diameter column
    planets_df[1:5,"diameter"]
```

```
## [1]  0.382  0.949  1.000  0.532 11.209
```

<center>**Shortcut `$` to select dataframe elements**</center>

The sign `$` is a shortcut to select the wanted column that has a name:
    

```r
#Select the rings variable from planets_df
rings_vector <- planets_df$rings
  
rings_vector
```

```
## [1] FALSE FALSE FALSE FALSE  TRUE  TRUE  TRUE  TRUE
```

To show the variable names of the data frame instead of only the result above:


```r
#Select all columns for planets with rings 
planets_df[rings_vector,]
```

```
##      name      type diameter rotation rings
## 5 Jupiter Gas giant   11.209     0.41  TRUE
## 6  Saturn Gas giant    9.449     0.43  TRUE
## 7  Uranus Gas giant    4.007    -0.72  TRUE
## 8 Neptune Gas giant    3.883     0.67  TRUE
```
    
<center>**Shortcut `subset()` to select dataframe elements**</center>

The `subset(planets_df, subset = rings)` function give out the exact result like above.

Another example from Datacamp for variety:


```r
#Select planets with diameter < 1
subset(planets_df, diameter < 1)
```

```
##      name               type diameter rotation rings
## 1 Mercury Terrestrial planet    0.382    58.64 FALSE
## 2   Venus Terrestrial planet    0.949  -243.02 FALSE
## 4    Mars Terrestrial planet    0.532     1.03 FALSE
```

**Sorting**

Use function `sort()` to organize elements into ranks. To find out the order of planets according to diameter:


```r
order(planets_df$diameter)
```

```
## [1] 1 4 2 3 8 7 6 5
```

The results indicate that planet #1 is the smallest, #4 is the second-smallest, and planet #5 is the largest. Now, to reorganize the data frame according to the ranking of diameter: 


```r
positions <-  order(planets_df$diameter)
```

## Lists

A list could contain matrices, vectors, other lists, … To create a list, use the `function list()`:

    my_list <- list(comp1, comp2 ...)
    
To give names to components of a list, there are two ways:

    my_list <- list(your_comp1, your_comp2)
    names(my_list) <- c("name1", "name2")
    
    or 
    
    my_list <- list(name1 = your_comp1, 
                    name2 = your_comp2)
                    
To select elements from a list:

    -Select the vector representing the actors
    shining_list[["actors"]]
    shining_list$actors
    
    Select the second element of the vector representing the actors
    shining_list[["actors"]][2]
    
  
