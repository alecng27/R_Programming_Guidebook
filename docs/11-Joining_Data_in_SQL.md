# Joining Data in SQL

<https://learn.datacamp.com/courses/joining-data-in-postgresql>


## Introduction to joins

**Inner join**


```r
SELECT *
FROM left_table
INNER JOIN right_table
ON left_table.id = right_table.id;
```

Instead of writing the full table name, you can use table aliasing as a shortcut. For tables you also use `AS` to add the alias immediately after the table name with a space.


```r
SELECT c1.name AS city, c2.name AS country
FROM cities AS c1
INNER JOIN countries AS c2
ON c1.country_code = c2.code;
```

Notice that to select a field in your query that appears in multiple tables, you'll need to identify which table/table alias you're referring to by using a `.` in your `SELECT` statement.

The ability to combine multiple joins in a single query is a powerful feature of SQL:


```r
# Select fields
SELECT c.code, name, region, e.year, fertility_rate, unemployment_rate
# From countries (alias as c)
  FROM countries AS c
# Join to populations (as p)
  INNER JOIN populations AS p
# Match on country code
    ON c.code = p.country_code
# Join to economies (as e)
  INNER JOIN economies AS e
# Match on country code
    ON c.code = e.code;
```

**Inner join with USING**

When joining tables with a common field name, e.g.


```r
SELECT *
FROM countries
  INNER JOIN economies
    ON countries.code = economies.code
```

You can use `USING` as a shortcut:


```r
SELECT *
FROM countries
  INNER JOIN economies
    USING(code)
```

**CASE WHEN and THEN**

Often it's useful to look at a numerical field not as raw data, but instead as being in different categories or groups.

You can use `CASE` with `WHEN`, `THEN`, `ELSE`, and `END` to define a new grouping field.


```r
SELECT name, continent, code, surface_area,
# First case
    CASE WHEN surface_area > 2000000 THEN 'large'
# Second case
        WHEN surface_area > 350000 THEN 'medium'
# Else clause + end
        ELSE 'small' END
# Alias name
        AS geosize_group
# From table
FROM countries;
```


## Outer joins and cross joins

**Left join**


```r
# Select name, region, and gdp_percapita
SELECT name, region, gdp_percapita
# From countries (alias as c)
FROM countries AS c
# Left join with economies (alias as e)
  LEFT JOIN economies AS e
# Match on code fields
    ON c.code = e.code
# Focus on 2010
WHERE year = 2010;
```

**Right join**

Right joins aren't as common as left joins. One reason why is that you can always write a right join as a left join.


```r
SELECT cities.name AS city, urbanarea_pop, countries.name AS country,
       indep_year, languages.name AS language, percent
FROM languages
  RIGHT JOIN countries
    ON languages.code = countries.code
  RIGHT JOIN cities
    ON countries.code = cities.country_code
ORDER BY city, language;
```

**Full join**


```r
SELECT name AS country, code, region, basic_unit
# From countries
FROM countries
# Join to currencies
  FULL JOIN currencies
# Match on code
    USING (code)
# Where region is North America or null
WHERE region = 'North America' OR region IS NULL
# Order by region
ORDER BY region;
```


## Set theory clauses

**UNION**

`UNION` includes every record in both tables but doesn't double count the matched/overlapped ones.


```r
# Select fields from 2010 table
SELECT *
# From 2010 table
  FROM economies2010
# Set theory clause
    UNION
# Select fields from 2015 table
SELECT *
# From 2015 table
  FROM economies2015
# Order by code and year
ORDER BY code, year;
```

**UNION ALL**

`UNION ALL` includes every record in both tables and does replicate the matched/overlapped ones.


```r
# Select fields
SELECT code, year
# From economies
  FROM economies
# Set theory clause
	UNION ALL
# Select fields
SELECT country_code, year
# From populations
  FROM populations
# Order by code, year
ORDER BY code, year;
```

**INTERSECT**

`INTERSECT` results in only those matched/overlapped.


```r
# Select fields
SELECT code, year
# From economies
  FROM economies
# Set theory clause
	INTERSECT
# Select fields
SELECT country_code, year
# From populations
  FROM populations
# Order by code and year
ORDER BY code, year;
```

**EXCEPT**

`EXCEPT` results in only those doesn't matched/overlapped.


```r
# Select field
SELECT name
# From cities
  FROM cities
#Set theory clause
	EXCEPT
# Select field
SELECT capital
# From countries
  FROM countries
# Order by result
ORDER BY name;
```

**Diagnosing problems using anti-join**

Another powerful join in SQL is the anti-join. It is particularly useful in identifying which records are causing an incorrect number of records to appear in join queries.


```r
# 3. Select fields
SELECT code, name
# 4. From Countries
  FROM countries
# 5. Where continent is Oceania
  WHERE continent = 'Oceania'
# 1. And code not in
  	AND code NOT IN
# 2. Subquery
  	(SELECT code
  	 FROM currencies);
```


## Subqueries

**Subquery inside WHERE**


```r
# Select fields
SELECT *
# From populations
  FROM populations
# Where life_expectancy is greater than
WHERE life_expectancy >
# 1.15 * subquery
  1.15 * (SELECT AVG(life_expectancy)
   FROM populations
   WHERE year = 2015) AND
  year = 2015;
```

**Subquery inside SELECT**

The code given in `query.sql` selects the top nine countries in terms of number of `cities` appearing in the cities table. Recall that this corresponds to the most populous cities in the world. 


```r
SELECT countries.name AS country, COUNT(*) AS cities_num
FROM cities
INNER JOIN countries
ON countries.code = cities.country_code
GROUP BY country
ORDER BY cities_num DESC, country
LIMIT 9;
```

 convert the code to get the same result as the code shown:
 

```r
SELECT countries.name AS country,
(SELECT COUNT(*)
FROM cities
WHERE countries.code = cities.country_code) AS cities_num
FROM countries
ORDER BY cities_num DESC, country
LIMIT 9;
```

**Subquery inside FROM**

Begin by determining for each country code how many `languages` are listed in the languages table using `SELECT`, `FROM`, and `GROUP BY`.

Alias the aggregated field as `lang_num`.


```r
-- Select fields (with aliases)
SELECT code, COUNT(name) AS lang_num
-- From languages
FROM languages
-- Group by code
GROUP BY code;
```

Include the previous query (aliased as `subquery`) as a subquery in the `FROM` clause of a new query.

Select the local name of the country from `countries.`

Also, select `lang_num` from subquery.

Make sure to use `WHERE` appropriately to match `code` in `countries` and in `subquery`.

Sort by `lang_num` in descending order.


```r
# Select fields
SELECT local_name, subquery.lang_num
# From countries
FROM countries,
# Subquery (alias as subquery)
(SELECT code, COUNT(name) AS lang_num
FROM languages
GROUP BY code) AS subquery
# Where codes match
WHERE countries.code = subquery.code
# Order by descending number of languages
ORDER BY lang_num DESC;
```



