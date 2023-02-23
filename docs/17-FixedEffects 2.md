# Fixed Effects

[Solutions](https://moodle.lawrence.edu/pluginfile.php/691081/mod_resource/content/3/17-FixedEffects-SOLN.html)

This file demonstrates three approaches to estimating "fixed effects" models (remember this is what economists call "fixed effects", but other disciplines use "fixed effects" to refer to something different). We're going to use the [`wbstats` package](http://nset-ornl.github.io/wbstats/) to download country-level data from the World Bank for 2015 - 2018 (the most recently available for the variables I chose). We'll then estimate models with country fixed effects, with year fixed effects, and with both country and year fixed effects. We'll estimate each model three ways: using the within transformation, using dummy variables, and using the `plm` package to apply the within transformation for us. If you don't know what these are, go through LN5 first. 

Note that the model we'll estimate isn't a great model in terms of producing interesting, reliable result. This is by design. Part of this chapter is introducing you to another API you can use to get data. You've worked with the US Census Bureau's API to get data on US counties. Now you have experience getting data on countries from the World Bank. You are free to use either source for data for your RP. If the model here was a great one, it might take away something you want to do for your RP. This way you get experience with fixed effects models and with getting data from the World Bank, but don't waste any potential ideas you might have for your RP. So just don't read too much into the results. They like suffer from reverse causation and omitted variable bias (both violations of ZCM). If you're interested in cross-country variation in life expectancy you'll need to dig much deeper than we'll go in this chapter. 



```r
library(wbstats) # To get data from the World Bank API
library(plm) # To estimate fixed effects models
library(formattable) # to make colorful tables similar to Excel's conditional formatting
library(stargazer)
library(tidyverse)

## Set how many decimals at which R starts to use scientific notation 
options(scipen=3)

# This shows you a lot of what's available from the World Bank API
## listWBinfo <- wb_cachelist

# List of countries in the World Bank data
countryList <- wb_countries()

# List of all available variables (what they call "indicators")
availableIndicators <- wb_cachelist$indicators

## Sometimes it's easier to look through the indicator list if you write it to CSV and open it in Excel (you can do the same thing with the US Census data). The following does that: 
# write.csv(select(availableIndicators,indicator_id, indicator, indicator_desc),"indicators.csv")

## NOTE: if you use any of these (the full list of variables, exporting it to CSV), make sure you do NOT leave that in your final code. It doesn't belong in your RMD file. I just put these thigns in here so you see how to get to this information. 

## We'll use the following variables: 
# SP.DYN.LE00.IN	Life expectancy at birth, total (years)
# NY.GDP.PCAP.KD	GDP per capita (constant 2016 US$)
# SP.POP.TOTL	Population, total
# SP.POP.TOTL.FE.ZS	Population, female (% of total population)
# SP.RUR.TOTL.ZS	Rural population (% of total population)


## Create named vector of indicators to download
indicatorsToDownload <- c(
  lifeExp = "SP.DYN.LE00.IN", 
  gdpPerCapita ="NY.GDP.PCAP.KD", 
  pop = "SP.POP.TOTL",
  pctFemale = "SP.POP.TOTL.FE.ZS",
  pctRural = "SP.RUR.TOTL.ZS"
)

## Download descriptions of on World Bank indicators (i.e., variables)
indicatorInfo <- availableIndicators %>% 
                  filter(indicator_id %in% indicatorsToDownload)


## Build description of our variables that we'll output in the HTML body
sDesc <- ""
for(i in 1:nrow(indicatorInfo)){
  sDesc <- paste0(sDesc
                  ,"<b>",
                  indicatorInfo$indicator[i],
                  " (",indicatorInfo$indicator_id[i]
                  , ")</b>: "
                  ,indicatorInfo$indicator_desc[i]
                  ,"<br>")
}

## Download data
mydataOrig <- wb_data(indicatorsToDownload, 
                      start_date = 2015, 
                      end_date = 2018)

## get vector of TRUE and FALSE where FALSE indicates there's one or more NA
noNAs <- complete.cases(mydataOrig)

## When writing this code, I first checked how many rows do have NAs, and then out of how many rows 
# sum(noNAs)
## out of how many rows:
# nrow(noNAs)

## keep rows without any NA
mydata <- mydataOrig[noNAs,]

## get count of rows for each country
countOfYearsByCountry <-  mydata %>% count(country)

## merge the count variable with the data
mydata <- inner_join(mydata,countOfYearsByCountry, by="country")

## keep only countries that have all 4 years complete
mydata <- mydata %>% filter(n==4)

## drop count variable (since all are now 4)
mydata <- mydata %>% select(-n)


## For the purposes of this chapter, lets only examine one group of countries 
## so that we can output results without it taking up hundreds of lines
## If this weren't a BP chapter we wouldn't do this

## Merge in country info (e.g., region)
mydata <- inner_join(mydata,select(countryList,country,region),by="country")

## Keep only region "Latin America & Caribbean" (so we end up with only 31 countries)
mydata <- mydata %>% filter(region == "Latin America & Caribbean") %>% select(-region)

mydata <- mydata %>% rename(year=date)

## Change scale of variables. This re-scales regression coefficients (instead of getting 0.00000123)
####  Measure population in millions of people instead of people
####  Measure GDP per Capita in thousands of 2010 US $ (instead of 2010 US $)
mydata <- mydata %>% mutate(pop=pop/1000000, 
                            gdpPerCapita=gdpPerCapita/1000)
```




```r
mydata %>% select(lifeExp, gdpPerCapita, pop, pctFemale, pctRural) %>% as.data.frame() %>%
  stargazer(., type = "html",summary.stat = c("n","mean","sd", "min", "p25", "median", "p75", "max"))
```


<table style="text-align:center"><tr><td colspan="9" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Statistic</td><td>N</td><td>Mean</td><td>St. Dev.</td><td>Min</td><td>Pctl(25)</td><td>Median</td><td>Pctl(75)</td><td>Max</td></tr>
<tr><td colspan="9" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">lifeExp</td><td>128</td><td>74.921</td><td>3.455</td><td>62.485</td><td>73.244</td><td>74.910</td><td>77.174</td><td>80.095</td></tr>
<tr><td style="text-align:left">gdpPerCapita</td><td>128</td><td>10.659</td><td>8.200</td><td>1.387</td><td>5.321</td><td>8.248</td><td>13.817</td><td>35.074</td></tr>
<tr><td style="text-align:left">pop</td><td>128</td><td>18.791</td><td>41.175</td><td>0.094</td><td>0.516</td><td>5.611</td><td>12.407</td><td>209.469</td></tr>
<tr><td style="text-align:left">pctFemale</td><td>128</td><td>50.656</td><td>0.931</td><td>49.071</td><td>49.965</td><td>50.614</td><td>51.131</td><td>53.114</td></tr>
<tr><td style="text-align:left">pctRural</td><td>128</td><td>35.487</td><td>21.253</td><td>4.279</td><td>19.814</td><td>33.622</td><td>47.630</td><td>81.485</td></tr>
<tr><td colspan="9" style="border-bottom: 1px solid black"></td></tr></table>



## Variables

Variable descriptions from the World Bank API. **These descriptions do not reflect two changes we made (that are reflected in the table of summary statistics above and in the regression results that follow): population is measured in millions of people and GDP per capita is measured in thousands of 2010 US dollars.**

<b>GDP per capita (constant 2010 US$) (NY.GDP.PCAP.KD)</b>: GDP per capita is gross domestic product divided by midyear population. GDP is the sum of gross value added by all resident producers in the economy plus any product taxes and minus any subsidies not included in the value of the products. It is calculated without making deductions for depreciation of fabricated assets or for depletion and degradation of natural resources. Data are in constant 2010 U.S. dollars.<br><b>Life expectancy at birth, total (years) (SP.DYN.LE00.IN)</b>: Life expectancy at birth indicates the number of years a newborn infant would live if prevailing patterns of mortality at the time of its birth were to stay the same throughout its life.<br><b>Population, total (SP.POP.TOTL)</b>: Total population is based on the de facto definition of population, which counts all residents regardless of legal status or citizenship. The values shown are midyear estimates.<br><b>Population, female (% of total population) (SP.POP.TOTL.FE.ZS)</b>: Female population is the percentage of the population that is female. Population is based on the de facto definition of population, which counts all residents regardless of legal status or citizenship.<br><b>Rural population (% of total population) (SP.RUR.TOTL.ZS)</b>: Rural population refers to people living in rural areas as defined by national statistical offices. It is calculated as the difference between total population and urban population.<br>

Source of data definitions: [`wbstats` package](http://nset-ornl.github.io/wbstats/)

(Note that this is not how you would describe your variables in a paper, but for the purposes of this assignment, it's an easy way to get an accurate description of each variable)



## OLS

Our focus is fixed effects models, but often when estimating fixed effects models we also estimate regular OLS without fixed effects for comparison.


```r
ols <- lm(data=mydata,lifeExp~gdpPerCapita+pop+pctFemale+pctRural)
```



## Country Fixed Effects


Our basic OLS model is the following:
$$
lifeExp_{it} = \beta_0+\beta_1 gdpPerCapita_{it} + \beta_2 pop_{it} + \beta_3 pctFemale_{it} + \beta_4 pctRural_{it} + v_{it}
$$
To save on notation, we'll use generic variables (and I wrote out the "composite error term"), i.e., 


$$
y_{it} = \beta_0+\beta_1 x_{1it} + \beta_2 x_{2it} + \beta_3 x_{2it} + \beta_4 x_{4it} + (c_i + u_{it})
$$



Below there are 3 equations. The first is for country $i$ in year $t$ (the same as the one above). The second equation is the average over the four years for country $i$, where $\bar{y}_{i}=\sum_{t=2015}^{2018}y_{it}$ is the average value of $y_{it}$ over the 4 years for country $i$, $\bar{x}_{ji}=\sum_{t=2015}^{2018}x_{jti}$ is the average value of $x_{jti}$ over the 4 years for country $i$ for the four explanatory variables $j\in\{1,2,3,4\}$, $\bar{c}_{i}=\sum_{t=2015}^{2018}c_{i}=c_i$ is the average value of $c_{i}$ over the 4 years for country $i$ (which just equals $c_i$ because $c_i$ is the same in all years for country $i$), and $\bar{u}_{i}=\sum_{t=2015}^{2018}u_{it}$ is the average value of $u_{it}$ over the 4 years for country $i$. For the final equation, subtract country $i$'s average from the value in each year $t$. 

$$
\begin{align}
y_{it} &= \beta_0+\beta_1 x_{1it} + \beta_2 x_{2it} + \beta_3 x_{2it} + \beta_4 x_{4it} + (c_i + u_{it})
\\
\bar{y}_{i} &= \beta_0+\beta_1 \bar{x}_{1i} + \beta_2 \bar{x}_{2i} + \beta_3 \bar{x}_{3i} + \beta_4 \bar{x}_{4i}  + (\bar{c}_i + \bar{u}_{i})
\\
y_{it}-\bar{y}_{i} &= (\beta_0-\beta_0)+\beta_1 (x_{1it}-\bar{x}_{1i}) + \beta_2 (x_{2it}-\bar{x}_{2i}) + \beta_3 (x_{3it}-\bar{x}_{3i})
\\ &\hspace{6cm} + \beta_4 (x_{4it}-\bar{x}_{4i})  + (c_i-\bar{c}_i + u_{it}-\bar{u}_{i})
\end{align}
$$

This final equation simplifies to the "within transformation" for country $i$,
$$
y_{it}-\bar{y}_{i} = \beta_1 (x_{1it}-\bar{x}_{1i}) + \beta_2 (x_{2it}-\bar{x}_{2i}) + \beta_3 (x_{3it}-\bar{x}_{3i}) + \beta_4 (x_{4it}-\bar{x}_{4i})  + (u_{it}-\bar{u}_{i})
$$
because $\beta_0-\beta_0=0$ and $c_i-\bar{c}_i=0$, where $\bar{c}_i=c_i$ because $c_i$ is the same in all years for country $i$. Mathematically, this is why the fixed effects model allows us to control for unobservable factors that do not change of time (or whatever is measured by $t=1,,,,.T$). If $c_i$ is not constant for all time periods, then $\bar{c}_i=c_i$ isn't correct and it doesn't drop out of the final equation. That means it remains in the equations we estimate, and our coefficients are biased. 

At the end of this file there are tables that demonstrate the within transformation for our dataset. There is a table for each variable. Look at the table for [Life expectancy]. Find the row for Argentina (iso3c code ARG). It's average value of life expectancy is 76.3.  In 2015, their value was 76.07, which is 0.23 below Argentina's four-year average value of 76.3. In 2016, Argentina's life expectancy was 76.22, which is 0.08 above Argentina's four-year average. Below this table for life expectancy is a similar table for each explanatory variable. When economists say a model has country "fixed effects", they mean estimating an OLS regression using data transformed by this "within" transformation.


Alternatively, a model with country "fixed effects" can be estimated using the original OLS equation with the addition of a dummy variable for each country (omitting one).

$$
y_{it} = \beta_0+\beta_1 x_{1it} + \beta_2 x_{2it} + \beta_3 x_{2it} + \beta_4 x_{4it} + \sum_{i=2}^{50}\sigma_idC_i + (c_i + u_{it})
$$

where $dC_i$ is a dummy variable with a value of 1 if that observation is country $i$ and equals 0 otherwise (and $\sigma_i$ is the coefficient on dummy variable $dC_i$).

These two models, the "within transformation" and the model with a dummy variable for each country, are mathematically and empirically equivalent. To see that they are empirically equivalent, we'll estimate both models and compare the results. Note that the standard errors and $R^2$ values are not equivalent, as discussed below.




```r
## Dummy variable for each country (it automatically omits one)
countryDummies <- lm(lifeExp~gdpPerCapita+pop+pctFemale+pctRural+factor(country),data=mydata)

## Within transformation (subtract country averages from each observation for each variable)

## Create Country Averages
mydata <- mydata %>%
  group_by(country) %>%
  mutate(cAvg_lifeExp=mean(lifeExp),
         cAvg_gdpPerCapita=mean(gdpPerCapita),
         cAvg_pop=mean(pop),
         cAvg_pctFemale=mean(pctFemale),
         cAvg_pctRural=mean(pctRural)
         ) %>%
  ungroup()

## Within transformation
mydataCountry <- mydata %>%
  mutate(lifeExp=lifeExp-cAvg_lifeExp,
         gdpPerCapita=gdpPerCapita-cAvg_gdpPerCapita,
         pop=pop-cAvg_pop,
         pctFemale=pctFemale-cAvg_pctFemale,
         pctRural=pctRural-cAvg_pctRural
         )  %>%
  ungroup()

## Estimate within transformation using the transformed data
countryWithin <- lm(lifeExp~gdpPerCapita+pop+pctFemale+pctRural,data=mydataCountry)

## Using plm package
countryPlm <- plm(lifeExp~gdpPerCapita+pop+pctFemale+pctRural,data=mydata, index=c("country"), model = "within", effect="individual")
```




```r
stargazer(countryDummies,countryWithin,countryPlm, 
          type = "html", 
          report=('vcs*'),
          single.row = TRUE,
          digits = 4,
          keep.stat = c("n","rsq","adj.rsq"), 
          notes = "Standard Errors reported in parentheses, <em>&#42;p&lt;0.1;&#42;&#42;p&lt;0.05;&#42;&#42;&#42;p&lt;0.01</em>", 
          notes.append = FALSE)
```


<table style="text-align:center"><tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left"></td><td colspan="3"><em>Dependent variable:</em></td></tr>
<tr><td></td><td colspan="3" style="border-bottom: 1px solid black"></td></tr>
<tr><td style="text-align:left"></td><td colspan="3">lifeExp</td></tr>
<tr><td style="text-align:left"></td><td colspan="2"><em>OLS</em></td><td><em>panel</em></td></tr>
<tr><td style="text-align:left"></td><td colspan="2"><em></em></td><td><em>linear</em></td></tr>
<tr><td style="text-align:left"></td><td>(1)</td><td>(2)</td><td>(3)</td></tr>
<tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">gdpPerCapita</td><td>0.0651 (0.0533)</td><td>0.0651 (0.0461)</td><td>0.0651 (0.0533)</td></tr>
<tr><td style="text-align:left">pop</td><td>0.0746 (0.0292)<sup>**</sup></td><td>0.0746 (0.0253)<sup>***</sup></td><td>0.0746 (0.0292)<sup>**</sup></td></tr>
<tr><td style="text-align:left">pctFemale</td><td>-0.0821 (0.4343)</td><td>-0.0821 (0.3756)</td><td>-0.0821 (0.4343)</td></tr>
<tr><td style="text-align:left">pctRural</td><td>-0.3415 (0.0389)<sup>***</sup></td><td>-0.3415 (0.0336)<sup>***</sup></td><td>-0.3415 (0.0389)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Argentina</td><td>-26.4352 (2.5323)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Bahamas, The</td><td>-24.2300 (2.1318)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Barbados</td><td>-0.0618 (0.2654)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Belize</td><td>-8.9778 (1.5499)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Bolivia</td><td>-21.1922 (2.4176)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Brazil</td><td>-37.4007 (5.6170)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Chile</td><td>-19.5972 (2.5142)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Colombia</td><td>-21.9154 (2.4081)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Costa Rica</td><td>-15.3878 (2.4876)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Cuba</td><td>-16.3491 (2.4175)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Dominican Republic</td><td>-22.3355 (2.6281)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Ecuador</td><td>-14.2608 (2.0636)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)El Salvador</td><td>-19.2981 (1.9355)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Grenada</td><td>-7.9733 (1.3193)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Guatemala</td><td>-12.3493 (1.4744)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Guyana</td><td>-7.3506 (1.0755)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Haiti</td><td>-23.5398 (1.7323)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Honduras</td><td>-12.5973 (1.9355)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Jamaica</td><td>-12.5303 (1.7901)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Mexico</td><td>-29.4623 (3.5027)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Nicaragua</td><td>-13.8416 (1.8645)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Panama</td><td>-13.5153 (2.0809)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Paraguay</td><td>-15.3079 (2.2995)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Peru</td><td>-20.4454 (2.4008)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Puerto Rico</td><td>-21.7780 (2.3829)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)St. Lucia</td><td>1.4653 (0.5105)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)St. Vincent and the Grenadines</td><td>-13.2790 (2.0031)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Suriname</td><td>-19.1671 (2.2540)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Trinidad and Tobago</td><td>-13.5590 (1.3500)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Uruguay</td><td>-23.4638 (2.7021)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Virgin Islands (U.S.)</td><td>-22.7951 (2.4659)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">Constant</td><td>105.6350 (23.8548)<sup>***</sup></td><td>0.0000 (0.0114)</td><td></td></tr>
<tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Observations</td><td>128</td><td>128</td><td>128</td></tr>
<tr><td style="text-align:left">R<sup>2</sup></td><td>0.9987</td><td>0.6324</td><td>0.6324</td></tr>
<tr><td style="text-align:left">Adjusted R<sup>2</sup></td><td>0.9981</td><td>0.6205</td><td>0.4926</td></tr>
<tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left"><em>Note:</em></td><td colspan="3" style="text-align:right">Standard Errors reported in parentheses, <em>&#42;p&lt;0.1;&#42;&#42;p&lt;0.05;&#42;&#42;&#42;p&lt;0.01</em></td></tr>
</table>


We've changed a few of the stargazer options for this chapter. We're displaying standard errors instead of p-values so that we can use only one row per variable (it lets us report coefficient and standard error on one line, but not if we use p-values instead). I modified the note to say that standard errors are reported in parentheses. 

Now look at the coefficient estimates. The 3 models all have the same coefficients on `gdpPerCapita`, `pop`, `pctFemale`, and `pctRurla`. In LN5 we discuss how the within transformation is equivalent to including dummy variables for each group (in this case, countries). That's exactly what see in the table. The PLM package estimates the within transformation for us. 

You also may notice that the standard errors (and statistical significance stars) are different in the middle column. When we estimate the model with dummy variables (column 1), the regular OLS standard errors are correct. But when we apply the within transformation, we need to adjust the standard errors to account for the within transformation. This would be a bit difficult for us to do. Thankfully, the PLM package correctly adjusts the standard errors (and thus p-values) for us.  Thus, in practice we won't actually want to apply the within transformation ourselves. We're doing it in this chapter so you can see exactly what it is in practice and see that the coefficient estimates for all 3 versions result in the same coefficients. 

If you compare the $R^2$ values across models, you'll notice that the $R^2$ for the model with dummy variables is much higher. Including all the dummy variables makes it artificially high. We want to use the $R^2$ from the within transformation. The PLM model does this for us. 

Another reason we want to use the PLM model when we estimate fixed effects models is that we often don't want to see all of the coefficients on the dummy variables. For country fixed effects, the coefficient on each country dummy is estimated off of only 4 observations. Thus, it is not a reliable estimate of the effect of being Argentina (or Belize, etc). It still allows us to estimate the model with country fixed effects, even if we don't care about the coefficient estimates themselves.  However, if we had not dropped all countries except South America, we would have hundreds of dummy variables. If we were estimating a model using US county data, we would have over 3000. R probably wouldn't even let us estimate a model with that many variables. This again makes the PLM package preferable. 


## Year Fixed Effects

Above you saw how to estimate models with country fixed effects in three different ways. Here, you should estimate models with year fixed effects in the same three ways. Hint: you just have to switch "country" with "year" and everythign else is the same. 


```r
## Dummy variable for each year (it automatically omits one)
yearDummies <- lm(lifeExp~gdpPerCapita+pop+pctFemale+pctRural+factor(year),data=mydata)

## Within transformation (subtract year averages from each observation for each variable)

## Create Year Averages
mydata <- mydata %>%
  group_by(year) %>%
  mutate(yrAvg_lifeExp=mean(lifeExp),
         yrAvg_gdpPerCapita=mean(gdpPerCapita),
         yrAvg_pop=mean(pop),
         yrAvg_pctFemale=mean(pctFemale),
         yrAvg_pctRural=mean(pctRural)
         ) %>%
  ungroup()

## Within transformation
mydataYear <- mydata %>%
  mutate(lifeExp=lifeExp-yrAvg_lifeExp,
         gdpPerCapita=gdpPerCapita-yrAvg_gdpPerCapita,
         pop=pop-yrAvg_pop,
         pctFemale=pctFemale-yrAvg_pctFemale,
         pctRural=pctRural-yrAvg_pctRural
         )  %>%
  ungroup()



## Estimate within transformation using the transformed data
yearWithin <- lm(lifeExp~gdpPerCapita+pop+pctFemale+pctRural,data=mydataYear)

## Using plm package
yearPlm <- plm(lifeExp~gdpPerCapita+pop+pctFemale+pctRural,data=mydata, index=c("year"), model = "within", effect="individual")
```



```r
stargazer(yearDummies,yearWithin,yearPlm, 
          type = "html", 
          report=('vcs*'),
          single.row = TRUE,
          digits = 4,
          keep.stat = c("n","rsq","adj.rsq"), 
          notes = "Standard Errors reported in parentheses, <em>&#42;p&lt;0.1;&#42;&#42;p&lt;0.05;&#42;&#42;&#42;p&lt;0.01</em>", 
          notes.append = FALSE)
```


<table style="text-align:center"><tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left"></td><td colspan="3"><em>Dependent variable:</em></td></tr>
<tr><td></td><td colspan="3" style="border-bottom: 1px solid black"></td></tr>
<tr><td style="text-align:left"></td><td colspan="3">lifeExp</td></tr>
<tr><td style="text-align:left"></td><td colspan="2"><em>OLS</em></td><td><em>panel</em></td></tr>
<tr><td style="text-align:left"></td><td colspan="2"><em></em></td><td><em>linear</em></td></tr>
<tr><td style="text-align:left"></td><td>(1)</td><td>(2)</td><td>(3)</td></tr>
<tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">gdpPerCapita</td><td>0.1628 (0.0416)<sup>***</sup></td><td>0.1628 (0.0411)<sup>***</sup></td><td>0.1628 (0.0416)<sup>***</sup></td></tr>
<tr><td style="text-align:left">pop</td><td>0.0026 (0.0072)</td><td>0.0026 (0.0071)</td><td>0.0026 (0.0072)</td></tr>
<tr><td style="text-align:left">pctFemale</td><td>0.2108 (0.3435)</td><td>0.2108 (0.3393)</td><td>0.2108 (0.3435)</td></tr>
<tr><td style="text-align:left">pctRural</td><td>-0.0325 (0.0147)<sup>**</sup></td><td>-0.0325 (0.0146)<sup>**</sup></td><td>-0.0325 (0.0147)<sup>**</sup></td></tr>
<tr><td style="text-align:left">factor(year)2016</td><td>0.1532 (0.7523)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(year)2017</td><td>0.2970 (0.7524)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(year)2018</td><td>0.4251 (0.7524)</td><td></td><td></td></tr>
<tr><td style="text-align:left">Constant</td><td>63.3957 (17.2241)<sup>***</sup></td><td>0.0000 (0.2627)</td><td></td></tr>
<tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Observations</td><td>128</td><td>128</td><td>128</td></tr>
<tr><td style="text-align:left">R<sup>2</sup></td><td>0.2831</td><td>0.2809</td><td>0.2809</td></tr>
<tr><td style="text-align:left">Adjusted R<sup>2</sup></td><td>0.2413</td><td>0.2576</td><td>0.2390</td></tr>
<tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left"><em>Note:</em></td><td colspan="3" style="text-align:right">Standard Errors reported in parentheses, <em>&#42;p&lt;0.1;&#42;&#42;p&lt;0.05;&#42;&#42;&#42;p&lt;0.01</em></td></tr>
</table>




## Country and Year Fixed Effects

Now that you've estimated the models with year fixed effects, estimate models with both country and year fixed effects. It works the same way as above, just doing it for both country and year.


```r
## Dummy variable for each country and each year (it automatically omits one of each)
countryyearDummies <- lm(lifeExp~gdpPerCapita+pop+pctFemale+pctRural+factor(year)+factor(country),data=mydata)

## Within transformation (subtract country AND year averages from each observation for each variable)
## We already created the country averages and year averages above so we don't need to create them again

## Within transformation
mydataCountryYear <- mydata %>%
  mutate(lifeExp=lifeExp-cAvg_lifeExp-yrAvg_lifeExp,
         gdpPerCapita=gdpPerCapita-cAvg_gdpPerCapita-yrAvg_gdpPerCapita,
         pop=pop-cAvg_pop-yrAvg_pop,
         pctFemale=pctFemale-cAvg_pctFemale-yrAvg_pctFemale,
         pctRural=pctRural-cAvg_pctRural-yrAvg_pctRural
         )  %>%
  ungroup()

##Estimate within transformation using the transformed data
countryYearWithin <- lm(lifeExp~gdpPerCapita+pop+pctFemale+pctRural,data=mydataCountryYear)

##Using plm package
countryYearPlm <- plm(lifeExp~gdpPerCapita+pop+pctFemale+pctRural,data=mydata, index=c("country","year"), model = "within", effect="twoways")
```


```r
stargazer(countryyearDummies,countryYearWithin, countryYearPlm, 
          type = "html", 
          report=('vcs*'),
          single.row = TRUE,
          digits = 4,
          keep.stat = c("n","rsq","adj.rsq"), 
          notes = "Standard Errors reported in parentheses, <em>&#42;p&lt;0.1;&#42;&#42;p&lt;0.05;&#42;&#42;&#42;p&lt;0.01</em>", 
          notes.append = FALSE)
```


<table style="text-align:center"><tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left"></td><td colspan="3"><em>Dependent variable:</em></td></tr>
<tr><td></td><td colspan="3" style="border-bottom: 1px solid black"></td></tr>
<tr><td style="text-align:left"></td><td colspan="3">lifeExp</td></tr>
<tr><td style="text-align:left"></td><td colspan="2"><em>OLS</em></td><td><em>panel</em></td></tr>
<tr><td style="text-align:left"></td><td colspan="2"><em></em></td><td><em>linear</em></td></tr>
<tr><td style="text-align:left"></td><td>(1)</td><td>(2)</td><td>(3)</td></tr>
<tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">gdpPerCapita</td><td>-0.0770 (0.0332)<sup>**</sup></td><td>-0.0770 (0.0282)<sup>***</sup></td><td>-0.0770 (0.0332)<sup>**</sup></td></tr>
<tr><td style="text-align:left">pop</td><td>-0.0122 (0.0184)</td><td>-0.0122 (0.0156)</td><td>-0.0122 (0.0184)</td></tr>
<tr><td style="text-align:left">pctFemale</td><td>-0.4519 (0.2566)<sup>*</sup></td><td>-0.4519 (0.2183)<sup>**</sup></td><td>-0.4519 (0.2566)<sup>*</sup></td></tr>
<tr><td style="text-align:left">pctRural</td><td>-0.1670 (0.0263)<sup>***</sup></td><td>-0.1670 (0.0224)<sup>***</sup></td><td>-0.1670 (0.0263)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(year)2016</td><td>0.1409 (0.0231)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(year)2017</td><td>0.2808 (0.0265)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(year)2018</td><td>0.4187 (0.0322)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Argentina</td><td>-11.4184 (1.8649)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Bahamas, The</td><td>-11.8448 (1.5588)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Barbados</td><td>1.2703 (0.1851)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Belize</td><td>-7.4386 (0.9190)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Bolivia</td><td>-15.0054 (1.4948)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Brazil</td><td>-10.0431 (3.8838)<sup>**</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Chile</td><td>-7.6845 (1.7254)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Colombia</td><td>-9.6159 (1.6882)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Costa Rica</td><td>-6.7688 (1.5976)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Cuba</td><td>-7.8679 (1.5560)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Dominican Republic</td><td>-13.6116 (1.6768)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Ecuador</td><td>-8.0428 (1.2992)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)El Salvador</td><td>-11.8715 (1.2660)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Grenada</td><td>-7.6154 (0.7762)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Guatemala</td><td>-8.4458 (0.9150)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Guyana</td><td>-9.0234 (0.6456)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Haiti</td><td>-19.9197 (1.0544)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Honduras</td><td>-8.7937 (1.1733)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Jamaica</td><td>-8.9748 (1.0857)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Mexico</td><td>-10.1515 (2.5150)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Nicaragua</td><td>-9.7345 (1.1388)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Panama</td><td>-6.6135 (1.3272)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Paraguay</td><td>-10.7231 (1.3946)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Peru</td><td>-10.3166 (1.6017)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Puerto Rico</td><td>-7.2295 (1.7743)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)St. Lucia</td><td>-0.6604 (0.3402)<sup>*</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)St. Vincent and the Grenadines</td><td>-10.7481 (1.1927)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Suriname</td><td>-13.6043 (1.3885)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Trinidad and Tobago</td><td>-8.6764 (0.8731)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Uruguay</td><td>-10.8173 (1.8483)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Virgin Islands (U.S.)</td><td>-7.4864 (1.8481)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">Constant</td><td>113.6250 (14.0285)<sup>***</sup></td><td>-104.7890 (11.3948)<sup>***</sup></td><td></td></tr>
<tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Observations</td><td>128</td><td>128</td><td>128</td></tr>
<tr><td style="text-align:left">R<sup>2</sup></td><td>0.9996</td><td>0.3214</td><td>0.3214</td></tr>
<tr><td style="text-align:left">Adjusted R<sup>2</sup></td><td>0.9994</td><td>0.2994</td><td>0.0317</td></tr>
<tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left"><em>Note:</em></td><td colspan="3" style="text-align:right">Standard Errors reported in parentheses, <em>&#42;p&lt;0.1;&#42;&#42;p&lt;0.05;&#42;&#42;&#42;p&lt;0.01</em></td></tr>
</table>





## Comparison of all models

Below we have comparisons of the four models: ols, country fixed effects, year fixed effects, and country and year fixed effects. The comparisons are done three times, one for each method of estimating the models. 

### Within Transformation



```r
stargazer(ols,countryWithin,yearWithin,countryYearWithin, 
          type = "html", 
          report=('vcs*'),
          single.row = TRUE,
          digits = 3,
          keep.stat = c("n","rsq","adj.rsq"), 
          notes = "Standard Errors reported in parentheses, <em>&#42;p&lt;0.1;&#42;&#42;p&lt;0.05;&#42;&#42;&#42;p&lt;0.01</em>", 
          notes.append = FALSE)
```


<table style="text-align:center"><tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left"></td><td colspan="4"><em>Dependent variable:</em></td></tr>
<tr><td></td><td colspan="4" style="border-bottom: 1px solid black"></td></tr>
<tr><td style="text-align:left"></td><td colspan="4">lifeExp</td></tr>
<tr><td style="text-align:left"></td><td>(1)</td><td>(2)</td><td>(3)</td><td>(4)</td></tr>
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">gdpPerCapita</td><td>0.163 (0.041)<sup>***</sup></td><td>0.065 (0.046)</td><td>0.163 (0.041)<sup>***</sup></td><td>-0.077 (0.028)<sup>***</sup></td></tr>
<tr><td style="text-align:left">pop</td><td>0.003 (0.007)</td><td>0.075 (0.025)<sup>***</sup></td><td>0.003 (0.007)</td><td>-0.012 (0.016)</td></tr>
<tr><td style="text-align:left">pctFemale</td><td>0.210 (0.340)</td><td>-0.082 (0.376)</td><td>0.211 (0.339)</td><td>-0.452 (0.218)<sup>**</sup></td></tr>
<tr><td style="text-align:left">pctRural</td><td>-0.033 (0.015)<sup>**</sup></td><td>-0.342 (0.034)<sup>***</sup></td><td>-0.033 (0.015)<sup>**</sup></td><td>-0.167 (0.022)<sup>***</sup></td></tr>
<tr><td style="text-align:left">Constant</td><td>63.633 (17.031)<sup>***</sup></td><td>0.000 (0.011)</td><td>0.000 (0.263)</td><td>-104.789 (11.395)<sup>***</sup></td></tr>
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Observations</td><td>128</td><td>128</td><td>128</td><td>128</td></tr>
<tr><td style="text-align:left">R<sup>2</sup></td><td>0.281</td><td>0.632</td><td>0.281</td><td>0.321</td></tr>
<tr><td style="text-align:left">Adjusted R<sup>2</sup></td><td>0.258</td><td>0.620</td><td>0.258</td><td>0.299</td></tr>
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left"><em>Note:</em></td><td colspan="4" style="text-align:right">Standard Errors reported in parentheses, <em>&#42;p&lt;0.1;&#42;&#42;p&lt;0.05;&#42;&#42;&#42;p&lt;0.01</em></td></tr>
</table>

### PLM Package


```r
stargazer(ols,countryPlm,yearPlm,countryYearPlm, 
          type = "html", 
          report=('vcs*'),
          single.row = TRUE,
          digits = 3,
          keep.stat = c("n","rsq","adj.rsq"), 
          notes = "Standard Errors reported in parentheses, <em>&#42;p&lt;0.1;&#42;&#42;p&lt;0.05;&#42;&#42;&#42;p&lt;0.01</em>", 
          notes.append = FALSE)
```


<table style="text-align:center"><tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left"></td><td colspan="4"><em>Dependent variable:</em></td></tr>
<tr><td></td><td colspan="4" style="border-bottom: 1px solid black"></td></tr>
<tr><td style="text-align:left"></td><td colspan="4">lifeExp</td></tr>
<tr><td style="text-align:left"></td><td><em>OLS</em></td><td colspan="3"><em>panel</em></td></tr>
<tr><td style="text-align:left"></td><td><em></em></td><td colspan="3"><em>linear</em></td></tr>
<tr><td style="text-align:left"></td><td>(1)</td><td>(2)</td><td>(3)</td><td>(4)</td></tr>
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">gdpPerCapita</td><td>0.163 (0.041)<sup>***</sup></td><td>0.065 (0.053)</td><td>0.163 (0.042)<sup>***</sup></td><td>-0.077 (0.033)<sup>**</sup></td></tr>
<tr><td style="text-align:left">pop</td><td>0.003 (0.007)</td><td>0.075 (0.029)<sup>**</sup></td><td>0.003 (0.007)</td><td>-0.012 (0.018)</td></tr>
<tr><td style="text-align:left">pctFemale</td><td>0.210 (0.340)</td><td>-0.082 (0.434)</td><td>0.211 (0.344)</td><td>-0.452 (0.257)<sup>*</sup></td></tr>
<tr><td style="text-align:left">pctRural</td><td>-0.033 (0.015)<sup>**</sup></td><td>-0.342 (0.039)<sup>***</sup></td><td>-0.033 (0.015)<sup>**</sup></td><td>-0.167 (0.026)<sup>***</sup></td></tr>
<tr><td style="text-align:left">Constant</td><td>63.633 (17.031)<sup>***</sup></td><td></td><td></td><td></td></tr>
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Observations</td><td>128</td><td>128</td><td>128</td><td>128</td></tr>
<tr><td style="text-align:left">R<sup>2</sup></td><td>0.281</td><td>0.632</td><td>0.281</td><td>0.321</td></tr>
<tr><td style="text-align:left">Adjusted R<sup>2</sup></td><td>0.258</td><td>0.493</td><td>0.239</td><td>0.032</td></tr>
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left"><em>Note:</em></td><td colspan="4" style="text-align:right">Standard Errors reported in parentheses, <em>&#42;p&lt;0.1;&#42;&#42;p&lt;0.05;&#42;&#42;&#42;p&lt;0.01</em></td></tr>
</table>

### Dummy Variables



```r
stargazer(ols,countryDummies,yearDummies,countryyearDummies, 
          type = "html", 
          report=('vcs*'),
          single.row = TRUE,
          digits = 3,
          keep.stat = c("n","rsq","adj.rsq"), 
          notes = "Standard Errors reported in parentheses, <em>&#42;p&lt;0.1;&#42;&#42;p&lt;0.05;&#42;&#42;&#42;p&lt;0.01</em>", 
          notes.append = FALSE)
```


<table style="text-align:center"><tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left"></td><td colspan="4"><em>Dependent variable:</em></td></tr>
<tr><td></td><td colspan="4" style="border-bottom: 1px solid black"></td></tr>
<tr><td style="text-align:left"></td><td colspan="4">lifeExp</td></tr>
<tr><td style="text-align:left"></td><td>(1)</td><td>(2)</td><td>(3)</td><td>(4)</td></tr>
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">gdpPerCapita</td><td>0.163 (0.041)<sup>***</sup></td><td>0.065 (0.053)</td><td>0.163 (0.042)<sup>***</sup></td><td>-0.077 (0.033)<sup>**</sup></td></tr>
<tr><td style="text-align:left">pop</td><td>0.003 (0.007)</td><td>0.075 (0.029)<sup>**</sup></td><td>0.003 (0.007)</td><td>-0.012 (0.018)</td></tr>
<tr><td style="text-align:left">pctFemale</td><td>0.210 (0.340)</td><td>-0.082 (0.434)</td><td>0.211 (0.344)</td><td>-0.452 (0.257)<sup>*</sup></td></tr>
<tr><td style="text-align:left">pctRural</td><td>-0.033 (0.015)<sup>**</sup></td><td>-0.342 (0.039)<sup>***</sup></td><td>-0.033 (0.015)<sup>**</sup></td><td>-0.167 (0.026)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Argentina</td><td></td><td>-26.435 (2.532)<sup>***</sup></td><td></td><td>-11.418 (1.865)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Bahamas, The</td><td></td><td>-24.230 (2.132)<sup>***</sup></td><td></td><td>-11.845 (1.559)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Barbados</td><td></td><td>-0.062 (0.265)</td><td></td><td>1.270 (0.185)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Belize</td><td></td><td>-8.978 (1.550)<sup>***</sup></td><td></td><td>-7.439 (0.919)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Bolivia</td><td></td><td>-21.192 (2.418)<sup>***</sup></td><td></td><td>-15.005 (1.495)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Brazil</td><td></td><td>-37.401 (5.617)<sup>***</sup></td><td></td><td>-10.043 (3.884)<sup>**</sup></td></tr>
<tr><td style="text-align:left">factor(country)Chile</td><td></td><td>-19.597 (2.514)<sup>***</sup></td><td></td><td>-7.684 (1.725)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Colombia</td><td></td><td>-21.915 (2.408)<sup>***</sup></td><td></td><td>-9.616 (1.688)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Costa Rica</td><td></td><td>-15.388 (2.488)<sup>***</sup></td><td></td><td>-6.769 (1.598)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Cuba</td><td></td><td>-16.349 (2.417)<sup>***</sup></td><td></td><td>-7.868 (1.556)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Dominican Republic</td><td></td><td>-22.336 (2.628)<sup>***</sup></td><td></td><td>-13.612 (1.677)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Ecuador</td><td></td><td>-14.261 (2.064)<sup>***</sup></td><td></td><td>-8.043 (1.299)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)El Salvador</td><td></td><td>-19.298 (1.936)<sup>***</sup></td><td></td><td>-11.871 (1.266)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Grenada</td><td></td><td>-7.973 (1.319)<sup>***</sup></td><td></td><td>-7.615 (0.776)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Guatemala</td><td></td><td>-12.349 (1.474)<sup>***</sup></td><td></td><td>-8.446 (0.915)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Guyana</td><td></td><td>-7.351 (1.076)<sup>***</sup></td><td></td><td>-9.023 (0.646)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Haiti</td><td></td><td>-23.540 (1.732)<sup>***</sup></td><td></td><td>-19.920 (1.054)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Honduras</td><td></td><td>-12.597 (1.936)<sup>***</sup></td><td></td><td>-8.794 (1.173)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Jamaica</td><td></td><td>-12.530 (1.790)<sup>***</sup></td><td></td><td>-8.975 (1.086)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Mexico</td><td></td><td>-29.462 (3.503)<sup>***</sup></td><td></td><td>-10.152 (2.515)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Nicaragua</td><td></td><td>-13.842 (1.864)<sup>***</sup></td><td></td><td>-9.735 (1.139)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Panama</td><td></td><td>-13.515 (2.081)<sup>***</sup></td><td></td><td>-6.613 (1.327)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Paraguay</td><td></td><td>-15.308 (2.300)<sup>***</sup></td><td></td><td>-10.723 (1.395)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Peru</td><td></td><td>-20.445 (2.401)<sup>***</sup></td><td></td><td>-10.317 (1.602)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Puerto Rico</td><td></td><td>-21.778 (2.383)<sup>***</sup></td><td></td><td>-7.229 (1.774)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)St. Lucia</td><td></td><td>1.465 (0.511)<sup>***</sup></td><td></td><td>-0.660 (0.340)<sup>*</sup></td></tr>
<tr><td style="text-align:left">factor(country)St. Vincent and the Grenadines</td><td></td><td>-13.279 (2.003)<sup>***</sup></td><td></td><td>-10.748 (1.193)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Suriname</td><td></td><td>-19.167 (2.254)<sup>***</sup></td><td></td><td>-13.604 (1.389)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Trinidad and Tobago</td><td></td><td>-13.559 (1.350)<sup>***</sup></td><td></td><td>-8.676 (0.873)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Uruguay</td><td></td><td>-23.464 (2.702)<sup>***</sup></td><td></td><td>-10.817 (1.848)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Virgin Islands (U.S.)</td><td></td><td>-22.795 (2.466)<sup>***</sup></td><td></td><td>-7.486 (1.848)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(year)2016</td><td></td><td></td><td>0.153 (0.752)</td><td>0.141 (0.023)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(year)2017</td><td></td><td></td><td>0.297 (0.752)</td><td>0.281 (0.027)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(year)2018</td><td></td><td></td><td>0.425 (0.752)</td><td>0.419 (0.032)<sup>***</sup></td></tr>
<tr><td style="text-align:left">Constant</td><td>63.633 (17.031)<sup>***</sup></td><td>105.635 (23.855)<sup>***</sup></td><td>63.396 (17.224)<sup>***</sup></td><td>113.625 (14.028)<sup>***</sup></td></tr>
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Observations</td><td>128</td><td>128</td><td>128</td><td>128</td></tr>
<tr><td style="text-align:left">R<sup>2</sup></td><td>0.281</td><td>0.999</td><td>0.283</td><td>1.000</td></tr>
<tr><td style="text-align:left">Adjusted R<sup>2</sup></td><td>0.258</td><td>0.998</td><td>0.241</td><td>0.999</td></tr>
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left"><em>Note:</em></td><td colspan="4" style="text-align:right">Standard Errors reported in parentheses, <em>&#42;p&lt;0.1;&#42;&#42;p&lt;0.05;&#42;&#42;&#42;p&lt;0.01</em></td></tr>
</table>




## Data Summary by Country


```r
stargazer(as.data.frame(mydata) , type = "html",digits = 2 ,summary.stat = c("n","mean","sd", "min", "median", "max"))
```


<table style="text-align:center"><tr><td colspan="7" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Statistic</td><td>N</td><td>Mean</td><td>St. Dev.</td><td>Min</td><td>Median</td><td>Max</td></tr>
<tr><td colspan="7" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">year</td><td>128</td><td>2,016.50</td><td>1.12</td><td>2,015</td><td>2,016.5</td><td>2,018</td></tr>
<tr><td style="text-align:left">gdpPerCapita</td><td>128</td><td>10.66</td><td>8.20</td><td>1.39</td><td>8.25</td><td>35.07</td></tr>
<tr><td style="text-align:left">lifeExp</td><td>128</td><td>74.92</td><td>3.45</td><td>62.48</td><td>74.91</td><td>80.10</td></tr>
<tr><td style="text-align:left">pop</td><td>128</td><td>18.79</td><td>41.17</td><td>0.09</td><td>5.61</td><td>209.47</td></tr>
<tr><td style="text-align:left">pctFemale</td><td>128</td><td>50.66</td><td>0.93</td><td>49.07</td><td>50.61</td><td>53.11</td></tr>
<tr><td style="text-align:left">pctRural</td><td>128</td><td>35.49</td><td>21.25</td><td>4.28</td><td>33.62</td><td>81.48</td></tr>
<tr><td style="text-align:left">cAvg_lifeExp</td><td>128</td><td>74.92</td><td>3.45</td><td>63.08</td><td>74.87</td><td>79.84</td></tr>
<tr><td style="text-align:left">cAvg_gdpPerCapita</td><td>128</td><td>10.66</td><td>8.20</td><td>1.40</td><td>8.21</td><td>34.53</td></tr>
<tr><td style="text-align:left">cAvg_pop</td><td>128</td><td>18.79</td><td>41.17</td><td>0.09</td><td>5.63</td><td>206.98</td></tr>
<tr><td style="text-align:left">cAvg_pctFemale</td><td>128</td><td>50.66</td><td>0.93</td><td>49.13</td><td>50.61</td><td>53.05</td></tr>
<tr><td style="text-align:left">cAvg_pctRural</td><td>128</td><td>35.49</td><td>21.25</td><td>4.46</td><td>33.38</td><td>81.41</td></tr>
<tr><td style="text-align:left">yrAvg_lifeExp</td><td>128</td><td>74.92</td><td>0.19</td><td>74.67</td><td>74.92</td><td>75.17</td></tr>
<tr><td style="text-align:left">yrAvg_gdpPerCapita</td><td>128</td><td>10.66</td><td>0.12</td><td>10.53</td><td>10.64</td><td>10.84</td></tr>
<tr><td style="text-align:left">yrAvg_pop</td><td>128</td><td>18.79</td><td>0.23</td><td>18.49</td><td>18.79</td><td>19.09</td></tr>
<tr><td style="text-align:left">yrAvg_pctFemale</td><td>128</td><td>50.66</td><td>0.01</td><td>50.65</td><td>50.66</td><td>50.67</td></tr>
<tr><td style="text-align:left">yrAvg_pctRural</td><td>128</td><td>35.49</td><td>0.28</td><td>35.11</td><td>35.49</td><td>35.87</td></tr>
<tr><td colspan="7" style="border-bottom: 1px solid black"></td></tr></table>


### Average Values for Each Country


<table class="table table-condensed">
 <thead>
  <tr>
   <th style="text-align:right;"> country </th>
   <th style="text-align:right;"> Avg<br>lifeExp </th>
   <th style="text-align:right;"> Avg<br>gdpPerCapita </th>
   <th style="text-align:right;"> Avg<br>pop </th>
   <th style="text-align:right;"> Avg<br>pctFemale </th>
   <th style="text-align:right;"> Avg<br>pctRural </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> Antigua and Barbuda </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a4f1a4">76.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dceef4">15.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffffff">0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6e4ed">51.83</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #98ef98">75.21</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Argentina </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a7f1a7">76.30</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e1f0f5">13.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e7fbe7">43.82</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2e9f1">51.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f9fef9">8.31</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Bahamas, The </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #baf4ba">73.43</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3dbe8">31.80</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.38</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cee7f0">51.45</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ecfcec">17.12</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Barbados </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #95ee95">78.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8ecf3">16.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8e5ee">51.73</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a2f0a2">68.81</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Belize </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4f3b4">74.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f6fbfc">4.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e9f4f8">50.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6f3b6">54.44</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Bolivia </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ccf7cc">70.77</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fafcfd">3.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f9fef9">11.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f1f8fa">49.76</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8f9d8">31.09</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Brazil </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #adf2ad">75.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #edf6f9">8.59</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">206.98</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dceef4">50.80</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f1fcf1">13.83</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Chile </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">79.84</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e0f0f5">13.67</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f5fdf5">18.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddeef4">50.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f3fdf3">12.54</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Colombia </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a4f1a4">76.82</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f3f9fb">6.22</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e4fbe4">48.57</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8ecf3">50.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8fbe8">19.73</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Costa Rica </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">79.83</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e4f2f6">12.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fcfefc">4.92</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #edf6f9">49.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e5fbe5">21.88</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Cuba </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #97ef97">78.64</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #eff7fa">7.83</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f8fef8">11.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e6f3f7">50.32</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e4fae4">23.04</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Dominican Republic </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9f4b9">73.57</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f0f7fa">7.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f9fef9">10.45</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #edf6f9">49.97</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8fbe8">20.16</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Ecuador </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a6f1a6">76.47</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f3f9fb">6.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f6fdf6">16.64</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #edf6f9">49.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d0f7d0">36.39</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> El Salvador </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bef5be">72.76</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f9fcfd">3.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fbfefb">6.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #add8e6">53.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbf9db">29.13</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Grenada </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1f5c1">72.41</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #eaf5f8">9.58</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f5fafc">49.58</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a9f1a9">63.87</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Guatemala </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8f4b8">73.67</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f8fbfc">4.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f6fdf6">15.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dceef4">50.78</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bef5be">49.49</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Guyana </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4f8d4">69.53</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f3f9fb">5.87</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.77</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #eef7fa">49.90</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9bef9b">73.48</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Haiti </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffffff">63.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffffff">1.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f9fef9">10.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfeff5">50.64</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c2f5c2">46.14</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Honduras </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1f3b1">74.80</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fcfdfe">2.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fafefa">9.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ebf5f9">50.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6f6c6">43.87</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Jamaica </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5f3b5">74.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f6fafc">4.97</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fdfefd">2.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e6f3f7">50.32</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4f6c4">44.75</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Mexico </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b0f2b0">74.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #eaf5f8">9.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcf4bc">124.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5ebf2">51.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8fbe8">20.28</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Nicaragua </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6f3b6">73.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fdfefe">2.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fbfefb">6.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddeff4">50.71</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9f6c9">41.80</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Panama </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9bef9b">78.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfeff5">14.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fcfefc">4.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #eff7fa">49.88</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6f8d6">32.80</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Paraguay </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7f4b7">73.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f4f9fb">5.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fbfefb">6.82</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffffff">49.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdf7cd">38.83</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Peru </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a8f1a8">76.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f2f9fb">6.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #eefcee">31.21</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e5f2f7">50.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e5fbe5">22.37</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Puerto Rico </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #91ee91">79.57</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8dde9">29.82</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fdfefd">3.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcdfea">52.30</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fcfefc">6.40</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> St. Lucia </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aaf2aa">75.83</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8f4f8">10.55</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddeef4">50.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">81.41</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> St. Vincent and the Grenadines </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c2f5c2">72.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f0f8fa">7.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">49.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bff5bf">48.42</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Suriname </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7f6c7">71.41</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ecf6f9">8.87</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.57</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f3f9fb">49.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4f8d4">33.95</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Trinidad and Tobago </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcf4bc">73.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8ecf3">17.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">1.38</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e0f0f5">50.57</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1f5c1">46.76</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Uruguay </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9ff09f">77.57</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbedf4">15.87</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fdfefd">3.43</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8e4ee">51.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">4.81</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Virgin Islands (U.S.) </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #93ee93">79.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #add8e6">34.53</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #badeea">52.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffffff">4.46</span> </td>
  </tr>
</tbody>
</table>


### Variable-Specific Values and Within Transformation for Each Country
For each variable, display each country's values in 2015, 2016, 2017, and 2018, followed by the country's average. These 5 columns are shaded from red (lowest) to green (highest) for each country. Then, in the final 4 columns display the within transformation (i.e., subtract the country's average from each year's value). These last 4 columns are also shaded for each country.












### Life expectancy

<table class="table table-condensed">
 <thead>
  <tr>
   <th style="text-align:right;"> iso3c </th>
   <th style="text-align:right;"> 2015<br>Life Exp </th>
   <th style="text-align:right;"> 2016<br>Life Exp </th>
   <th style="text-align:right;"> 2017<br>Life Exp </th>
   <th style="text-align:right;"> 2018<br>Life Exp </th>
   <th style="text-align:right;"> Avg<br>Life Exp </th>
   <th style="text-align:right;">   </th>
   <th style="text-align:right;"> 2015<br>Within </th>
   <th style="text-align:right;"> 2016<br>Within </th>
   <th style="text-align:right;"> 2017<br>Within </th>
   <th style="text-align:right;"> 2018<br>Within </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> ARG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">76.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">76.22</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">76.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">76.52</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a7">76.30</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.22</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ATG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">76.48</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c9b0">76.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5daa0">76.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">76.89</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a9">76.68</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.20</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4db9f">0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.20</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BHS </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">73.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caaf">73.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">73.55</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">73.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d2a7">73.43</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caaf">-0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.32</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BLZ </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">74.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">74.22</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">74.36</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">74.50</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c3d3a6">74.28</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d3cbad">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #afde9d">0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.22</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BOL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">70.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caaf">70.63</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">70.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">71.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a7">70.77</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.49</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">-0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.47</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">74.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">75.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">75.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">75.67</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d2a7">75.34</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">-0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.33</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">78.80</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc8b1">78.89</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">78.98</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">79.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">78.94</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc8b1">-0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.14</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CHL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">79.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">79.78</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">79.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">80.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a9">79.84</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.20</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9af">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4db9f">0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.20</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> COL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">76.53</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">76.73</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">76.92</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">77.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">76.82</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9af">-0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3dc9f">0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.28</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">79.56</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">79.74</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">79.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">80.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">79.83</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc7b1">-0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5daa0">0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.27</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CUB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">78.56</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dec6b2">78.61</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdd6a4">78.66</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">78.73</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cad0a9">78.64</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dec6b2">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdd6a4">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.09</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> DOM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">73.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">73.47</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">73.69</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">73.89</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">73.57</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">-0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.32</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ECU </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">76.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">76.36</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">76.58</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">76.80</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">76.47</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.33</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GRD </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">72.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">72.41</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ecbfb8">72.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">72.38</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">72.41</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GTM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">73.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7caaf">73.54</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">73.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">74.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d3a7">73.67</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.42</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caaf">-0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3dc9f">0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.40</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GUY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">69.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5caae">69.45</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b0dd9e">69.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">69.77</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">69.53</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4cbae">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b0dd9e">0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.25</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HND </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">74.50</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">74.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3db9f">74.90</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">75.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">74.80</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.30</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">-0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3db9f">0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.29</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HTI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">62.48</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">62.90</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">63.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">63.66</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">63.08</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.60</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9af">-0.19</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">0.21</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.58</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> JAM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">74.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e2c4b4">74.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d9a2">74.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">74.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d0a9">74.23</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dec6b2">-0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d9a2">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.14</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> LCA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">75.60</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">75.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4db9f">75.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">76.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">75.83</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4db9f">0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.23</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> MEX </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">74.90</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e6c2b6">74.92</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d5a5">74.95</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">74.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdceab">74.94</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e6c2b6">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d5a5">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.05</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> NIC </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">73.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">73.86</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">74.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">74.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">73.96</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.31</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">-0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3db9f">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.31</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PAN </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">77.78</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">77.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">78.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">78.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">78.05</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">-0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.27</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PER </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">75.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">76.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">76.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">76.52</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">76.16</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">-0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.36</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">79.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">79.49</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa1">79.63</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">79.78</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a7">79.57</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.21</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.21</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">73.66</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4cbae">73.84</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">73.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">74.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c3d3a6">73.91</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caaf">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.23</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SLV </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">72.41</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">72.64</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">72.87</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">73.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">72.76</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">-0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3db9f">0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.34</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SUR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">71.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">71.36</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">71.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">71.57</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">71.41</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">-0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.16</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> TTO </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">72.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caaf">73.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b0dd9e">73.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">73.38</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">73.17</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caaf">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b0dd9e">0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.21</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> URY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">77.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">77.50</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa1">77.63</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">77.77</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">77.57</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.20</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa1">0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.20</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VCT </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">72.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc5b3">72.19</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d8a2">72.30</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">72.42</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cad0aa">72.25</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dcc7b1">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.16</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VIR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">79.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc6b2">79.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">79.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">79.52</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">79.27</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc6b2">-0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.25</span> </td>
  </tr>
</tbody>
</table>

### GDP per capita


<table class="table table-condensed">
 <thead>
  <tr>
   <th style="text-align:right;"> iso3c </th>
   <th style="text-align:right;"> 2015<br>GDP per Capita </th>
   <th style="text-align:right;"> 2016<br>GDP per Capita </th>
   <th style="text-align:right;"> 2017<br>GDP per Capita </th>
   <th style="text-align:right;"> 2018<br>GDP per Capita </th>
   <th style="text-align:right;"> Avg<br>GDP per Capita </th>
   <th style="text-align:right;">   </th>
   <th style="text-align:right;"> 2015<br>Within </th>
   <th style="text-align:right;"> 2016<br>Within </th>
   <th style="text-align:right;"> 2017<br>Within </th>
   <th style="text-align:right;"> 2018<br>Within </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> ARG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">13789.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5caae">13360.21</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #afde9d">13595.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">13105.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d3a7">13462.43</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5cbae">-0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b0dd9e">0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.36</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ATG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">14285.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c9b0">14919.21</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d2a7">15242.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">16146.59</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cbcfaa">15148.38</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.86</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">-0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a7">0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BHS </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4cbae">31776.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">31491.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e2c4b4">31682.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">32231.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d1ccac">31795.43</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5cbae">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.30</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e2c4b4">-0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.44</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BLZ </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">4770.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f6babd">4671.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">4663.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d1cdac">4707.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5caae">4703.21</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f4bbbc">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caaf">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BOL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">3035.97</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">3118.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">3203.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">3291.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">3162.26</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.13</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">8813.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">8455.31</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f1bcbb">8498.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">8582.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caae">8587.48</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f2bcbb">-0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">16524.90</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9de696">16906.59</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">16961.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #afde9d">16838.73</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">16807.89</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9ce795">0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aede9d">0.03</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CHL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">13574.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #edbeb9">13624.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f9b8be">13590.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">13901.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc6b2">13672.71</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #eebeb9">-0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f8b9be">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.23</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> COL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">6175.88</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cccfaa">6219.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">6208.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6271.88</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdcfab">6218.97</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdceab">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.05</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">11642.78</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d3cbad">12004.67</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a7e29a">12375.92</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">12573.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c2d4a6">12149.33</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.51</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">-0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a6e29a">0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.42</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CUB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">7694.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f4bbbc">7726.50</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a9">7863.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">8040.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d3ccad">7831.22</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f2bcbb">-0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d1a9">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.21</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> DOM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">6921.52</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">7300.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdd7a4">7556.85</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">7997.76</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d1a9">7444.04</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.52</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">-0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdd6a4">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.55</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ECU </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6124.49</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">5947.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e9c0b7">5981.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fbb7bf">5952.22</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc7b2">6001.21</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ebbfb8">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.05</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GRD </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">9096.54</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc5b3">9380.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8d9a1">9742.60</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">10109.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d0a9">9582.11</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.49</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc5b3">-0.20</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8d9a1">0.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.53</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GTM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">3994.64</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e4c3b5">4034.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bed6a4">4091.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">4160.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cccfaa">4070.03</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e4c3b5">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdd6a4">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.09</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GUY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">5576.83</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc7b2">5759.69</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bbd8a3">5945.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6178.89</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d0a9">5865.12</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc6b2">-0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bad8a2">0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.31</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HND </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">2286.20</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc5b3">2334.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">2406.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">2457.97</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">2371.42</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e0c5b3">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aede9d">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.09</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HTI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">1386.85</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e6c2b5">1393.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a5e399">1409.63</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1415.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">1401.17</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> JAM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">4907.93</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc7b2">4949.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d1a9">4973.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">5043.54</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdcfab">4968.64</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dcc7b1">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c3d4a6">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.07</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> LCA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">10093.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7caaf">10409.54</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b0dd9e">10718.86</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">10975.83</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d2a7">10549.46</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7caaf">-0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b0dd9e">0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.43</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> MEX </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">9616.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d1ccac">9751.57</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">9842.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">9945.78</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">9789.10</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d3ccad">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.16</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> NIC </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">2049.85</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d1a9">2115.93</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">2185.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e1c4b3">2086.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ceceab">2109.43</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc6b3">-0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PAN </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">13630.30</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">14062.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a9e19b">14596.74</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">14880.66</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">14292.54</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.66</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">-0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a9e19b">0.30</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.59</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PER </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">6229.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ceceab">6380.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdd7a4">6432.92</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6574.32</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">6404.09</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ceceab">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdd6a4">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.17</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f9b8be">29763.49</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">29961.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e1c5b3">29809.22</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">29753.36</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">29821.96</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f9b8be">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc6b3">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.07</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">5413.78</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">5570.61</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aae09b">5762.73</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">5871.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">5654.60</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9af">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aae09b">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.22</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SLV </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">3705.58</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">3781.38</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5daa0">3846.97</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">3920.53</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">3813.61</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caaf">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8d9a1">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.11</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SUR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">9168.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">8628.86</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f4bbbc">8677.78</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aede9d">9020.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cccfaa">8873.83</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f6babd">-0.20</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #addf9c">0.15</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> TTO </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">18214.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caae">17103.87</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fbb7bf">16515.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">16457.19</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9af">17072.64</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caae">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fbb7bf">-0.56</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.62</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> URY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">15613.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a9">15821.36</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #94eb92">16020.38</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">16037.93</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bbd8a3">15873.35</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">-0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #92ec91">0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.16</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VCT </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">6921.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4cbae">7031.73</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d4a5">7078.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">7206.50</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d1a9">7059.68</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4cbae">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d4a5">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.15</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VIR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">34007.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bfd5a5">34614.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">34435.49</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">35073.63</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">34532.81</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.53</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bfd5a5">0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">-0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.54</span> </td>
  </tr>
</tbody>
</table>


### Population

<table class="table table-condensed">
 <thead>
  <tr>
   <th style="text-align:right;"> iso3c </th>
   <th style="text-align:right;"> 2015<br>Population </th>
   <th style="text-align:right;"> 2016<br>Population </th>
   <th style="text-align:right;"> 2017<br>Population </th>
   <th style="text-align:right;"> 2018<br>Population </th>
   <th style="text-align:right;"> Avg<br>Population </th>
   <th style="text-align:right;">   </th>
   <th style="text-align:right;"> 2015<br>Within </th>
   <th style="text-align:right;"> 2016<br>Within </th>
   <th style="text-align:right;"> 2017<br>Within </th>
   <th style="text-align:right;"> 2018<br>Within </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> ARG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">43131966</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">43590368</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">44044811</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">44494502</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">43815411.8</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.68</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ATG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">93571</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9af">94520</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3dc9f">95425</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">96282</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">94949.5</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BHS </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">374200</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">377923</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5daa0">381749</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">385635</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">379876.8</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BLZ </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">360926</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">368399</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">375775</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">383071</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">372042.8</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BOL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">10869732</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">11031822</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">11192853</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">11353140</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">11111886.8</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.24</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">204471759</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">206163056</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">207833825</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">209469320</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">206984490.0</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-2.51</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">-0.82</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">0.85</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">2.48</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">285327</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7caaf">285798</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">286229</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">286640</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a7">285998.5</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CHL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">17969356</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc7b1">18209072</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5daa0">18470435</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">18729166</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">18344507.2</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.38</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc7b1">-0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.38</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> COL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">47520667</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc7b2">48175048</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa1">48909844</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49661056</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a9">48566653.8</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-1.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dcc7b1">-0.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa1">0.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1.09</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">4847805</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c9b0">4899336</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4db9f">4949955</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">4999443</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">4924134.8</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5cbae">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.08</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CUB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">11324777</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #afdd9e">11335108</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">11339255</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #98e993">11338146</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5daa0">11334321.5</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> DOM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">10281675</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">10397738</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">10513111</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">10627147</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">10454917.8</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc8b1">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3db9f">0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.17</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ECU </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">16212022</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc7b1">16491116</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">16785356</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">17084359</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">16643213.2</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.43</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc8b1">-0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.44</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GRD </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">109603</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7caaf">110263</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">110874</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">111449</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a7">110547.2</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GTM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">15567419</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">15827690</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">16087418</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">16346950</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">15957369.2</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.39</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GUY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">767433</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c9b0">771363</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">775218</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">779007</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">773255.2</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HND </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">9112904</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">9270794</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">9429016</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">9587523</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">9350059.2</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.24</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HTI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">10695540</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">10839976</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">10982367</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">11123183</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">10910266.5</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.21</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.21</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> JAM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">2891024</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9af">2906242</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3dc9f">2920848</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">2934853</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">2913241.8</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> LCA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">179131</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">180028</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">180955</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">181890</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">180501.0</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> MEX </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">121858251</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c9b0">123333379</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4db9f">124777326</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">126190782</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">124039934.5</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-2.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c9b0">-0.71</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4db9f">0.74</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">2.15</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> NIC </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">6223234</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">6303970</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">6384843</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6465502</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">6344387.2</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.12</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PAN </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">3968490</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">4037073</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">4106764</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">4176868</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">4072298.8</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc6b2">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa1">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.10</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PER </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">30470739</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc6b2">30926036</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7d9a1">31444299</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">31989265</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d1a9">31207584.8</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.74</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc6b2">-0.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">0.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.78</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">3473232</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aae09b">3406672</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cad0a9">3325286</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">3193354</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d5a5">3349636.0</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a7e29a">0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.16</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">6688746</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">6777878</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">6867058</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6956069</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">6822437.8</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.13</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SLV </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">6325121</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">6356137</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5daa0">6388124</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6420740</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">6372530.5</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc6b2">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.05</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SUR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">559136</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c9b0">564883</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4db9f">570501</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">575987</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">567626.8</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> TTO </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">1370332</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5caae">1377563</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b0dd9e">1384060</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1389841</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d3a7">1380449.0</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> URY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">3412013</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">3424139</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">3436645</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">3449290</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">3430521.8</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VCT </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">109135</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dcc7b1">109467</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7d9a1">109826</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">110210</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a9">109659.5</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VIR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">107712</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aede9d">107516</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d3ccad">107281</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">107001</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">107377.5</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
</tbody>
</table>

### Percent female

<table class="table table-condensed">
 <thead>
  <tr>
   <th style="text-align:right;"> iso3c </th>
   <th style="text-align:right;"> 2015<br>%Female </th>
   <th style="text-align:right;"> 2016<br>%Female </th>
   <th style="text-align:right;"> 2017<br>%Female </th>
   <th style="text-align:right;"> 2018<br>%Female </th>
   <th style="text-align:right;"> Avg<br>%Female </th>
   <th style="text-align:right;">   </th>
   <th style="text-align:right;"> 2015<br>Within </th>
   <th style="text-align:right;"> 2016<br>Within </th>
   <th style="text-align:right;"> 2017<br>Within </th>
   <th style="text-align:right;"> 2018<br>Within </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> ARG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">51.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">51.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">51.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">51.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">51.26</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ATG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">51.88</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d5a5">51.84</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e6c2b6">51.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">51.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdceab">51.83</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d5a5">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e6c2b6">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.04</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BHS </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">51.47</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">51.45</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c3b4">51.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">51.43</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">51.45</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BLZ </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc6b2">50.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">50.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.19</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">50.14</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc6b2">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.05</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BOL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.74</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c3b4">49.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">49.77</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.78</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">49.76</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.77</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">50.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">50.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.83</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">50.80</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.03</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">51.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">51.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">51.71</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">51.67</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">51.73</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.06</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CHL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.78</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">50.76</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">50.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.73</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">50.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a6e299">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.03</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> COL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">50.97</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">50.95</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.93</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">50.96</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.03</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.97</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c3b4">49.98</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">49.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">49.99</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CUB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.31</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.31</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">50.32</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">50.32</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> DOM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc6b2">49.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bfd5a5">49.98</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfceab">49.97</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5cbae">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d9a2">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.04</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ECU </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">49.95</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">49.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.97</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">49.96</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GRD </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.56</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">49.58</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">49.59</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.61</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">49.58</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a6e299">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GTM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">50.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">50.78</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.76</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">50.78</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GUY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bed6a4">49.92</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e1c4b4">49.86</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cad0a9">49.90</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8d9a2">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e1c4b4">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.09</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HND </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">50.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">50.06</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HTI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.64</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.64</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.64</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> JAM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.30</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">50.32</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abdf9c">50.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">50.32</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> LCA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.76</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> MEX </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">51.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">51.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">51.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">51.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">51.09</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> NIC </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.71</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.71</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.71</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.71</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.71</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PAN </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.85</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">49.87</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">49.89</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">49.88</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.03</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PER </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">50.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">50.34</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">52.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e0c5b3">52.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bad8a2">52.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">52.45</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d1a9">52.30</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e0c5b3">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bad8a2">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.15</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c3b4">49.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">49.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">49.13</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SLV </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">52.98</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4cbae">53.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">53.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">53.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c3d4a6">53.05</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7caaf">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.07</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SUR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.69</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">49.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">49.71</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">49.70</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> TTO </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.54</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">50.56</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">50.57</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.59</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">50.57</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> URY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">51.77</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">51.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">51.74</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">51.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">51.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VCT </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc5b3">49.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7d9a1">49.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.21</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">49.14</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e5c2b5">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bad8a2">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.07</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VIR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">52.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e6c2b6">52.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">52.41</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">52.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdceab">52.39</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e6c2b6">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.05</span> </td>
  </tr>
</tbody>
</table>

### Percent rural

<table class="table table-condensed">
 <thead>
  <tr>
   <th style="text-align:right;"> iso3c </th>
   <th style="text-align:right;"> 2015<br>%Rural </th>
   <th style="text-align:right;"> 2016<br>%Rural </th>
   <th style="text-align:right;"> 2017<br>%Rural </th>
   <th style="text-align:right;"> 2018<br>%Rural </th>
   <th style="text-align:right;"> Avg<br>%Rural </th>
   <th style="text-align:right;">   </th>
   <th style="text-align:right;"> 2015<br>Within </th>
   <th style="text-align:right;"> 2016<br>Within </th>
   <th style="text-align:right;"> 2017<br>Within </th>
   <th style="text-align:right;"> 2018<br>Within </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> ARG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">8.50</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">8.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc8b1">8.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">8.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a9">8.31</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.18</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ATG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">75.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5cbae">75.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aede9d">75.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">75.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">75.21</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.21</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5cbae">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aede9d">0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.19</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BHS </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">17.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b0dd9e">17.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5caae">17.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">16.98</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d3a7">17.12</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3dc9f">0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7caaf">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.14</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BLZ </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">54.59</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #acdf9c">54.51</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4cbae">54.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">54.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d2a7">54.44</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #afde9d">0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d1ccad">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.17</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BOL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">31.61</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5daa0">31.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">30.92</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">30.58</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">31.09</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.52</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">-0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.52</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">14.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">13.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">13.69</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">13.43</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">13.83</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">-0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.40</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">68.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">68.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9be894">68.84</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">68.85</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">68.81</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9be894">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.04</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CHL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">12.64</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">12.58</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9af">12.51</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">12.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">12.54</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #afde9e">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4cbae">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.11</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> COL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">20.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">19.89</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc8b1">19.55</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">19.22</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">19.73</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.51</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.50</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">23.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">22.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dcc7b1">21.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">20.66</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">21.88</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa1">0.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dcc7b1">-0.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-1.22</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CUB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">23.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a7e19a">23.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfcdac">23.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">22.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bfd5a5">23.04</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a7e29a">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfceac">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.08</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> DOM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">21.43</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa1">20.56</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc7b1">19.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">18.93</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">20.16</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">0.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc7b1">-0.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-1.24</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ECU </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">36.60</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">36.47</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">36.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">36.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">36.39</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.21</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4db9f">0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5caae">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.22</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GRD </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">64.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #acdf9c">63.93</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d1ccad">63.84</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">63.73</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d3a7">63.87</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #afde9e">0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d3ccad">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.15</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GTM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3db9f">49.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">49.32</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">48.95</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">49.49</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.54</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3dc9f">0.19</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9af">-0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.55</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GUY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">73.56</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aae09b">73.52</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d1cdac">73.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">73.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">73.48</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b0dd9e">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d1cdac">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.09</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HND </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">44.84</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">44.19</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">43.54</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">42.90</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">43.87</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.97</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.32</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.32</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.96</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HTI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">47.57</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5daa0">46.60</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">45.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">44.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">46.14</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1.43</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.47</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.48</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-1.42</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> JAM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">45.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3db9f">44.90</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">44.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">44.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">44.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.41</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7caaf">-0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.43</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> LCA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">81.48</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abdf9c">81.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ceceab">81.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">81.32</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c0d5a5">81.41</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b0dd9e">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d1cdac">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.09</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> MEX </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">20.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5daa0">20.42</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">20.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">19.84</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">20.28</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc8b1">-0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.43</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> NIC </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">42.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">41.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">41.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">41.48</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d2a7">41.80</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.31</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3dc9f">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9af">-0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.32</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PAN </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">33.30</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">32.97</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">32.63</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">32.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">32.80</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.50</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">-0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.51</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PER </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">22.64</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">22.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">22.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">22.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">22.37</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">-0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.28</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">6.38</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">6.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">6.41</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6.42</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">6.40</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a6e299">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">39.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">38.97</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">38.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">38.42</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">38.83</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.42</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">-0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.42</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SLV </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">30.30</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">29.50</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc8b1">28.73</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">27.98</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">29.13</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">0.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc8b1">-0.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-1.15</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SUR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">33.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">33.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">33.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">33.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">33.95</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> TTO </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">46.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">46.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9fe597">46.80</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">46.82</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bfd5a5">46.76</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a7e29a">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.06</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> URY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">4.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">4.86</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dcc7b1">4.76</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">4.67</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d1a9">4.81</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dcc7b1">-0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.14</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VCT </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">48.63</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">48.22</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">47.80</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">48.42</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">0.21</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">-0.20</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.62</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VIR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">4.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">4.52</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">4.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">4.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d1a9">4.46</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.19</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc8b1">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.18</span> </td>
  </tr>
</tbody>
</table>

## Bookdown Style Note

To get the above tables to display in bookdown I added HTML code to allow the maximum width to be larger than the default setting. If you find this messes with the rest of your book, you can remove from here to the bottom of this file instide the HTML "style" tag.


<style>
.book .book-body .page-wrapper .page-inner {
  max-width: 90%;
}
</style>
