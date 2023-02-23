# Web Scraping in R

<https://learn.datacamp.com/courses/web-scraping-in-r>


## Introduction to HTML and Web Scraping

**Read in HTML**

A necessary package to read HTML is `rvest`:


```r
library(rvest)
library(tidyverse)
library(httr)
library(xml2)
```

Take the `html_excerpt_raw` variable and turn it into an HTML document that R understands using a function from the `rvest` package:


```r
html_excerpt_raw <- '
<html> 
  <body> 
    <h1>Web scraping is cool</h1>
    <p>It involves writing code – be it R or Python.</p>
    <p><a href="https://datacamp.com">DataCamp</a> 
		has courses on it.</p>
  </body> 
</html>'

# Turn the raw excerpt into an HTML document R understands
html_excerpt <- read_html(html_excerpt_raw)
html_excerpt
```

```
## {html_document}
## <html>
## [1] <body> \n    <h1>Web scraping is cool</h1>\n    <p>It involves writing co ...
```

Use the `xml_structure()` function to get a better overview of the tag hierarchy of the HTML excerpt:


```r
xml_structure(html_excerpt)
```

```
## <html>
##   <body>
##     {text}
##     <h1>
##       {text}
##     {text}
##     <p>
##       {text}
##     {text}
##     <p>
##       <a [href]>
##         {text}
##       {text}
##     {text}
```

`read_html(url)` : scrape HTML content from a given URL

`html_nodes()`: identifies HTML wrappers.

`html_nodes(“.class”)`: calls node based on CSS class

`html_nodes(“#id”)`: calls node based on <div> id

`html_nodes(xpath=”xpath”)`: calls node based on xpath (we’ll cover this later)

`html_table()`: turns HTML tables into data frames

`html_text()`: strips the HTML tags and extracts only the text


## Navigation and Selection with CSS

**Select multiple HTML types**

CSS can be used to style a web page. In the most basic form, this happens via type selectors, where styles are defined for and applied to all HTML elements of a certain type. In turn, you can also use type selectors to scrape pages for specific HTML elements.

CSS can also combine multiple type selectors via a comma, i.e. with `html_nodes("type1, type2")`. This selects all elements that have `type1` or `type2`.


```r
languages_raw_html <- '<html> 
  <body> 
    <div>Python is perfect for programming.</div>
    <p>Still, R might be better suited for data analysis.</p>
    <small>(And has prettier charts, too.)</small>
  </body> 
</html>'

# Read in the HTML
languages_html <- read_html(languages_raw_html)
# Select the div and p tags and print their text
languages_html %>%
	html_nodes('div, p') %>%
	html_text()
```

```
## [1] "Python is perfect for programming."                
## [2] "Still, R might be better suited for data analysis."
```

**CSS classes and IDs**

IDs should be unique across a web page. If you can make sure this is the case, it can reduce the complexity of your scraping selectors drastically.

Here's the structure of an HTML page you might encounter in the wild:


```r
structured_html <- "<html>
  <body>
    <div id = 'first'>
      <h1 class = 'big'>Joe Biden</h1>
      <p class = 'first blue'>Democrat</p>
      <p class = 'second blue'>Male</p>
    </div>
    <div id = 'third'>
      <h1 class = 'big'>Donald Trump</h1>
      <p class = 'first red'>Republican</p>
      <p class = 'second red'>Male</p>
    </div>
  </body>
</html>"

structured_html <- read_html(structured_html)
```

Using `html_nodes()`, find the shortest possible selector to select the first `div` in `structured_html`:


```r
# Select the first div
structured_html %>%
  html_nodes('#first')
```

```
## {xml_nodeset (1)}
## [1] <div id="first">\n      <h1 class="big">Joe Biden</h1>\n      <p class="f ...
```

**Select the last child with a pseudo-class**

In the following HTML showing the author of a text in the last paragraph, there are two groups of `p` nodes:


```r
nested_html <- "<html>
  <body>
    <div>
      <p class = 'text'>A sophisticated text [...]</p>
      <p class = 'text'>Another paragraph following [...]</p>
      <p class = 'text'>Author: T.G.</p>
    </div>
    <p>Copyright: DC</p>
  </body>
</html>"

nested_html <- read_html(nested_html)
```

use the pseudo-class that selects the last child to scrape the last `p` in each group.


```r
# Select the last child of each p group
nested_html %>%
	html_nodes('p:last-child')
```

```
## {xml_nodeset (2)}
## [1] <p class="text">Author: T.G.</p>
## [2] <p>Copyright: DC</p>
```

As this selected the last `p` node from both groups, make use of the `text` class to get only the authorship information.



```r
# This time for real: Select only the last node of the p's wrapped by the div
nested_html  %>% 
	html_nodes('p.text:last-child')
```

```
## {xml_nodeset (1)}
## [1] <p class="text">Author: T.G.</p>
```

**Select direct descendants with the child combinator**

There are cases where selectors like `type`, `class`, or `ID` won't work, for example, if you only want to extract direct descendants of the top `ul` element. For that, you will use the child combinator (`>`).

Here, your goal is to scrape a list of all mentioned computer languages, but without the accompanying information in the sub-bullets:


```r
languages_html_2 <- "<html>
  <ul id = 'languages'>
    <li>SQL</li>
    <ul>    
      <li>Databases</li>
      <li>Query Language</li>
    </ul>
    <li>R</li>
    <ul>
      <li>Collection</li>
      <li>Analysis</li>
      <li>Visualization</li>
    </ul>
    <li>Python</li>
  </ul>
  </html>"

languages_html_2 <- read_html(languages_html_2)
```

First, gather all the `li` elements in the nested list shown above and print their text:


```r
# Extract the text of all list elements
languages_html_2 %>% 
	html_nodes('li') %>% 
	html_text()
```

```
## [1] "SQL"            "Databases"      "Query Language" "R"             
## [5] "Collection"     "Analysis"       "Visualization"  "Python"
```

Extract only direct descendants of the top-level `ul` element, using the child combinator:


```r
# Extract only the text of the computer languages (without the sub lists)
languages_html_2 %>% 
	html_nodes('ul#languages > li') %>% 
	html_text()
```

```
## [1] "SQL"    "R"      "Python"
```

**Not every sibling is the same**

The following HTML code contains two headings followed by some `code` and `span` tags:


```r
code_html <- "<html> 
  <body> 
    <h2 class = 'first'>First example:</h2>
    <code>some = code(2)</code>
    <span>will compile to...</span>
    <code>some = more_code()</code>
    <h2 class = 'second'>Second example:</h2>
    <code>another = code(3)</code>
    <span>will compile to...</span>
    <code>another = more_code()</code>
  </body> 
  </html>"

code_html <- read_html(code_html)
```

Select the first `code` element in the second example using `html_nodes()` with the correct sibling combinator.


```r
# Select only the first code element in the second example
code_html %>% 
	html_nodes('h2.second + code')
```

```
## {xml_nodeset (1)}
## [1] <code>another = code(3)</code>
```

Now select all `code` elements that are in the second example using another type of sibling combinator.


```r
# Select all code elements in the second example
code_html %>% 
	html_nodes('h2.second ~ code')
```

```
## {xml_nodeset (2)}
## [1] <code>another = code(3)</code>
## [2] <code>another = more_code()</code>
```


## Advanced Selection with XPATH

**Select by class and ID with XPATH**


```r
weather_html <- "
<html>
  <body>
    <div id = 'first'>
      <h1 class = 'big'>Berlin Weather Station</h1>
      <p class = 'first'>Temperature: 20°C</p>
      <p class = 'second'>Humidity: 45%</p>
    </div>
    <div id = 'second'>...</div>
    <div id = 'third'>
      <p class = 'first'>Sunshine: 5hrs</p>
      <p class = 'second'>Precipitation: 0mm</p>
    </div>
  </body>
</html>"

weather_html <- read_html(weather_html)
```

Start by selecting all `p` tags in the above HTML using `XPATH`.


```r
# Select all p elements
weather_html %>%
	html_nodes(xpath = '//p')
```

```
## {xml_nodeset (4)}
## [1] <p class="first">Temperature: 20°C</p>
## [2] <p class="second">Humidity: 45%</p>
## [3] <p class="first">Sunshine: 5hrs</p>
## [4] <p class="second">Precipitation: 0mm</p>
```

Now select only the `p` elements with class `second`.

The corresponding CSS selector would be `.second`, so here you need to use a `[@class = ...]` predicate applied to all `p` tags.


```r
# Select p elements with the second class
weather_html %>%
	html_nodes(xpath = '//p[@class = "second"]')
```

```
## {xml_nodeset (2)}
## [1] <p class="second">Humidity: 45%</p>
## [2] <p class="second">Precipitation: 0mm</p>
```

Now select all `p` elements that are children of the element with ID `third`.

The corresponding CSS selector would be `#third > p` – don't forget the universal selector (`*`) before the `@id = ...` predicate and remember that children are selected with a `/`, not a `//`.


```r
# Select p elements that are children of "#third"
weather_html %>%
	html_nodes(xpath = '//*[@id = "third"]/p')
```

```
## {xml_nodeset (2)}
## [1] <p class="first">Sunshine: 5hrs</p>
## [2] <p class="second">Precipitation: 0mm</p>
```

Now select only the `p` element with class `second` that is a direct child of #`third`, again using XPATH.

Here, you need to append to the XPATH from the previous step the `@class` predicate you used in the second step.


```r
# Select p elements with class "second" that are children of "#third"
weather_html %>%
	html_nodes(xpath = '//*[@id = "third"]/p[@class = "second"]')
```

```
## {xml_nodeset (1)}
## [1] <p class="second">Precipitation: 0mm</p>
```

**Use predicates to select nodes based on their children**

Here's almost the same HTML as before. In addition, the third `div` has a `p` child with a `third` class.


```r
weather_html_2 <- "<html>
  <body>
    <div id = 'first'>
      <h1 class = 'big'>Berlin Weather Station</h1>
      <p class = 'first'>Temperature: 20°C</p>
      <p class = 'second'>Humidity: 45%</p>
    </div>
    <div id = 'second'>...</div>
    <div id = 'third'>
      <p class = 'first'>Sunshine: 5hrs</p>
      <p class = 'second'>Precipitation: 0mm</p>
      <p class = 'third'>Snowfall: 0mm</p>
    </div>
  </body>
</html>"

weather_html_2 <- read_html(weather_html_2)
```

With XPATH, something that's not possible with CSS can be done: selecting elements based on the properties of their descendants. For this, predicates may be used.

Using XPATH, select all the `div` elements.


```r
# Select all divs
weather_html_2 %>% 
  html_nodes(xpath = '//div')
```

```
## {xml_nodeset (3)}
## [1] <div id="first">\n      <h1 class="big">Berlin Weather Station</h1>\n     ...
## [2] <div id="second">...</div>
## [3] <div id="third">\n      <p class="first">Sunshine: 5hrs</p>\n      <p cla ...
```

Now select all `div`s with `p` descendants using the predicate notation.


```r
# Select all divs with p descendants
weather_html_2 %>% 
  html_nodes(xpath = '//div[p]')
```

```
## {xml_nodeset (2)}
## [1] <div id="first">\n      <h1 class="big">Berlin Weather Station</h1>\n     ...
## [2] <div id="third">\n      <p class="first">Sunshine: 5hrs</p>\n      <p cla ...
```

Now select `div`s with `p` descendants which have the `third` class.


```r
# Select all divs with p descendants having the "third" class
weather_html_2 %>% 
  html_nodes(xpath = '//div[p[@class = "third"]]')
```

```
## {xml_nodeset (1)}
## [1] <div id="third">\n      <p class="first">Sunshine: 5hrs</p>\n      <p cla ...
```

**Get to know the position() function**

`position()` function is very powerful when used within a predicate. Together with operators, you can basically select any node from those that match a certain path.

You'll try this out with the following HTML excerpt that is available to you via `rules_html`. Let's assume this is a continuously updated website that displays certain Coronavirus rules for a given day and the day after.


```r
rules_html <- "<html>
<div>
  <h2>Today's rules</h2>
  <p>Wear a mask</p>
  <p>Wash your hands</p>
</div>
<div>
  <h2>Tomorrow's rules</h2>
  <p>Wear a mask</p>
  <p>Wash your hands</p>
  <small>Bring hand sanitizer with you</small>
</div>
</html>"

rules_html <- read_html(rules_html)
```

Extract the text of the second `p` in every `div` using XPATH.


```r
# Select the text of the second p in every div
rules_html %>% 
  html_nodes(xpath = '//div/p[position() = 2]') %>%
  html_text()
```

```
## [1] "Wash your hands" "Wash your hands"
```

Now extract the text of every `p` (except the `second`) in every `div`.


```r
# Select every p except the second from every div
rules_html %>% 
  html_nodes(xpath = '//div/p[position() != 2]') %>%
  html_text()
```

```
## [1] "Wear a mask" "Wear a mask"
```

Extract the text of the last three children of the second `div`.

Only use the `>=` operator for selecting these nodes.


```r
# Select the text of the last three nodes of the second div
rules_html %>% 
  html_nodes(xpath = '//div[position() = 2]/*[position() >= 2]') %>%
  html_text()
```

```
## [1] "Wear a mask"                   "Wash your hands"              
## [3] "Bring hand sanitizer with you"
```

**Extract nodes based on the number of their children**

XPATH `count()` function can be used within a predicate to narrow down a selection to these nodes that match a certain children count. This is especially helpful if your scraper depends on some nodes having a minimum amount of children.

You're only interested in `div`s that have exactly one `h2` header and at least two paragraphs. 

Select the desired `div`s with the appropriate XPATH selector, making use of the `count()` function.


```r
# Select only divs with one header and at least two paragraphs
rules_html %>%
	html_nodes(xpath = '//div[count(h2) = 1 and count(p) > 1]')
```

```
## {xml_nodeset (2)}
## [1] <div>\n  <h2>Today's rules</h2>\n  <p>Wear a mask</p>\n  <p>Wash your han ...
## [2] <div>\n  <h2>Tomorrow's rules</h2>\n  <p>Wear a mask</p>\n  <p>Wash your  ...
```

**Select directly from a parent element with XPATH's text()**

extract the `function` information in parentheses into their own column, so you are required to extract a data frame with not two, but three columns: `actors`, `roles`, and `functions`.


```r
roles_html <- "<html>
<table>
 <tr>
  <th>Actor</th>
  <th>Role</th>
 </tr>
 <tr>
  <td class = 'actor'>Jayden Carpenter</td>
  <td class = 'role'><em>Mickey Mouse</em> (Voice)</td>
 </tr>
</table>
</html>"

roles_html <- read_html(roles_html)
```

Extract the `actors` and `roles` from the table using XPATH.


```r
# Extract the actors in the cells having class "actor"
actors <- roles_html %>% 
  html_nodes(xpath = '//table//td[@class = "actor"]') %>%
  html_text()
actors
```

```
## [1] "Jayden Carpenter"
```

```r
# Extract the roles in the cells having class "role"
roles <- roles_html %>% 
  html_nodes(xpath = '//table//td[@class = "role"]/em') %>% 
  html_text()
roles
```

```
## [1] "Mickey Mouse"
```

Then, extract the `function` using the XPATH `text()` function.

Extract only the text with the parentheses, which is contained within the same cell as the corresponding role, and trim leading spaces.


```r
# Extract the functions using the appropriate XPATH function
functions <- roles_html %>% 
  html_nodes(xpath = '//table//td[@class = "role"]/text()') %>%
  html_text(trim = TRUE)
functions
```

```
## [1] "(Voice)"
```

**Combine extracted data into a data frame**

Combine the three vectors `actors`, `roles`, and `functions` into a data frame called `cast` (with columns `Actor`, `Role` and `Function`, respectively).


```r
# Create a new data frame from the extracted vectors
cast <- tibble(
  Actor = actors, 
  Role = roles, 
  Function = functions)

cast
```

```
## # A tibble: 1 × 3
##   Actor            Role         Function
##   <chr>            <chr>        <chr>   
## 1 Jayden Carpenter Mickey Mouse (Voice)
```


## Scraping Best Practices

*httr**

`read_html()` actually issues an **HTTP GET** request if provided with a URL.

The goal of this exercise is to replicate the same query without `read_html()`, but with httr methods instead.

Use only httr functions to replicate the behavior of `read_html()`, including getting the response from Wikipedia and parsing the response object into an HTML document.

Check the resulting HTTP status code with the appropriate httr function.


```r
# Get the HTML document from Wikipedia using httr
wikipedia_response <- GET('https://en.wikipedia.org/wiki/Varigotti')
# Parse the response into an HTML doc
wikipedia_page <- content(wikipedia_response)
# Check the status code of the response
status_code(wikipedia_response)
```

```
## [1] 200
```

a fundamental part of the HTTP system are status codes: They tell you if everything is okay (200) or if there is a problem (404) with your request.

It is good practice to always check the status code of a response before you start working with the downloaded page. For this, you can use the `status_code()` function from the httr() package. 
 
**Add a custom user agent**

There are two ways of customizing your user agent when using httr for fetching web resources:

Locally, i.e. as an argument to the current request method.

Globally via `set_config()`.

Send a GET request to `https://httpbin.org/user-agent` with a custom user agent that says `"A request from a DataCamp course on scraping"` and print the response.

In this step, set the user agent locally.


```r
# Pass a custom user agent to a GET query to the mentioned URL
response <- GET('https://httpbin.org/user-agent', user_agent("A request from a DataCamp course on scraping"))
# Print the response content
content(response)
```

```
## $`user-agent`
## [1] "A request from a DataCamp course on scraping"
```

Now, make that custom user agent (`"A request from a Alec at LU"`) globally available across all future requests with `set_config()`.


```r
# Globally set the user agent to "A request from a DataCamp course on scraping"
set_config(add_headers(`User-Agent` = "A request from a Alec at LU"))
# Pass a custom user agent to a GET query to the mentioned URL
response <- GET('https://httpbin.org/user-agent')
# Print the response content
content(response)
```

```
## $`user-agent`
## [1] "A request from a Alec at LU"
```

**Apply throttling to a multi-page crawler**

You'll find the name of the peak within an element with the ID `"firstHeading"`, while the coordinates are inside an element with class `"geo-dms"`, which is a descendant of an element with ID `"coordinates"`.

Construct a `read_html()` function that executes with a delay of a half second when executed in a loop.


```r
mountain_wiki_pages <- c("https://en.wikipedia.org/w/index.php?title=Mount_Everest&oldid=958643874", "https://en.wikipedia.org/w/index.php?title=K2&oldid=956671989", "https://en.wikipedia.org/w/index.php?title=Kangchenjunga&oldid=957008408")
```


```r
# Define a throttled read_html() function with a delay of 0.5s
read_html_delayed <- slowly(read_html, 
                            rate = rate_delay(0.5))
```

Now write a `for` loop that goes over every page URL in the prepared variable `mountain_wiki_pages` and stores the HTML available at the corresponding Wikipedia URL into the `html` variable


```r
# Construct a loop that goes over all page urls
for(page_url in mountain_wiki_pages){
  # Read in the html of each URL with a delay of 0.5s
  html <- read_html_delayed(page_url)
}
```

Finally, extract the name of the peak as well as its coordinates using the correct CSS selectors given above and store it in `peak` and `coords`.


```r
  # Extract the name of the peak and its coordinates
  peak <- html %>% 
  	html_node("#firstHeading") %>% html_text()
  coords <- html %>% 
    html_node("#coordinates .geo-dms") %>% html_text()
  print(paste(peak, coords, sep = ": "))
}
```

Merge all the code chunks above to make it functional:


```r
# Define a throttled read_html() function with a delay of 0.5s
read_html_delayed <- slowly(read_html, 
                            rate = rate_delay(0.5))
# Construct a loop that goes over all page urls
for(page_url in mountain_wiki_pages){
  # Read in the html of each URL with a delay of 0.5s
  html <- read_html_delayed(page_url)
  # Extract the name of the peak and its coordinates
  peak <- html %>% 
  	html_node("#firstHeading") %>% html_text()
  coords <- html %>% 
    html_node("#coordinates .geo-dms") %>% html_text()
  print(paste(peak, coords, sep = ": "))
}
```

```
## [1] "Mount Everest: 27°59′17″N 86°55′31″E"
## [1] "K2: 35°52′57″N 76°30′48″E"
## [1] "Kangchenjunga: 27°42′09″N 88°08′48″E"
```



