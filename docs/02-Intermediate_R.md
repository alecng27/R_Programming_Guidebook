# Intermediate R

<https://learn.datacamp.com/courses/intermediate-r>

## Conditionals And Control Flow

Reminder: `==` is for comparison and `=` is for assignment.

`TRUE` is treated as `1` for arithmetic, and `FALSE` is treated as `0`.

**Compare Vectors and Matrices**

<center>**Vectors**</center>

Number of views on each site:

```r
linkedin <- c(16, 9, 13, 5, 2, 17, 14)
facebook <- c(17, 7, 5, 16, 8, 13, 14)
```

To find out which had more views:


```r
linkedin >= facebook
```

```
## [1] FALSE  TRUE  TRUE FALSE FALSE  TRUE  TRUE
```

<center>**Matrices**</center>

Compare data in matrices:

```r
views <- matrix(c(linkedin, facebook), nrow = 2, byrow = TRUE)
# When is views less than or equal to 14?
views <= 14
```

```
##       [,1] [,2] [,3]  [,4] [,5]  [,6] [,7]
## [1,] FALSE TRUE TRUE  TRUE TRUE FALSE TRUE
## [2,] FALSE TRUE TRUE FALSE TRUE  TRUE TRUE
```

**Logical Operators**

<center>**& and |**</center>

With "Or" function: `|', only one condition (the right one or the left one) needs to be satisfied to spit out `TRUE`:

```r
socks <- 13
# Is last under 5 or above 10?
socks < 5 | socks >10
```

```
## [1] TRUE
```

With "And" function: "&", both conditions (the right and left ones) need to be satisfied to spit out `TRUE`. In this case, the "left condition" is not satisfied, hence:

```r
# Is last between 15 (exclusive) and 20 (inclusive)?
socks > 15 & socks <= 20
```

```
## [1] FALSE
```

<center>**"!" Operator**</center>

The `!` operator negates a logical value:


```r
x <- 5
y <- 7
!((x < 4) & !(y > 12))
```

```
## [1] TRUE
```

This can be a brain-twister. Both `x` and `y` element is `FALSE`. However, there's a `!` in front of `y`, therefore, `(y-12)` is `TRUE`. So, within the entire `()`, a `FALSE` and `TRUE` would spit out a `FALSE` because one component is `FALSE`. Then, the `!` operator would flip the result, which becomes `TRUE` in the end.

<center>**The "IF" Statement**</center>

`IF `function allows the logic of "if this happens then do A." For example:

```r
num_views <- 14

if (num_views > 15) {
  print("You're popular!")
} 
```

This is the expanded of the above logic thought: "if this happens then do A, if doesn't, then do B, and if both doesn't happen, then do the C":


```r
if (num_views > 15) {
  print("You're popular!")
} else if (num_views <= 15 & num_views > 10) {
  print("Your number of views is average")
} else {
  print("Try to be more visible!")
}
```

```
## [1] "Your number of views is average"
```

Logic are lawless inside "if-else" constructs. This is a good example from Datacamp:

    if (number < 10) {
      if (number < 5) {
        result <- "extra small"
      } else {
        result <- "small"
      }
    } else if (number < 100) {
      result <- "medium"
    } else {
      result <- "large"
    }
    print(result)

## Loops

**"While" Loop**

`While` function's logic is "while this is true, keep doing the task". Example from datacamp to understand:


```r
speed <- 64

# Extend/adapt the while loop
while (speed > 30) {
  print(paste("Your speed is",speed))
  if (speed > 48) {
    print("Slow down big time!")
    speed <- speed - 11
  } else { 
    print("Slow down!")
    speed <- speed - 6
  }
}
```

```
## [1] "Your speed is 64"
## [1] "Slow down big time!"
## [1] "Your speed is 53"
## [1] "Slow down big time!"
## [1] "Your speed is 42"
## [1] "Slow down!"
## [1] "Your speed is 36"
## [1] "Slow down!"
```

**"For" Loop**

<center>**Loop A Vector, List**</center>

This loop will print out all the `views` listed in the `linkedin` vector orderly from left to right:

```r
linkedin <- c(16, 9, 13, 5, 2, 17, 14)

# Loop version 1
for (views in linkedin) {
    print(views)
}
```

```
## [1] 16
## [1] 9
## [1] 13
## [1] 5
## [1] 2
## [1] 17
## [1] 14
```


This loop will also print out all the views from the vector, but it does so by reffering to the location of the specific element within the vector to print out:

```r
# Loop version 2
for (i in 1:length(linkedin)) {
    print(linkedin[i])
}
```

```
## [1] 16
## [1] 9
## [1] 13
## [1] 5
## [1] 2
## [1] 17
## [1] 14
```

```r
#Note: use "[[ ]]" to select the elements in loop version 2 when looping a list.
```

<center>**Loop A Matrix**</center>

A `for` loop inside a `for` loop is called a `nested` loop, example from Datacamp:

```r
ttt <- matrix(c( "O", NA, "X", NA, "O", "O", "X", NA, "X"), byrow = TRUE, nrow =3)

for (i in 1:nrow(ttt)) {
  for (j in 1:ncol(ttt)) {
    print(paste("On row", i, "and column", j, "the board contains", ttt[i,j]))
  }
}
```

```
## [1] "On row 1 and column 1 the board contains O"
## [1] "On row 1 and column 2 the board contains NA"
## [1] "On row 1 and column 3 the board contains X"
## [1] "On row 2 and column 1 the board contains NA"
## [1] "On row 2 and column 2 the board contains O"
## [1] "On row 2 and column 3 the board contains O"
## [1] "On row 3 and column 1 the board contains X"
## [1] "On row 3 and column 2 the board contains NA"
## [1] "On row 3 and column 3 the board contains X"
```

**"Break" And "Next"**

The `break` terminate the running code if the condition is `FALSE`. 

The `next` allow the code to the run after `break`. It skip over the element that made the code `FALSE` then continue.


```r
likes <- c(16, 9, 13, 5, 2, 17, 14)

for (heart in likes) {
  if (heart > 10) {
    print("You're popular!")
  } else {print("Be more visible!")
  }
  if (heart > 16) {print("This is ridiculous, I'm outta here!")
    break
  } 
  if(heart < 5) {print("This is too embarrassing!")
    next
  } 
  print(heart)
}
```

```
## [1] "You're popular!"
## [1] 16
## [1] "Be more visible!"
## [1] 9
## [1] "You're popular!"
## [1] 13
## [1] "Be more visible!"
## [1] 5
## [1] "Be more visible!"
## [1] "This is too embarrassing!"
## [1] "You're popular!"
## [1] "This is ridiculous, I'm outta here!"
```

## Functions

A way to see the components of a function is the args() function:


```r
args(sum)
```

```
## function (..., na.rm = FALSE) 
## NULL
```

**Exclude "NA" From A Calculation**


```r
linkedin <- c(16, 9, 13, 5, NA, 17, 14)
facebook <- c(17, NA, 5, 16, 8, 13, 14)

# Basic average of linkedin
mean(linkedin)
```

```
## [1] NA
```

```r
# Advanced average of linkedin
mean(linkedin, na.rm = TRUE)
```

```
## [1] 12.33333
```

The default setting in the `mean()` function is `na.rm = FALSE`, which means it doesn't exclude the `NA` variables. However, when switched to `TRUE`, the function excludes the `NA` varaibles.

**When Is It Required?**

    mean(x, trim = 0, na.rm = FALSE, ...)

`x` is required; if you do not specify it, R will throw an error. `trim` and `na.rm` are optional arguments: they have a default value which is used if the arguments are not explicitly specified.

***Create A Function***

To create a `function`, assign a variable the function `function(condition){body}`:

```r
pow_two <- function(x) {
  y <- x ^ 2
  print(paste(x, "to the power two equals", y))
  return(y)
}
pow_two(6)
```

```
## [1] "6 to the power two equals 36"
```

```
## [1] 36
```

```r
#NOTE: "y" was defined inside the "pow_two()" function and therefore it is not accessible outside of that function. This is also true for the function's arguments of course - "x" in this case.
```



**Internal Variables of "Function()" are FIXED**

An external varaible can't be entered in to change the internal make-up of a created function:

```r
triple <- function(x) {
  x <- 3*x
  x
}
#Testing whether R updates the variable "a":
a <- 5
triple(a)
```

```
## [1] 15
```

```r
a
```

```
## [1] 5
```

Even though the function `triple(a)` outputted 15, R didn't print the new `a` variable as `15` but as `5`.

**Example from Datacamp:**

```r
likes <- c(10, 18, 4)

interpret <- function(num_views) {
  if (num_views > 15) {
      print("You're popular!")
    return(num_views)
  } else {
      print("Try to be more visible!")
    return(0)
  }
}

interpret(likes[2])
```

```
## [1] "You're popular!"
```

```
## [1] 18
```

```r
interpret(likes[3])
```

```
## [1] "Try to be more visible!"
```

```
## [1] 0
```

**Load an R Package**

There are basically two important functions when it comes to R packages:

`install.packages()` installs a given package.

`library()` which loads packages, i.e. attaches them to the search list on your R workspace.

**Anonymous functions**

An anonymous function is a function that's NOT aasigned a variable(name):

```r
# Named function
triple <- function(x) { 3 * x }

# Anonymous function with same implementation
function(x) { 3 * x }
```

```
## function(x) { 3 * x }
```

## The apply family

**"lapply()"***

`lappy()` applies the function inputted inside the `()` over a vector or list, and spit out a list.

    lapply(X, FUN, ...)
    
For example:


```r
numbers_list <- list(c(17, 28, -2, 9, 22), c(2, -19, 54, 27, 11), c(91, 76, -34, 8, 10))

extremes_avg <- function(x) {
  ( min(x) + max(x) ) / 2
}

# Apply extremes_avg() over numbers_list using lapply()
lapply(numbers_list, extremes_avg)
```

```
## [[1]]
## [1] 13
## 
## [[2]]
## [1] 17.5
## 
## [[3]]
## [1] 28.5
```
  

**"sapply()"***

`sapply()` applies the function inputted inside the `()` over a vector or list, and *try* to arrange the resulting list into an organized array. If not possible, `sapply()` will return the same list as `lapply()` spit out.

    sapply(X, FUN, ...)

Continuing the example above:


```r
# Apply extremes_avg() over numbers_list using sapply()
sapply(numbers_list, extremes_avg)
```

```
## [1] 13.0 17.5 28.5
```

The outputted result looks more compact than `lapply()`.


**"vapply()"***

`vapply()` applies the function inputted inside the `()` over a vector or list like `lapply()` or `sapply()`. However, with `vapply()`, it requires a specified output format, meaning tell it what result type it should spit out:

    vapply(X, FUN, FUN.VALUE, ..., USE.NAMES = TRUE)
    
The `FUN.VALUE` argument expects a template for the return argument of this function `FUN.` `USE.NAMES` is `TRUE` by default; in this case `vapply()` tries to generate a named array, if possible.

Example:

```r
# Definition of the basics() function
basics <- function(x) {
  c(min = min(x), mean = mean(x), median = median(x), max = max(x))
}

# Fix the error:
vapply(numbers_list, basics, numeric(4))
```

```
##        [,1] [,2]  [,3]
## min    -2.0  -19 -34.0
## mean   14.8   15  30.2
## median 17.0   11  10.0
## max    28.0   54  91.0
```

In this example, if `numerics` specified was `3` instead of `4`, the code would **NOT** run and give an error because `vapply()` function requires a specific output format. In this case, the `basics` function has 4 elements: `min`, `mean`, `median`, and `max`, therefore, `vapply()` need to specified as `numeric(4)`.

## Utilities

**Mathematical Utilities**

`abs()`  : Calculate the absolute value.

`sum()`  : Calculate the sum of all the values in a data structure.

`mean()` : Calculate the arithmetic mean.

`round()`: Round the values to 0 decimal places by default.

Example: 


```r
digits <- c(1.9, -2.6, 4.0, -9.5, -3.4, 7.3)

# Sum of absolute rounded values of errors
sum(abs(round(digits)))
```

```
## [1] 29
```

**Data Utilities**

`seq()`: Generate sequences, by specifying the `from`, `to`, and `by` arguments.

`rep()`: Replicate elements of vectors and lists.

```r
rep(seq(1, 7, by = 2), times = 7)
```

```
##  [1] 1 3 5 7 1 3 5 7 1 3 5 7 1 3 5 7 1 3 5 7 1 3 5 7 1 3 5 7
```

`sort()`: Sort a vector in ascending order by default. Works on numerics, but also on character strings and logicals.

`rev()`: Reverse the elements in a data structures for which reversal is defined.

`str()`: Display the structure of any R object.

`append()`: Merge vectors or lists.

`is.*()`: Check for the class of an R object.

`as.*()`: Convert an R object from one class to another.

`unlist()`: Flatten (possibly embedded) lists to produce a vector.

Example:

```r
linkedin <- list(16, 9, 13, 5, 2, 17, 14)
facebook <- list(17, 7, 5, 16, 8, 13, 14)

# Convert linkedin and facebook to a vector: li_vec and fb_vec
li_vec <- unlist(linkedin)
fb_vec <- unlist(facebook)

# Append fb_vec to li_vec: social_vec
social_vec <- append(li_vec, fb_vec)

# Sort social_vec
sort(social_vec, decreasing = TRUE)
```

```
##  [1] 17 17 16 16 14 14 13 13  9  8  7  5  5  2
```

**Regular Expressions**

 Regular expressions can be used to see whether a pattern exists inside a character string or a vector of character strings.
 
<center>**"grep()" and "grepl()"**</center>

`grepl()`: the `l` in `grepl()` stands for logical, which indicates that this function returns `TRUE` when a pattern is found in the corresponding character string.

`grep()`: returns a vector contains the location of the character strings(by order) that contains the pattern searched for.

    The caret:       `^`, to match the content located in the start of a string.
    
    The dollar-sign: `$`, to match the content located in the end of a string.


```r
emails <- c("john.doe@ivyleague.edu", "education@world.gov", "dalai.lama@peace.org",
            "invalid.edu", "quant@bigdatacollege.edu", "cookie.monster@sesame.tv")

#Search for the email addresses in the vector above that contains "@", anything in between, and "edu":
grepl(pattern = "@.*\\.edu$", emails)
```

```
## [1]  TRUE FALSE FALSE FALSE  TRUE FALSE
```

```r
grep(pattern = "@.*\\.edu$", emails)
```

```
## [1] 1 5
```

```r
# Subset emails using hits
emails[grep(pattern = "@.*\\.edu$", emails)]
```

```
## [1] "john.doe@ivyleague.edu"   "quant@bigdatacollege.edu"
```

`.*`: can be read as "any character that is matched zero or more times".Both the dot and the asterisk are metacharacters. 

`\\`: is like a cut-off. It put a separation wall between the `.*` and `.edu`.

<center>**"sub()" and "gsub()"**</center>

`sub()` and `gsub()`can specify a `replacement` argument. If inside the character vector `x`, the regular expression `pattern` is found, the matching element(s) will be replaced with `replacement`.`sub()` only replaces the first match, whereas `gsub()` replaces all matches.


```r
emails <- c("john.doe@ivyleague.edu", "education@world.gov", "dalai.lama@peace.org",
            "invalid.edu", "quant@bigdatacollege.edu", "cookie.monster@sesame.tv")

sub(pattern = "@.*\\.edu$", replacement = "@datacamp.edu", emails)
```

```
## [1] "john.doe@datacamp.edu"    "education@world.gov"     
## [3] "dalai.lama@peace.org"     "invalid.edu"             
## [5] "quant@datacamp.edu"       "cookie.monster@sesame.tv"
```

```r
#The [1] and [5] elements has been changed.
```

`\\s`: Match a space. The "s" is normally a character, escaping it `\\` makes it a metacharacter.

[0-9]+: Match the numbers 0 to 9, at least once (+).

([0-9]+): The parentheses are used to NOT confuse the `pattern` matching criterias.

The `\\1`: is to input the regular expression `[0-9]+` matched  into the `replacement` argument.


```r
awards <- c("Won 1 Oscar.", "Another 9 wins & 24 nominations.", 
            "2 wins & 3 nominations.", 
            "Nominated for 2 Golden Globes. 1 more win & 2 nominations.")

sub(".*\\s([0-9]+)\\snomination.*$", "\\1", awards)
```

```
## [1] "Won 1 Oscar." "24"           "3"            "2"
```

The logic behind the `pattern` criteria is as follow: skip any character and then a space between the number and the word "nomination". Then, skip again any character after the word "nomination".

**Date And Time**

Dates are represented by `Date` objects. Times are represented by `POSIXct` objects.
However, dates and times are simple numerical values. `Date` objects store the number of *days* since the 1st of January in 1970. `POSIXct` store the number of *seconds* since the 1st of January in 1970.


```r
# Get the current date: today
today <- Sys.Date()
today
```

```
## [1] "2022-06-03"
```

```r
# See what today looks like under the hood
unclass(today)
```

```
## [1] 19146
```

```r
# Get the current time: now
now <- Sys.time()
now
```

```
## [1] "2022-06-03 17:38:41 CDT"
```

```r
# See what now looks like under the hood
unclass(now)
```

```
## [1] 1654295922
```

<center>**Date Formats**</center>

Use the `as.Date()` function to create a `Date` object from a simple character string.

    %Y: 4-digit year (1982)
    %y: 2-digit year (82)
    %m: 2-digit month (01)
    %d: 2-digit day of the month (13)
    %A: weekday (Wednesday)
    %a: abbreviated weekday (Wed)
    %B: month (January)
    %b: abbreviated month (Jan)


```r
# Definition of character strings representing dates
str1 <- "May 23, '96"
str2 <- "2012-03-15"
str3 <- "30/January/2006"

# Convert the strings to dates: date1, date2, date3
date1 <- as.Date(str1, format = "%b %d, '%y")
date2 <- as.Date(str2, format = "%Y-%m-%d")
date3 <- as.Date(str3, format = "%d/%B/%Y")

# Convert dates to formatted strings
format(date1, "%A")
```

```
## [1] "Thursday"
```

```r
format(date2, "%d")
```

```
## [1] "15"
```

```r
format(date3, "%b %Y")
```

```
## [1] "Jan 2006"
```

<center>**Date Calculations**</center>

Both `Date` and `POSIXct` objects are represented by simple numerical values under the hood.


```r
today <- Sys.Date()
today + 1
```

```
## [1] "2022-06-04"
```

```r
today - 1
```

```
## [1] "2022-06-02"
```

```r
as.Date("2015-03-12") - as.Date("2015-02-27")
```

```
## Time difference of 13 days
```

<center>**Time Formats**</center>

Use `as.POSIXct()` to convert a character string to a `POSIXct` object.

    %H: hours as a decimal number (00-23)
    %I: hours as a decimal number (01-12)
    %M: minutes as a decimal number
    %S: seconds as a decimal number
    %T: shorthand notation for the typical format %H:%M:%S
    %p: AM/PM indicator
    
For a full list of conversion symbols, consult the `strptime` documentation in the console: `?strptime`


```r
# Definition of character strings representing times
str1 <- "May 23, '96 hours:23 minutes:01 seconds:45"
str2 <- "2012-3-12 14:23:08"

# Convert the strings to POSIXct objects: time1, time2
time1 <- as.POSIXct(str1, format = "%B %d, '%y hours:%H minutes:%M seconds:%S")
time2 <- as.POSIXct(str2, format = "%Y-%m-%d %H:%M:%S")

# Convert times to formatted strings
format(time1, "%M")
```

```
## [1] "01"
```

```r
format(time2, "%I:%M %p")
```

```
## [1] "02:23 PM"
```

<center>**Time Calculations**</center>
Examples of doing calculations with `POSIXct` objects: 

```r
now <- Sys.time()
now + 3600          # add an hour
```

```
## [1] "2022-06-03 18:38:41 CDT"
```

```r
now - 3600 * 24     # subtract a day
```

```
## [1] "2022-06-02 17:38:41 CDT"
```


```r
birth <- as.POSIXct("1879-03-14 14:37:23")
death <- as.POSIXct("1955-04-18 03:47:12")
einstein <- death - birth
einstein
```

```
## Time difference of 27792.56 days
```

