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
<tr><td colspan="9" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">lifeExp</td><td>136</td><td>74.654</td><td>3.503</td><td>63.237</td><td>72.986</td><td>74.472</td><td>77.016</td><td>80.350</td></tr>
<tr><td style="text-align:left">gdpPerCapita</td><td>136</td><td>11.695</td><td>8.803</td><td>1.404</td><td>5.974</td><td>8.449</td><td>15.592</td><td>35.183</td></tr>
<tr><td style="text-align:left">pop</td><td>136</td><td>17.640</td><td>40.122</td><td>0.037</td><td>0.373</td><td>4.530</td><td>11.365</td><td>210.167</td></tr>
<tr><td style="text-align:left">pctFemale</td><td>136</td><td>50.647</td><td>1.022</td><td>48.782</td><td>49.877</td><td>50.410</td><td>51.056</td><td>52.814</td></tr>
<tr><td style="text-align:left">pctRural</td><td>136</td><td>35.285</td><td>21.488</td><td>4.279</td><td>19.471</td><td>33.622</td><td>48.319</td><td>81.485</td></tr>
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

At the end of this file there are tables that demonstrate the within transformation for our dataset. There is a table for each variable. Look at the table for [Life expectancy]. Find the row for Argentina (iso3c code ARG). It's average value of life expectancy is 76.72.  In 2015, their value was 76.76, which is 0.04 below Argentina's four-year average value of 76.72. In 2016, Argentina's life expectancy was 76.31, which is 0.41 above Argentina's four-year average. Below this table for life expectancy is a similar table for each explanatory variable. When economists say a model has country "fixed effects", they mean estimating an OLS regression using data transformed by this "within" transformation.


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
<tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">gdpPerCapita</td><td>0.1131 (0.0600)<sup>*</sup></td><td>0.1131 (0.0519)<sup>**</sup></td><td>0.1131 (0.0600)<sup>*</sup></td></tr>
<tr><td style="text-align:left">pop</td><td>0.0598 (0.0503)</td><td>0.0598 (0.0435)</td><td>0.0598 (0.0503)</td></tr>
<tr><td style="text-align:left">pctFemale</td><td>0.6970 (0.5382)</td><td>0.6970 (0.4655)</td><td>0.6970 (0.5382)</td></tr>
<tr><td style="text-align:left">pctRural</td><td>-0.1476 (0.0600)<sup>**</sup></td><td>-0.1476 (0.0519)<sup>***</sup></td><td>-0.1476 (0.0600)<sup>**</sup></td></tr>
<tr><td style="text-align:left">factor(country)Argentina</td><td>-12.4450 (4.1437)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Aruba</td><td>-6.9573 (1.2192)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Bahamas, The</td><td>-14.6820 (3.5130)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Barbados</td><td>-2.3363 (0.4518)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Belize</td><td>-4.8728 (2.1054)<sup>**</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Bolivia</td><td>-14.5484 (3.2176)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Brazil</td><td>-23.1193 (9.7968)<sup>**</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Chile</td><td>-6.8776 (3.9218)<sup>*</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Colombia</td><td>-10.4753 (3.6862)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Costa Rica</td><td>-4.8909 (3.6734)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Cuba</td><td>-6.5904 (3.4765)<sup>*</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Dominican Republic</td><td>-11.0797 (3.8017)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Ecuador</td><td>-5.2878 (2.7981)<sup>*</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)El Salvador</td><td>-11.8277 (2.8804)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Grenada</td><td>-2.4931 (1.6301)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Guatemala</td><td>-7.9023 (2.0302)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Guyana</td><td>-7.8718 (0.8472)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Haiti</td><td>-16.5223 (2.2762)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Honduras</td><td>-7.2340 (2.6626)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Jamaica</td><td>-8.2572 (2.3559)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Mexico</td><td>-17.7523 (5.9223)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Nicaragua</td><td>-7.4429 (2.4331)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Panama</td><td>-5.1770 (3.0542)<sup>*</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Paraguay</td><td>-7.5663 (2.8103)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Peru</td><td>-9.6800 (3.4234)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Puerto Rico</td><td>-10.4554 (4.0113)<sup>**</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)St. Lucia</td><td>-2.1323 (1.0039)<sup>**</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)St. Vincent and the Grenadines</td><td>-4.5311 (2.7316)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Suriname</td><td>-10.0725 (3.0393)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Trinidad and Tobago</td><td>-7.2752 (2.0746)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Turks and Caicos Islands</td><td>-10.5865 (4.6999)<sup>**</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Uruguay</td><td>-10.7839 (4.2651)<sup>**</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Virgin Islands (U.S.)</td><td>-11.7226 (4.1648)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">Constant</td><td>51.0531 (29.2668)<sup>*</sup></td><td>-0.0000 (0.0186)</td><td></td></tr>
<tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Observations</td><td>136</td><td>136</td><td>136</td></tr>
<tr><td style="text-align:left">R<sup>2</sup></td><td>0.9963</td><td>0.2153</td><td>0.2153</td></tr>
<tr><td style="text-align:left">Adjusted R<sup>2</sup></td><td>0.9949</td><td>0.1913</td><td>-0.0810</td></tr>
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
<tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">gdpPerCapita</td><td>0.2125 (0.0398)<sup>***</sup></td><td>0.2125 (0.0394)<sup>***</sup></td><td>0.2125 (0.0398)<sup>***</sup></td></tr>
<tr><td style="text-align:left">pop</td><td>0.0033 (0.0069)</td><td>0.0033 (0.0069)</td><td>0.0033 (0.0069)</td></tr>
<tr><td style="text-align:left">pctFemale</td><td>-0.2282 (0.3154)</td><td>-0.2282 (0.3118)</td><td>-0.2282 (0.3154)</td></tr>
<tr><td style="text-align:left">pctRural</td><td>-0.0343 (0.0135)<sup>**</sup></td><td>-0.0343 (0.0133)<sup>**</sup></td><td>-0.0343 (0.0135)<sup>**</sup></td></tr>
<tr><td style="text-align:left">factor(year)2016</td><td>0.0918 (0.6993)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(year)2017</td><td>0.1802 (0.6994)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(year)2018</td><td>0.2203 (0.6994)</td><td></td><td></td></tr>
<tr><td style="text-align:left">Constant</td><td>84.7549 (15.5578)<sup>***</sup></td><td>-0.0000 (0.2444)</td><td></td></tr>
<tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Observations</td><td>136</td><td>136</td><td>136</td></tr>
<tr><td style="text-align:left">R<sup>2</sup></td><td>0.3578</td><td>0.3570</td><td>0.3570</td></tr>
<tr><td style="text-align:left">Adjusted R<sup>2</sup></td><td>0.3226</td><td>0.3374</td><td>0.3219</td></tr>
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
<tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">gdpPerCapita</td><td>0.0926 (0.0597)</td><td>0.0926 (0.0509)<sup>*</sup></td><td>0.0926 (0.0597)</td></tr>
<tr><td style="text-align:left">pop</td><td>0.0140 (0.0524)</td><td>0.0140 (0.0446)</td><td>0.0140 (0.0524)</td></tr>
<tr><td style="text-align:left">pctFemale</td><td>0.2558 (0.5550)</td><td>0.2558 (0.4726)</td><td>0.2558 (0.5550)</td></tr>
<tr><td style="text-align:left">pctRural</td><td>-0.0450 (0.0708)</td><td>-0.0450 (0.0603)</td><td>-0.0450 (0.0708)</td></tr>
<tr><td style="text-align:left">factor(year)2016</td><td>0.0877 (0.0636)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(year)2017</td><td>0.1715 (0.0749)<sup>**</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(year)2018</td><td>0.2247 (0.0889)<sup>**</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Argentina</td><td>-4.4289 (5.1083)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Aruba</td><td>-4.6134 (1.5027)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Bahamas, The</td><td>-8.5611 (4.1804)<sup>**</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Barbados</td><td>-1.7206 (0.5040)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Belize</td><td>-4.0949 (2.0919)<sup>*</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Bolivia</td><td>-10.9124 (3.4576)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Brazil</td><td>-8.1467 (11.1984)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Chile</td><td>-0.5159 (4.5668)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Colombia</td><td>-3.5323 (4.4966)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Costa Rica</td><td>-0.3523 (4.0134)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Cuba</td><td>-1.8233 (3.8771)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Dominican Republic</td><td>-6.3070 (4.1647)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Ecuador</td><td>-1.7799 (3.0648)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)El Salvador</td><td>-7.0790 (3.3555)<sup>**</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Grenada</td><td>-2.5864 (1.6033)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Guatemala</td><td>-5.5944 (2.1831)<sup>**</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Guyana</td><td>-8.4840 (0.8622)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Haiti</td><td>-14.2080 (2.4052)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Honduras</td><td>-5.1260 (2.7409)<sup>*</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Jamaica</td><td>-6.0970 (2.4589)<sup>**</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Mexico</td><td>-7.2146 (7.0815)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Nicaragua</td><td>-4.7104 (2.6076)<sup>*</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Panama</td><td>-1.7303 (3.2873)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Paraguay</td><td>-4.8894 (2.9497)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Peru</td><td>-3.8475 (4.0416)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Puerto Rico</td><td>-2.9235 (4.8889)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)St. Lucia</td><td>-3.7614 (1.1615)<sup>***</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)St. Vincent and the Grenadines</td><td>-3.4942 (2.7188)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Suriname</td><td>-6.9784 (3.2165)<sup>**</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Trinidad and Tobago</td><td>-4.9854 (2.2263)<sup>**</sup></td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Turks and Caicos Islands</td><td>-4.6965 (5.1569)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Uruguay</td><td>-3.6969 (4.9925)</td><td></td><td></td></tr>
<tr><td style="text-align:left">factor(country)Virgin Islands (U.S.)</td><td>-3.9435 (5.0677)</td><td></td><td></td></tr>
<tr><td style="text-align:left">Constant</td><td>66.6238 (29.3962)<sup>**</sup></td><td>-61.9555 (24.0430)<sup>**</sup></td><td></td></tr>
<tr><td colspan="4" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Observations</td><td>136</td><td>136</td><td>136</td></tr>
<tr><td style="text-align:left">R<sup>2</sup></td><td>0.9966</td><td>0.0389</td><td>0.0389</td></tr>
<tr><td style="text-align:left">Adjusted R<sup>2</sup></td><td>0.9951</td><td>0.0095</td><td>-0.3658</td></tr>
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
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">gdpPerCapita</td><td>0.212 (0.039)<sup>***</sup></td><td>0.113 (0.052)<sup>**</sup></td><td>0.212 (0.039)<sup>***</sup></td><td>0.093 (0.051)<sup>*</sup></td></tr>
<tr><td style="text-align:left">pop</td><td>0.003 (0.007)</td><td>0.060 (0.044)</td><td>0.003 (0.007)</td><td>0.014 (0.045)</td></tr>
<tr><td style="text-align:left">pctFemale</td><td>-0.227 (0.312)</td><td>0.697 (0.466)</td><td>-0.228 (0.312)</td><td>0.256 (0.473)</td></tr>
<tr><td style="text-align:left">pctRural</td><td>-0.034 (0.013)<sup>**</sup></td><td>-0.148 (0.052)<sup>***</sup></td><td>-0.034 (0.013)<sup>**</sup></td><td>-0.045 (0.060)</td></tr>
<tr><td style="text-align:left">Constant</td><td>84.800 (15.384)<sup>***</sup></td><td>-0.000 (0.019)</td><td>-0.000 (0.244)</td><td>-61.955 (24.043)<sup>**</sup></td></tr>
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Observations</td><td>136</td><td>136</td><td>136</td><td>136</td></tr>
<tr><td style="text-align:left">R<sup>2</sup></td><td>0.357</td><td>0.215</td><td>0.357</td><td>0.039</td></tr>
<tr><td style="text-align:left">Adjusted R<sup>2</sup></td><td>0.338</td><td>0.191</td><td>0.337</td><td>0.010</td></tr>
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
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">gdpPerCapita</td><td>0.212 (0.039)<sup>***</sup></td><td>0.113 (0.060)<sup>*</sup></td><td>0.212 (0.040)<sup>***</sup></td><td>0.093 (0.060)</td></tr>
<tr><td style="text-align:left">pop</td><td>0.003 (0.007)</td><td>0.060 (0.050)</td><td>0.003 (0.007)</td><td>0.014 (0.052)</td></tr>
<tr><td style="text-align:left">pctFemale</td><td>-0.227 (0.312)</td><td>0.697 (0.538)</td><td>-0.228 (0.315)</td><td>0.256 (0.555)</td></tr>
<tr><td style="text-align:left">pctRural</td><td>-0.034 (0.013)<sup>**</sup></td><td>-0.148 (0.060)<sup>**</sup></td><td>-0.034 (0.013)<sup>**</sup></td><td>-0.045 (0.071)</td></tr>
<tr><td style="text-align:left">Constant</td><td>84.800 (15.384)<sup>***</sup></td><td></td><td></td><td></td></tr>
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Observations</td><td>136</td><td>136</td><td>136</td><td>136</td></tr>
<tr><td style="text-align:left">R<sup>2</sup></td><td>0.357</td><td>0.215</td><td>0.357</td><td>0.039</td></tr>
<tr><td style="text-align:left">Adjusted R<sup>2</sup></td><td>0.338</td><td>-0.081</td><td>0.322</td><td>-0.366</td></tr>
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
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">gdpPerCapita</td><td>0.212 (0.039)<sup>***</sup></td><td>0.113 (0.060)<sup>*</sup></td><td>0.212 (0.040)<sup>***</sup></td><td>0.093 (0.060)</td></tr>
<tr><td style="text-align:left">pop</td><td>0.003 (0.007)</td><td>0.060 (0.050)</td><td>0.003 (0.007)</td><td>0.014 (0.052)</td></tr>
<tr><td style="text-align:left">pctFemale</td><td>-0.227 (0.312)</td><td>0.697 (0.538)</td><td>-0.228 (0.315)</td><td>0.256 (0.555)</td></tr>
<tr><td style="text-align:left">pctRural</td><td>-0.034 (0.013)<sup>**</sup></td><td>-0.148 (0.060)<sup>**</sup></td><td>-0.034 (0.013)<sup>**</sup></td><td>-0.045 (0.071)</td></tr>
<tr><td style="text-align:left">factor(country)Argentina</td><td></td><td>-12.445 (4.144)<sup>***</sup></td><td></td><td>-4.429 (5.108)</td></tr>
<tr><td style="text-align:left">factor(country)Aruba</td><td></td><td>-6.957 (1.219)<sup>***</sup></td><td></td><td>-4.613 (1.503)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Bahamas, The</td><td></td><td>-14.682 (3.513)<sup>***</sup></td><td></td><td>-8.561 (4.180)<sup>**</sup></td></tr>
<tr><td style="text-align:left">factor(country)Barbados</td><td></td><td>-2.336 (0.452)<sup>***</sup></td><td></td><td>-1.721 (0.504)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Belize</td><td></td><td>-4.873 (2.105)<sup>**</sup></td><td></td><td>-4.095 (2.092)<sup>*</sup></td></tr>
<tr><td style="text-align:left">factor(country)Bolivia</td><td></td><td>-14.548 (3.218)<sup>***</sup></td><td></td><td>-10.912 (3.458)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Brazil</td><td></td><td>-23.119 (9.797)<sup>**</sup></td><td></td><td>-8.147 (11.198)</td></tr>
<tr><td style="text-align:left">factor(country)Chile</td><td></td><td>-6.878 (3.922)<sup>*</sup></td><td></td><td>-0.516 (4.567)</td></tr>
<tr><td style="text-align:left">factor(country)Colombia</td><td></td><td>-10.475 (3.686)<sup>***</sup></td><td></td><td>-3.532 (4.497)</td></tr>
<tr><td style="text-align:left">factor(country)Costa Rica</td><td></td><td>-4.891 (3.673)</td><td></td><td>-0.352 (4.013)</td></tr>
<tr><td style="text-align:left">factor(country)Cuba</td><td></td><td>-6.590 (3.476)<sup>*</sup></td><td></td><td>-1.823 (3.877)</td></tr>
<tr><td style="text-align:left">factor(country)Dominican Republic</td><td></td><td>-11.080 (3.802)<sup>***</sup></td><td></td><td>-6.307 (4.165)</td></tr>
<tr><td style="text-align:left">factor(country)Ecuador</td><td></td><td>-5.288 (2.798)<sup>*</sup></td><td></td><td>-1.780 (3.065)</td></tr>
<tr><td style="text-align:left">factor(country)El Salvador</td><td></td><td>-11.828 (2.880)<sup>***</sup></td><td></td><td>-7.079 (3.356)<sup>**</sup></td></tr>
<tr><td style="text-align:left">factor(country)Grenada</td><td></td><td>-2.493 (1.630)</td><td></td><td>-2.586 (1.603)</td></tr>
<tr><td style="text-align:left">factor(country)Guatemala</td><td></td><td>-7.902 (2.030)<sup>***</sup></td><td></td><td>-5.594 (2.183)<sup>**</sup></td></tr>
<tr><td style="text-align:left">factor(country)Guyana</td><td></td><td>-7.872 (0.847)<sup>***</sup></td><td></td><td>-8.484 (0.862)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Haiti</td><td></td><td>-16.522 (2.276)<sup>***</sup></td><td></td><td>-14.208 (2.405)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)Honduras</td><td></td><td>-7.234 (2.663)<sup>***</sup></td><td></td><td>-5.126 (2.741)<sup>*</sup></td></tr>
<tr><td style="text-align:left">factor(country)Jamaica</td><td></td><td>-8.257 (2.356)<sup>***</sup></td><td></td><td>-6.097 (2.459)<sup>**</sup></td></tr>
<tr><td style="text-align:left">factor(country)Mexico</td><td></td><td>-17.752 (5.922)<sup>***</sup></td><td></td><td>-7.215 (7.082)</td></tr>
<tr><td style="text-align:left">factor(country)Nicaragua</td><td></td><td>-7.443 (2.433)<sup>***</sup></td><td></td><td>-4.710 (2.608)<sup>*</sup></td></tr>
<tr><td style="text-align:left">factor(country)Panama</td><td></td><td>-5.177 (3.054)<sup>*</sup></td><td></td><td>-1.730 (3.287)</td></tr>
<tr><td style="text-align:left">factor(country)Paraguay</td><td></td><td>-7.566 (2.810)<sup>***</sup></td><td></td><td>-4.889 (2.950)</td></tr>
<tr><td style="text-align:left">factor(country)Peru</td><td></td><td>-9.680 (3.423)<sup>***</sup></td><td></td><td>-3.848 (4.042)</td></tr>
<tr><td style="text-align:left">factor(country)Puerto Rico</td><td></td><td>-10.455 (4.011)<sup>**</sup></td><td></td><td>-2.924 (4.889)</td></tr>
<tr><td style="text-align:left">factor(country)St. Lucia</td><td></td><td>-2.132 (1.004)<sup>**</sup></td><td></td><td>-3.761 (1.161)<sup>***</sup></td></tr>
<tr><td style="text-align:left">factor(country)St. Vincent and the Grenadines</td><td></td><td>-4.531 (2.732)</td><td></td><td>-3.494 (2.719)</td></tr>
<tr><td style="text-align:left">factor(country)Suriname</td><td></td><td>-10.072 (3.039)<sup>***</sup></td><td></td><td>-6.978 (3.216)<sup>**</sup></td></tr>
<tr><td style="text-align:left">factor(country)Trinidad and Tobago</td><td></td><td>-7.275 (2.075)<sup>***</sup></td><td></td><td>-4.985 (2.226)<sup>**</sup></td></tr>
<tr><td style="text-align:left">factor(country)Turks and Caicos Islands</td><td></td><td>-10.586 (4.700)<sup>**</sup></td><td></td><td>-4.697 (5.157)</td></tr>
<tr><td style="text-align:left">factor(country)Uruguay</td><td></td><td>-10.784 (4.265)<sup>**</sup></td><td></td><td>-3.697 (4.993)</td></tr>
<tr><td style="text-align:left">factor(country)Virgin Islands (U.S.)</td><td></td><td>-11.723 (4.165)<sup>***</sup></td><td></td><td>-3.944 (5.068)</td></tr>
<tr><td style="text-align:left">factor(year)2016</td><td></td><td></td><td>0.092 (0.699)</td><td>0.088 (0.064)</td></tr>
<tr><td style="text-align:left">factor(year)2017</td><td></td><td></td><td>0.180 (0.699)</td><td>0.171 (0.075)<sup>**</sup></td></tr>
<tr><td style="text-align:left">factor(year)2018</td><td></td><td></td><td>0.220 (0.699)</td><td>0.225 (0.089)<sup>**</sup></td></tr>
<tr><td style="text-align:left">Constant</td><td>84.800 (15.384)<sup>***</sup></td><td>51.053 (29.267)<sup>*</sup></td><td>84.755 (15.558)<sup>***</sup></td><td>66.624 (29.396)<sup>**</sup></td></tr>
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Observations</td><td>136</td><td>136</td><td>136</td><td>136</td></tr>
<tr><td style="text-align:left">R<sup>2</sup></td><td>0.357</td><td>0.996</td><td>0.358</td><td>0.997</td></tr>
<tr><td style="text-align:left">Adjusted R<sup>2</sup></td><td>0.338</td><td>0.995</td><td>0.323</td><td>0.995</td></tr>
<tr><td colspan="5" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left"><em>Note:</em></td><td colspan="4" style="text-align:right">Standard Errors reported in parentheses, <em>&#42;p&lt;0.1;&#42;&#42;p&lt;0.05;&#42;&#42;&#42;p&lt;0.01</em></td></tr>
</table>




## Data Summary by Country


```r
stargazer(as.data.frame(mydata) , type = "html",digits = 2 ,summary.stat = c("n","mean","sd", "min", "median", "max"))
```


<table style="text-align:center"><tr><td colspan="7" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">Statistic</td><td>N</td><td>Mean</td><td>St. Dev.</td><td>Min</td><td>Median</td><td>Max</td></tr>
<tr><td colspan="7" style="border-bottom: 1px solid black"></td></tr><tr><td style="text-align:left">year</td><td>136</td><td>2,016.50</td><td>1.12</td><td>2,015</td><td>2,016.5</td><td>2,018</td></tr>
<tr><td style="text-align:left">gdpPerCapita</td><td>136</td><td>11.70</td><td>8.80</td><td>1.40</td><td>8.45</td><td>35.18</td></tr>
<tr><td style="text-align:left">lifeExp</td><td>136</td><td>74.65</td><td>3.50</td><td>63.24</td><td>74.47</td><td>80.35</td></tr>
<tr><td style="text-align:left">pop</td><td>136</td><td>17.64</td><td>40.12</td><td>0.04</td><td>4.53</td><td>210.17</td></tr>
<tr><td style="text-align:left">pctFemale</td><td>136</td><td>50.65</td><td>1.02</td><td>48.78</td><td>50.41</td><td>52.81</td></tr>
<tr><td style="text-align:left">pctRural</td><td>136</td><td>35.28</td><td>21.49</td><td>4.28</td><td>33.62</td><td>81.48</td></tr>
<tr><td style="text-align:left">cAvg_lifeExp</td><td>136</td><td>74.65</td><td>3.49</td><td>63.63</td><td>74.49</td><td>80.08</td></tr>
<tr><td style="text-align:left">cAvg_gdpPerCapita</td><td>136</td><td>11.70</td><td>8.79</td><td>1.42</td><td>8.59</td><td>34.56</td></tr>
<tr><td style="text-align:left">cAvg_pop</td><td>136</td><td>17.64</td><td>40.12</td><td>0.04</td><td>4.51</td><td>207.68</td></tr>
<tr><td style="text-align:left">cAvg_pctFemale</td><td>136</td><td>50.65</td><td>1.02</td><td>48.81</td><td>50.41</td><td>52.69</td></tr>
<tr><td style="text-align:left">cAvg_pctRural</td><td>136</td><td>35.28</td><td>21.48</td><td>4.46</td><td>33.38</td><td>81.41</td></tr>
<tr><td style="text-align:left">yrAvg_lifeExp</td><td>136</td><td>74.65</td><td>0.12</td><td>74.49</td><td>74.66</td><td>74.80</td></tr>
<tr><td style="text-align:left">yrAvg_gdpPerCapita</td><td>136</td><td>11.70</td><td>0.13</td><td>11.54</td><td>11.67</td><td>11.89</td></tr>
<tr><td style="text-align:left">yrAvg_pop</td><td>136</td><td>17.64</td><td>0.21</td><td>17.37</td><td>17.64</td><td>17.92</td></tr>
<tr><td style="text-align:left">yrAvg_pctFemale</td><td>136</td><td>50.65</td><td>0.02</td><td>50.62</td><td>50.65</td><td>50.67</td></tr>
<tr><td style="text-align:left">yrAvg_pctRural</td><td>136</td><td>35.28</td><td>0.28</td><td>34.91</td><td>35.29</td><td>35.66</td></tr>
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
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9cef9c">78.21</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbeef4">15.84</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dbe8">52.32</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #98ef98">75.21</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Argentina </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a6f1a6">76.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e1f0f5">13.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e7fbe7">43.82</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbedf4">50.51</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f9fef9">8.31</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Aruba </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #acf2ac">75.82</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8dde9">29.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #add8e6">52.69</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3f3b3">56.75</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Bahamas, The </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcf4bc">73.52</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7dde9">30.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcdfea">51.98</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ecfcec">17.12</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Barbados </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a5f1a5">76.87</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7ecf3">17.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8dde9">52.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a2f0a2">68.81</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Belize </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcf4bc">73.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f3f9fb">5.93</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ecf6f9">49.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6f3b6">54.44</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Bolivia </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e4fae4">67.60</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fafdfd">3.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f8fef8">11.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ebf5f9">49.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8f9d8">31.09</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Brazil </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4f3b4">74.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #edf6f9">8.56</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">207.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4eaf2">50.80</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f1fcf1">13.83</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Chile </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">80.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e0f0f5">13.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f5fdf5">18.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddeff4">50.38</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f3fdf3">12.54</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Colombia </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a7f1a7">76.53</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f2f9fb">6.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e5fbe5">48.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8ecf3">50.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8fbe8">19.73</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Costa Rica </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #94ee94">79.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e4f2f6">12.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fcfefc">4.97</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8f4f8">49.87</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e5fbe5">21.88</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Cuba </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a0f0a0">77.61</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #eff7fa">7.83</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f8fef8">11.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e1f1f6">50.20</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e4fae4">23.04</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Dominican Republic </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bff5bf">73.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f0f8fa">7.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f9fef9">10.59</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #edf6f9">49.64</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8fbe8">20.16</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Ecuador </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a5f1a5">76.90</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f3f9fb">6.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f6fdf6">16.59</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e6f3f7">49.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d0f7d0">36.39</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> El Salvador </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5f6c5">72.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f8fcfd">3.88</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fbfefb">6.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dce8">52.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbf9db">29.13</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Grenada </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3f3b3">74.84</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ecf6f9">8.80</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #eaf5f8">49.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a9f1a9">63.87</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Guatemala </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c3f5c3">72.43</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f8fbfd">4.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f6fdf6">15.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbeef4">50.47</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bef5be">49.49</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Guyana </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddf9dd">68.54</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f3f9fb">5.92</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.77</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2e9f1">50.92</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9bef9b">73.48</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Haiti </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffffff">63.63</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffffff">1.42</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f9fef9">10.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #deeff5">50.36</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c2f5c2">46.14</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Honduras </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c2f5c2">72.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fcfdfe">2.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f9fef9">9.54</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f1f8fa">49.45</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6f6c6">43.87</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Jamaica </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6f6c6">72.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f5fafc">5.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fdfefd">2.80</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #deeff5">50.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4f6c4">44.75</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Mexico </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6f3b6">74.31</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e9f4f8">9.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdf5bd">122.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfe8f0">51.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8fbe8">20.28</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Nicaragua </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdf4bd">73.41</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fdfefe">2.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fbfefb">6.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6ebf2">50.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9f6c9">41.80</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Panama </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a0f0a0">77.69</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfeff5">14.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fcfefc">4.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e7f3f7">49.93</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6f8d6">32.80</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Paraguay </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcf4bc">73.48</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f3f9fb">6.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fbfefb">6.31</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ebf5f9">49.73</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdf7cd">38.83</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Peru </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #acf2ac">75.82</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f2f9fb">6.36</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #eefcee">31.41</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dceef4">50.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e5fbe5">22.37</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Puerto Rico </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #92ee92">79.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8dde9">29.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fdfefd">3.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dae7">52.41</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fcfefc">6.40</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> St. Lucia </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bef5be">73.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e7f3f7">10.77</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dff0f5">50.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">81.41</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> St. Vincent and the Grenadines </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7f3b7">74.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #eff7fa">7.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffffff">48.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bff5bf">48.42</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Suriname </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7f6c7">71.84</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #edf6f9">8.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.58</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e5f2f7">50.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4f8d4">33.95</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Trinidad and Tobago </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7f4b7">74.20</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8ecf3">16.82</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">1.48</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7ecf2">50.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1f5c1">46.76</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Turks and Caicos Islands </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a6f1a6">76.73</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c3e2ec">25.50</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffffff">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f1f8fa">49.45</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fafefa">7.34</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Uruguay </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a0f0a0">77.57</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbedf4">15.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fdfefd">3.42</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c2e2ec">51.66</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">4.81</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> Virgin Islands (U.S.) </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #95ee95">79.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #add8e6">34.56</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fefefe">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aed8e6">52.63</span> </td>
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
   <td style="text-align:right;"> ABW </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f0bdba">75.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">75.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d8a2">75.90</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">76.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdceab">75.82</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f0bdba">-0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.20</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d8a2">0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.25</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ARG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa1">76.76</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">76.31</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">76.83</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">77.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdd7a3">76.72</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.42</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a9e19b">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.27</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ATG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">77.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">78.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">78.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">78.51</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">78.21</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.30</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.30</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BHS </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">73.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bad8a2">73.54</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #acdf9c">73.63</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">73.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdd7a4">73.52</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.42</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bad8a2">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #acdf9c">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.29</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BLZ </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">73.19</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d1cdac">73.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aede9d">73.56</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">73.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">73.46</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d0cdac">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #adde9d">0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.24</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BOL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">67.32</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aede9d">67.63</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9ce795">67.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">67.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa1">67.60</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aede9d">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9ce795">0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.15</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">74.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #efbdba">74.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7d9a1">74.83</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">75.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdcfab">74.68</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #efbdba">-0.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7d9a1">0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.43</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">76.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">76.82</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">76.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">77.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">76.87</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.22</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">-0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.20</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CHL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">79.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d4a6">80.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">80.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8d9a1">80.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d4a6">80.08</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d4a6">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa1">0.06</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> COL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">76.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfcdac">76.47</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a6e299">76.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">76.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d4a6">76.53</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfceac">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a6e29a">0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.22</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">79.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #95eb92">79.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #acdf9c">79.38</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">79.48</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">79.35</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #95eb92">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.13</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CUB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">77.77</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d3a7">77.64</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f2bcbb">77.53</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">77.50</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d1ccad">77.61</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d3a7">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f2bcbb">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.11</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> DOM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">72.95</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #efbdba">72.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d3cbad">73.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">73.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d3cbad">73.06</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #efbeba">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d3ccad">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.17</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ECU </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f4bbbc">76.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">76.76</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8d9a1">76.97</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">77.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfcdac">76.90</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f8b9be">-0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8d9a1">0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.19</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GRD </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">75.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">74.76</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">74.76</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8c1b7">74.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc7b1">74.84</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fab8bf">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e9c0b7">-0.03</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GTM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">72.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d1cdac">72.36</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #afdd9e">72.55</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">72.73</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">72.43</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aede9d">0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.29</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GUY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">68.20</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e2c4b4">68.38</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">68.67</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">68.90</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d1a9">68.54</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e0c5b3">-0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.36</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HND </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">72.49</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dcc7b1">72.59</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d8a2">72.69</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">72.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">72.65</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc6b2">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8d9a1">0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.17</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HTI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">63.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e9c0b7">63.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a8e19a">63.85</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">64.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">63.63</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8c1b6">-0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a6e29a">0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.39</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> JAM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">72.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4cbae">72.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8c1b7">71.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">71.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">72.03</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.36</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4cbae">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8c1b7">-0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.24</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> LCA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f1bcbb">73.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">73.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f6babd">73.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">73.36</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc5b3">73.18</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f5babc">-0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #fab8be">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.17</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> MEX </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">74.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdd7a4">74.41</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #eac0b8">74.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">74.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ceceab">74.31</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e9c0b7">-0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.30</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> NIC </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">72.98</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc8b1">73.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">73.55</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">73.85</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">73.41</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.43</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc8b1">-0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.44</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PAN </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">77.47</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cbcfaa">77.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a1e597">77.80</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">77.86</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c0d5a5">77.69</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cad0a9">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a3e498">0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.17</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PER </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">75.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ceceab">75.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">75.88</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">76.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a7">75.82</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.20</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d0cdac">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5daa0">0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.18</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfcdac">79.69</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">80.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">79.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">79.77</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d1a9">79.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ceceab">-0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.52</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.49</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">73.19</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09b">73.53</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">73.64</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a1e597">73.57</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">73.48</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09b">0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a3e498">0.08</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SLV </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">71.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dec6b2">72.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">72.31</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">72.56</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">72.18</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.36</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc5b3">-0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.38</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SUR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">70.80</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cccfaa">71.59</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #98e993">72.42</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">72.55</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdd7a3">71.84</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-1.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cccfaa">-0.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #98e993">0.58</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.71</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> TCA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">76.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9be895">76.86</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d4a5">76.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">76.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bad8a2">76.73</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9be895">0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d4a5">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.29</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> TTO </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">74.50</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">74.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bad8a2">74.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">73.80</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bfd6a4">74.20</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.30</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bad8a2">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.40</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> URY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">77.48</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7d9a1">77.57</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">77.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #97e993">77.61</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7d9a1">77.57</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #97ea93">0.04</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VCT </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">74.41</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c3d4a6">74.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">74.31</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">74.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c3d4a6">74.28</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c3d4a6">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.15</span> </td>
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
   <td style="text-align:right;"> ABW </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">28421.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f0bdba">28852.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c0d5a5">30270.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">31705.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfcdac">29812.46</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-1.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f0bdba">-0.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c0d5a5">0.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1.89</span> </td>
  </tr>
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
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">14861.88</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">15570.90</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">15962.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">16967.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cbd0aa">15840.64</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.98</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">-0.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d3a7">0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1.13</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BHS </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">30206.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">29699.19</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">30371.58</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">30705.60</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c2d4a6">30245.65</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.55</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.46</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BLZ </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6142.48</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dc9e">6024.84</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f0bdba">5806.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">5756.85</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cccfaa">5932.56</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.21</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f0bdba">-0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.18</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BOL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">2975.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b1">3054.89</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">3135.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">3219.20</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">3096.19</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.12</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">8783.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">8426.85</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f1bcba">8470.95</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">8553.88</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5caae">8558.73</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.22</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f2bcbb">-0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5caae">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">16990.22</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9be895">17385.20</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">17430.87</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">17220.93</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bbd7a3">17256.81</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9ae894">0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">-0.04</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CHL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">13569.95</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e6c2b6">13644.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #efbdba">13615.52</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">13906.77</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">13684.22</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e7c1b6">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f1bcbb">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.22</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> COL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">6228.43</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3db9f">6290.85</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c0d5a5">6280.66</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6320.76</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c0d5a5">6280.18</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d5a5">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.04</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">11529.95</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4cbae">11893.32</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a8e19a">12267.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">12470.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c2d4a6">12040.35</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.51</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d4cbae">-0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a7e29a">0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.43</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CUB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">7683.76</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f3bbbb">7721.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">7865.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">8048.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">7829.73</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f3bcbb">-0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a7">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.22</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> DOM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">6838.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">7209.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdd7a4">7461.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">7894.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d1a9">7351.38</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.51</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">-0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdd7a4">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.54</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ECU </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6130.59</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">5965.64</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc6b2">6012.80</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f7b9bd">5976.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">6021.32</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dec6b2">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f8b9be">-0.05</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GRD </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">8379.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e0c5b3">8621.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8d9a1">8933.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">9252.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d0a9">8796.79</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.42</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e0c5b3">-0.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8d9a1">0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.46</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GTM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">3994.64</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e5c3b5">4034.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bfd6a4">4091.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">4163.48</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cccfaa">4070.89</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e4c3b5">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdd6a4">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.09</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GUY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">5668.43</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">5852.84</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a5e399">6038.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6127.71</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d4a5">5921.81</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d3cbad">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a5e399">0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.21</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HND </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">2257.22</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc5b3">2303.88</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">2373.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">2423.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">2339.54</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.08</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HTI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">1404.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e7c2b6">1409.58</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a2e498">1425.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1429.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a7">1417.00</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> JAM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">5077.55</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dec6b2">5132.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a7">5172.92</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">5264.20</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cccfaa">5161.72</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e0c5b3">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.10</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> LCA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">10290.38</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caaf">10628.77</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b0dd9e">10942.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">11211.60</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d3a7">10768.25</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.48</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5caae">-0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b0dd9e">0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.44</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> MEX </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">9753.38</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d3cbad">9897.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">9997.69</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">10120.36</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d2a7">9942.15</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.19</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4db9f">0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.18</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> NIC </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">2025.32</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d1a9">2087.71</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">2153.61</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e7c1b6">2052.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfcdac">2079.69</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ecbfb8">-0.03</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PAN </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">13669.56</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">14099.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a9e19b">14634.84</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">14922.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">14331.62</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.66</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">-0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a9e19b">0.30</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.59</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PER </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">6180.19</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdcfaa">6337.66</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d9a2">6400.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6530.50</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d3a7">6362.12</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cccfaa">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d9a2">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.17</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e0c5b3">29763.49</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">29961.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdceab">29809.22</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">29687.21</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfceab">29805.42</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc6b3">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfceac">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.12</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">5861.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">6025.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aae09b">6226.69</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6338.51</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">6112.92</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.23</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SLV </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">3761.51</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">3845.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">3921.32</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">4009.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">3884.39</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc7b1">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7d9a1">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.13</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SUR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">8907.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">8382.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f5babc">8425.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">8750.92</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdceab">8616.78</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f6babd">-0.19</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">0.13</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> TCA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d8a2">25783.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">26417.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">24726.95</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e7c1b6">25080.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cccfaa">25502.09</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d8a2">0.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.92</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.78</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e7c1b6">-0.42</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> TTO </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">18389.53</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">17038.78</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #edbeb9">16134.89</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">15716.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d1cdac">16820.03</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1.57</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c8d1a8">0.22</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #edbeb9">-0.69</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-1.10</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> URY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">15655.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ceceab">15869.43</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9ce795">16088.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">16142.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bed6a4">15938.86</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ceceab">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9be895">0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.20</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VCT </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">7386.74</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdcfaa">7730.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5daa0">7890.82</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">8152.38</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">7790.22</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.40</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdcfab">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5daa0">0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.36</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VIR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">34007.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d2a7">34614.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caaf">34435.49</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">35183.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cad0a9">34560.21</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.55</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a7">0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caae">-0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.62</span> </td>
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
   <td style="text-align:right;"> ABW </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">104257</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caaf">104874</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">105439</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">105962</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d2a7">105133.00</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ARG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">43131966</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">43590368</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">44044811</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">44494502</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">43815411.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">0.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.68</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ATG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">89941</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5caae">90564</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">91119</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">91626</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d2a7">90812.50</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BHS </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">392697</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">395976</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">399020</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">401906</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a7">397399.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BLZ </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">359871</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">367313</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">374693</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">382066</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">370985.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BOL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">11090085</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">11263015</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">11435533</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">11606905</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">11348884.50</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">0.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.26</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">205188205</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">206859578</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">208504960</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">210166592</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">207679833.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-2.49</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">-0.82</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.83</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">2.49</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">278083</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">278649</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b2dc9f">279187</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">279688</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">278901.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CHL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">17870124</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e2c4b4">18083879</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">18368577</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">18701450</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cbcfaa">18256007.50</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e1c4b4">-0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.45</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> COL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">47119728</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e4c3b5">47625955</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bfd5a5">48351671</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49276961</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cccfaa">48093578.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.97</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e5c3b5">-0.47</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bfd6a4">0.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1.18</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">4895242</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">4945205</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3db9f">4993842</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">5040734</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">4968755.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7caaf">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #afde9e">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.07</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CUB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a1e597">11339894</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">11342012</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bdd7a3">11336405</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">11328244</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bbd8a3">11336638.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> DOM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">10405832</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">10527592</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">10647244</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">10765531</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">10586549.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.18</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ECU </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">16195902</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dec6b2">16439585</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bbd8a3">16696944</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">17015672</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cad0a9">16587025.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dec6b2">-0.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bbd8a3">0.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.43</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GRD </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">118980</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">119966</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3dc9f">120921</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">121838</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">120426.25</span> </td>
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
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">15957369.25</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.39</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GUY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">755031</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f0bdba">759087</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e1c5b3">763252</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">785514</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9af">765721.00</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HND </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">9294505</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">9460798</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">9626842</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">9792850</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">9543748.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.25</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c9b0">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5daa0">0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.25</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HTI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">10563757</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">10713849</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">10863543</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">11012421</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">10788392.50</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.22</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c9b0">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3dc9f">0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.22</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> JAM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">2794445</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cad0a9">2802695</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a6e299">2808376</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">2811835</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bfd5a5">2804337.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> LCA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">175623</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9af">176413</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3dc9f">177163</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">177888</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a8">176771.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> MEX </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">120149897</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">121519221</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dc9e">122839258</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">124013861</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c6d2a7">122130559.25</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-1.98</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7c9af">-0.61</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b1dd9e">0.71</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1.88</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> NIC </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">6298598</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">6389235</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">6480532</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6572233</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">6435149.50</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dbc8b1">-0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b3dc9f">0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.14</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PAN </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">3957099</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">4026336</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">4096063</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">4165255</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">4061188.25</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9af">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa1">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.10</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PER </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">30711863</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc5b3">31132779</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">31605486</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">32203944</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cad0a9">31413518.00</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc5b3">-0.28</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">0.19</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.79</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">3473232</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aae09b">3406672</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cad0a9">3325286</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">3193354</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d5a5">3349636.00</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.12</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a7e29a">0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.16</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">6177950</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">6266615</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">6355404</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6443328</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">6310824.25</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.13</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SLV </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">6231066</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfceab">6250510</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a7e29a">6266654</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">6276342</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d5a5">6256143.00</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a6e299">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SUR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">575475</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">581453</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">587559</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">593715</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">584550.50</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> TCA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">36538</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9b0">38246</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">39844</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">41487</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">39028.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> TTO </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">1460177</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8c1b6">1469330</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d1cdac">1478607</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">1504709</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">1478205.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8c1b7">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.03</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> URY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">3402818</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cccfaa">3413766</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a6e299">3422200</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">3427042</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c0d5a5">3416456.50</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VCT </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">106482</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bfd5a5">105963</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e6c2b6">105549</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">105281</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cdcfab">105818.75</span> </td>
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
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c4d3a7">107377.50</span> </td>
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
   <td style="text-align:right;"> ABW </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">52.59</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9af">52.66</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa1">52.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">52.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">52.69</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d8c9af">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa1">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.10</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ARG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.52</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">50.51</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">50.50</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.49</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">50.51</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ATG </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">52.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">52.33</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ecbfb8">52.30</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">52.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">52.32</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bfd6a5">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc6b3">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.03</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BHS </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">51.91</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">51.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">52.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">52.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cbd0aa">51.98</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.08</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dcc7b1">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d9a2">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.08</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BLZ </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.71</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">49.69</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">49.67</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">49.68</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.03</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BOL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8c1b7">49.71</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">49.73</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">49.72</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.03</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.77</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">50.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a6e299">50.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.82</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">50.80</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #a6e299">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> BRB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">52.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">52.15</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">52.13</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">52.11</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">52.14</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.03</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CHL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">50.38</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">50.38</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> COL </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.61</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.62</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.62</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.85</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8c1b7">49.86</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">49.88</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.90</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">49.87</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.03</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> CUB </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">50.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d9a2">50.21</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">50.20</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d9a2">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.04</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> DOM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.60</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5cbae">49.63</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d8a2">49.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.68</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">49.64</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5cbae">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d9a2">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.04</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> ECU </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.99</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GRD </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7caaf">49.77</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #afde9d">49.82</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.86</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">49.79</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d7caaf">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.07</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GTM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.47</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.47</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.47</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.47</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> GUY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.78</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c5d2a7">50.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">51.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e2c4b4">50.86</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cccfaa">50.92</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.06</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HND </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">49.45</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">49.45</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> HTI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e8c1b7">50.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bcd7a3">50.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">50.36</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> JAM </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.34</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.34</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> LCA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.23</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d9c8b0">50.27</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b4dba0">50.31</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.35</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">50.29</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b5dba0">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.06</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> MEX </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">51.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">51.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">51.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">51.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">51.05</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> NIC </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.73</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.72</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">0.00</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PAN </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.92</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.92</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">49.93</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.94</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">49.93</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PER </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">50.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.43</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.43</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dac8b0">50.44</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">0.00</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.01</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRI </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">52.31</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ddc6b2">52.37</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa1">52.44</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">52.51</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">52.41</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.10</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc6b3">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bad8a2">0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.11</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> PRY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.71</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">49.73</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">49.74</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.75</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d1a8">49.73</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c4b4">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #abe09c">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.02</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SLV </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">52.24</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e3c3b4">52.26</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d8a2">52.29</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">52.32</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c7d2a8">52.28</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e6c2b6">-0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c1d5a5">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.05</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> SUR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.96</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e5c2b5">49.99</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bad8a2">50.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.09</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cbcfaa">50.02</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e5c2b5">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bad8a2">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.07</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> TCA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">49.39</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d6caaf">49.43</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8d9a1">49.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">49.50</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c2d4a6">49.45</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cccfaa">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aede9d">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.05</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> TTO </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aae09b">50.72</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9de795">50.74</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">50.76</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">50.59</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">50.70</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #aae09b">0.02</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #9de795">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.11</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> URY </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">51.70</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bfd5a5">51.67</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc5b3">51.65</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">51.63</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfcdac">51.66</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d9a2">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d5cbae">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.04</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VCT </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">48.78</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #efbdba">48.79</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfceab">48.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">48.85</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #cfceab">48.81</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.03</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc6b3">-0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #bfd6a5">0.01</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.04</span> </td>
  </tr>
  <tr>
   <td style="text-align:right;"> VIR </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">52.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dcc7b1">52.57</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b6daa0">52.69</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">52.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d1a9">52.63</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.17</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dfc6b3">-0.07</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b9d9a2">0.05</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.18</span> </td>
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
   <td style="text-align:right;"> ABW </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">56.89</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #addf9d">56.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">56.71</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">56.59</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c3d3a6">56.75</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #addf9d">0.06</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #d2ccad">-0.04</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.16</span> </td>
  </tr>
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
   <td style="text-align:right;"> TCA </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">7.81</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b8d9a1">7.48</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dcc7b1">7.18</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">6.90</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #c9d1a9">7.34</span> </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #90ee90">0.46</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #b7daa1">0.14</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #dcc7b1">-0.16</span> </td>
   <td style="text-align:right;"> <span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffb6c1">-0.44</span> </td>
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
