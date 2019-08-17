---
title: "Time Series Project: Analysis of PM2.5 in Beijing"
author: "David Josephs"
date: "2019-08-17"
output: 
  html_document:
    toc: TRUE
    number_sections: true
    toc_float:
      collapsed: true
      smooth_scroll: false
    theme: paper
    df_print: paged
    keep_md: TRUE
---


# Project Overview

A common time series problem that you see papers about all the time is predicting air quality from sensor data. In this project, I propose and implement a new model to do 3-day ahead forecasts of hourly air quality data in Beijing. The proposed model is an ensemble of 5 models, discussed below:

## Model 1: Regression with autocorrelated errors

In order to deal with the complex seasonality of hourly data (typically it exhibits a daily, weekly, and yearly seasonality), I used fourier expansions of the three possible seasonalities within the dataset (24 hours, 168 hours, and 61320 hours) as external regressors, with autocorrelated errors (using `auto.arima`).

## Model 2: TBATS

In another attempt to deal with the complex seasonality, I tried a model I had never heard of which is becoming popular for complex seasonality series: TBATS. It decomposes the data, represents the seasonality as a fourier expansion, autocorellates the errors, and estimates the trend with damping (it also implements a box-cox transformation if the algorithm deems that appropriate).

## Model 3: VAR

The first multivariate model, VAR was used with exogenous regressors (in this case, dummies for daily and nightly, as many papers I read had done the same and my EDA agreed with that).

## Model 4: NNETAR

Instead of using `nnfor`, which for my massive amount of data, was astoundingly slow (calculated for over 18 hours, no luck), `nnetar` from the `forecast` library was used. Nnetar is still a feed-forward neural network, much like the `mlp` function, but is not as customizable, but far more user friendly (and far faster). We are saving our user unfriendly model for the last one!

[reference](https://kourentzes.com/forecasting/2017/02/10/forecasting-time-series-with-neural-networks-in-r/)
[reference](http://freerangestats.info/blog/2016/11/06/forecastxgb)

## Model 5: LSTM

This is the pride and joy of this report. A RNN, specifically the Long Short-Term Memory (LSTM) was used. LSTMs are frequently used in forecasting hourly air quality, as they perform exceptionally well. This was the case here too. The model architecture is presented below


```
#> Model
#> Model: "sequential_7"
#> ___________________________________________________________________________
#> Layer (type)                     Output Shape                  Param #     
#> ===========================================================================
#> lstm_21 (LSTM)                   (None, None, 10)              680         
#> ___________________________________________________________________________
#> bidirectional_14 (Bidirectional) (None, None, 80)              16320       
#> ___________________________________________________________________________
#> bidirectional_15 (Bidirectional) (None, 160)                   103040      
#> ___________________________________________________________________________
#> dense_7 (Dense)                  (None, 1)                     161         
#> ===========================================================================
#> Total params: 120,201
#> Trainable params: 120,201
#> Non-trainable params: 0
#> ___________________________________________________________________________
```

We will now walk through the model architecture. First, the data goes into the first LSTM, which has 10 nodes in the hidden layer, activates using the `sigmoid` function, and then enters the first `bidirectional` LSTM. A bidirectional LSTM actual consistsof two duplicate LSTMs, the first of which data is read in proper time order, that is from beginning to end. In the second, the data is read in backwards. Then, the results of these are combined. What this does is it allows the LSTM to recognize not only subtle short term trends and patterns, but very deep and interesting long term patterns. 

The first bidirectional LSTM has 80 hidden nodes total, 40 in each direction. The second bidirectional LSTM has 160 nodes, 80 in each direction. 

After passing through the first three layers, the data is then pumped into a standard, feed forward neural network (single node) to revert it to a vector.

[Amazing primer on LSTMs](https://skymind.com/wiki/lstm#feedforward)

[Reference I used throughout the process](https://www.amazon.com/Deep-Learning-R-Francois-Chollet/dp/161729554X)

## A Note

Although this paper is relatively short, every model in this paper is a result of *a lot* of work. Overall, well over 2000 models (at least 1000 of those LSTMs, and countless hours of `aic5.wge`) were trained. If you are interested in exploring the process in greater depth, please see the `analysis/hourly/` directory of this repository (there is also some interesting stuff in `analysis/daily`, but those forecasts were fundamentally flawed)

## Ensembling Method: Stochastic Gradient Boosting

To ensemble these series together, I used gradient boosting. Gradient boosting is frequently used as an ensembling method in normal machine learning, and in recent literature it has been shown that it is an extremely powerful ensembling method for time series as well. [This paper](https://res.mdpi.com/data/data-04-00015/article_deploy/data-04-00015.pdf?filename=&attachment=1) not only references the validity of this approach, but also describes an even cooler, double ensemble method, where ensemble models are ensembled together to produce a very accurate model.

# Data Preprocessing

## Data Definition

Open to the public is data from 5 cities:

* Beijing

* Chengdu

* Guangzhou

* Shanghai

* Shenyang

Each of these datasets contains:

* Air quality measurements at multiple locations (including a US Embassy post)

* Humity

* Air pressure

* Temperature

* Precipitation

* Wind direction

* Wind Speed

Measured at every hour. For detailed info on the data quality and collection methods, please refer to [this publication](https://agupubs.onlinelibrary.wiley.com/doi/full/10.1002/2016JD024877)

## Data processing pipeline

Before we can start analyzing, we must load our data in. Below we see a series of funtions which loads a directory containing time series data, counts the NAs, imputes them with `spline` interpolation (from the imputeTS) package, and then puts them in a convenient hash table for fast access:


```r
library(functional)  # to compose the preprocessing pipeline
library(data.table)  # to read csvs
library(rlist)  # for list manipulations
library(pipeR)  # fast %>>% dumb %>>% pipes
library(imputeTS)  # to impute NAs
library(pander)  # so i can read the output
library(foreach)  # go fast
library(doParallel)  # go fast

import <- function(path) {
    # first we list the files in our path, in this case 'data/'
    files <- list.files(path)
    # then we search for csvs
    files <- files[grepl(files, pattern = ".csv")]
    # paste the csv names onto the filepath(data/whatever.csv)
    filepaths <- sapply(files, function(x) paste0(path, x))
    # read them in to a list
    out <- lapply(filepaths, fread)
    # Get rid of .csv in each filename
    fnames <- gsub(".csv", "", files)
    # get rid of the confusing numbers
    fnames <- gsub("[[:digit:]]+", "", fnames)
    # set the names of the list to be the clean filenams
    names(out) <- fnames
    out
}

naCount <- function(xs) {
    rapply(xs, function(x) sum(is.na(x)/length(x)), how = "list")
}
# load in the data now
datadir = "data/"
china <- import(datadir)
pander(naCount(china))
```



  * **beij**:

      * **No**: _0_
      * **year**: _0_
      * **month**: _0_
      * **day**: _0_
      * **hour**: _0_
      * **season**: _0_
      * **PM_Dongsi**: _0.5236_
      * **PM_Dongsihuan**: _0.61_
      * **PM_Nongzhanguan**: _0.5259_
      * **PM_US Post**: _0.04178_
      * **DEWP**: _0.00009509_
      * **HUMI**: _0.006447_
      * **PRES**: _0.006447_
      * **TEMP**: _0.00009509_
      * **cbwd**: _0.00009509_
      * **Iws**: _0.00009509_
      * **precipitation**: _0.009204_
      * **Iprec**: _0.009204_

  * **BeijingPM_**:

      * **No**: _0_
      * **year**: _0_
      * **month**: _0_
      * **day**: _0_
      * **hour**: _0_
      * **season**: _0_
      * **PM_Dongsi**: _0.5236_
      * **PM_Dongsihuan**: _0.61_
      * **PM_Nongzhanguan**: _0.5259_
      * **PM_US Post**: _0.04178_
      * **DEWP**: _0.00009509_
      * **HUMI**: _0.006447_
      * **PRES**: _0.006447_
      * **TEMP**: _0.00009509_
      * **cbwd**: _0.00009509_
      * **Iws**: _0.00009509_
      * **precipitation**: _0.009204_
      * **Iprec**: _0.009204_

  * **ChengduPM_**:

      * **No**: _0_
      * **year**: _0_
      * **month**: _0_
      * **day**: _0_
      * **hour**: _0_
      * **season**: _0_
      * **PM_Caotangsi**: _0.5356_
      * **PM_Shahepu**: _0.5323_
      * **PM_US Post**: _0.4504_
      * **DEWP**: _0.01006_
      * **HUMI**: _0.01017_
      * **PRES**: _0.009908_
      * **TEMP**: _0.01002_
      * **cbwd**: _0.009908_
      * **Iws**: _0.01014_
      * **precipitation**: _0.0562_
      * **Iprec**: _0.0562_

  * **GuangzhouPM_**:

      * **No**: _0_
      * **year**: _0_
      * **month**: _0_
      * **day**: _0_
      * **hour**: _0_
      * **season**: _0.00001902_
      * **PM_City Station**: _0.3848_
      * **PM_5th Middle School**: _0.5988_
      * **PM_US Post**: _0.3848_
      * **DEWP**: _0.00001902_
      * **HUMI**: _0.00001902_
      * **PRES**: _0.00001902_
      * **TEMP**: _0.00001902_
      * **cbwd**: _0.00001902_
      * **Iws**: _0.00001902_
      * **precipitation**: _0.00001902_
      * **Iprec**: _0.00001902_

  * **ShanghaiPM_**:

      * **No**: _0_
      * **year**: _0_
      * **month**: _0_
      * **day**: _0_
      * **hour**: _0_
      * **season**: _0_
      * **PM_Jingan**: _0.5303_
      * **PM_US Post**: _0.3527_
      * **PM_Xuhui**: _0.521_
      * **DEWP**: _0.0002472_
      * **HUMI**: _0.0002472_
      * **PRES**: _0.0005325_
      * **TEMP**: _0.0002472_
      * **cbwd**: _0.0002282_
      * **Iws**: _0.0002282_
      * **precipitation**: _0.07624_
      * **Iprec**: _0.07624_

  * **ShenyangPM_**:

      * **No**: _0_
      * **year**: _0_
      * **month**: _0_
      * **day**: _0_
      * **hour**: _0_
      * **season**: _0_
      * **PM_Taiyuanjie**: _0.5362_
      * **PM_US Post**: _0.5877_
      * **PM_Xiaoheyan**: _0.5317_
      * **DEWP**: _0.01316_
      * **HUMI**: _0.01293_
      * **PRES**: _0.01316_
      * **TEMP**: _0.01316_
      * **cbwd**: _0.01316_
      * **Iws**: _0.01316_
      * **precipitation**: _0.2427_
      * **Iprec**: _0.2427_


<!-- end of list -->

```r
# convert a vector to a time series object
tots <- function(v) {
    ts(v, frequency = 365 * 24)
}

# apply that function to single data frame, ignoring the categorical columns

totslist <- function(df) {
    # define the column names which we do not want to alter
    badlist <- c("No", "year", "month", "day", "hour", "season", "cbwd")
    # get the column names of the original data frame
    nms <- colnames(df)
    # coerce it into a list
    df <- as.list(df)
    # if the name of the item of the list is in our to not change list, do
    # nothing otherwise, convert to ts
    for (name in nms) {
        if (name %in% badlist) {
            df[[name]] <- df[[name]]
        } else {
            df[[name]] <- tots(df[[name]])
        }
    }
    # return the created list
    df  # its a list dont be tricked
}

# scale this function to a list of data frames (i intend eventually to
# predict air quality in each of these cities)

totsall <- function(xs) {
    lapply(xs, totslist)
}

# use the try catch pattern to impute the NAs of every time series in the
# list of data frames


imp_test <- function(v) {
    out <- try(na.interpolation(v, "spline"))
    ifelse(is.ts(out), return(out), return(v))
}


impute <- function(xs) {
    foreach(i = 1:length(xs), .final = function(x) {
        setNames(x, names(xs))
    }) %dopar% imp_test(xs[[i]])
}


impute_list <- function(xs) {
    lapply(xs, impute)
}

# convert anything to a hash table

to_hash <- function(xs) {
    list2env(xs, envir = NULL, hash = TRUE)
}

# combine everything into one function with functional's compostion tool
preprocess <- Compose(import, totsall, impute_list, to_hash)
```

# Univariate EDA
Now we are ready for some EDA, we would like to analyze for any seasonal patterns, as well as the standard wge sample plots


```r
library(ggthemes)
library(ggplot2)
library(cowplot)

# steal functions from other libraries
seasonplot <- forecast::ggseasonplot
subseriesplot <- forecast::ggsubseriesplot
lagplot <- forecast::gglagplot
sampplot <- tswge::plotts.sample.wge
# clean up errant data points
clean <- forecast::tsclean

# resample a time series to different sample rates note this is a function
# that generates functions


change_samples <- function(n) {
    function(xs) {
        out <- unname(tapply(xs, (seq_along(xs) - 1)%/%n, sum))
        out <- ts(out, frequency = (8760/n))
        out
    }
}

# daily and weekly sampling, monthly is 4 weeks
to_daily <- change_samples(24)
to_weekly <- change_samples(24 * 7)
to_monthly <- change_samples(24 * 7 * 4)
to_season <- change_samples(24 * (365/4))

# pipelining final cleaning and conversion, removing outlier and the couple
# of errant negative values (which do not make sense)
cleandays <- function(xs) {
    xs %>>% clean %>>% abs %>>% to_daily
}

cleanweeks <- function(xs) {
    xs %>>% clean %>>% abs %>>% to_weekly
}
cleanmonths <- function(xs) {
    xs %>>% clean %>>% abs %>>% to_monthly
}
cleanseas <- function(xs) {
    xs %>>% clean %>>% abs %>>% to_season
}

# resample the whole ts
resample <- function(xs) {
    day <- xs %>>% cleandays %>>% window(end = 6)
    week <- xs %>>% cleanweeks %>>% window(end = 6)
    month <- xs %>>% cleanmonths %>>% window(end = 6)
    seas <- xs %>>% cleanseas %>>% window(end = 6)
    list(day = day, week = week, month = month, season = seas)
}
```

## Sample plots

It is important to note that looking at the raw data, nothing useful is found, there is simply too much data.


```r
china <- preprocess(datadir)
```

```
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
```

```r
bj <- china$BeijingPM_$PM_US

# this takes a long time, and we are using it only once so we wil just load
# it like this bj <- resample(bj) save(bj, file = 'resampled.Rda')
load("pres/resampled.Rda")

dy <- sampplot(bj$day)
```

<img src="fig/unnamed-chunk-3-1.png" style="display: block; margin: auto;" />

The ACF appears to have a wandering behavior, but the parzen window, logic, and the the scatterplot tell us differently, it is likely there is a longer seasonality in this noisy data, lets look at weekly, monthly, and quarterly data to confirm this:


```r
wy <- sampplot(bj$w)
```

<img src="fig/unnamed-chunk-4-1.png" style="display: block; margin: auto;" />

```r
my <- sampplot(bj$m)
```

<img src="fig/unnamed-chunk-4-2.png" style="display: block; margin: auto;" />

```r
sy <- sampplot(bj$s)
```

<img src="fig/unnamed-chunk-4-3.png" style="display: block; margin: auto;" />

## Seasonal Plots

This shows clear complex seasonal patterns in the data


```r
sda <- seasonplot(bj$day) + scale_color_hc() + theme_hc() + ggtitle("Seasonal plot: Daily")
sdw <- bj$week %>>% seasonplot + theme_hc() + scale_color_hc() + ggtitle("Seasonal plot: Weekly")
sdm <- bj$month %>>% seasonplot + theme_hc() + scale_color_hc() + ggtitle("Seasonal plot: Monthly")
sds <- bj$seas %>>% seasonplot + theme_hc() + scale_color_hc() + ggtitle("Seasonal plot: Quarterly")
plot_grid(sda, sdw, sdm, sds, ncol = 2)
```

<img src="fig/unnamed-chunk-5-1.png" style="display: block; margin: auto;" />

This long term seasonality is because every winter, Beijing-dwellers use central heating, causing pollution to spike.

# Classical Univariate Analysis

## S3 methods and classes

First we define a generic method for scores, and define 3 scoring methods: ASE, MAPE, and number of points of the test set within the confidence interval. We also define the `wge` class and provide `autoplot` and `scores` methods:


```r
library(tswgewrapped)  # utility functions for tswge I wrote
scores <- function(obj) {
    UseMethod("scores")
}

ASE <- function(predicted, actual) {
    mean((actual - predicted)^2)
}

MAPE <- function(predicted, actual) {
    100 * mean(abs((actual - predicted)/actual))
}


checkConfint <- function(upper, lower, actual) {
    (actual < lower) | (actual > upper)
}

confScore <- function(upper, lower, actual) {
    rawScores <- ifelse(checkConfint(upper, lower, actual), 1, 0)
    return(sum(rawScores)/(length(actual)) * 100)
}

# plot a list of data frames, test and predictions
.testPredPlot <- function(xs) {
    p <- ggplot() + theme_hc() + scale_color_hc()
    doplot <- function(df) {
        p <<- p + geom_line(data = df, aes(x = t, y = ppm, color = type))
    }
    out <- lapply(xs, doplot)
    out[[2]]
}

# wge class
as.wge <- function(x) structure(x, class = "wge")

# scoring methods for wge objects
scores.wge <- function(xs) {
    mape <- MAPE(xs$f, testU)
    ase <- ASE(xs$f, testU)
    confs <- confScore(xs$ul, xs$ll, testU)
    c(ASE = ase, MAPE = mape, Conf.Score = confs)
}

# plotting methods for wge objects
autoplot.wge <- function(obj) {
    testdf <- data.frame(type = "actual", t = seq_along(testU), ppm = as.numeric(testU))
    preddf <- data.frame(type = "predicted", t = seq_along(obj$f), ppm = as.numeric(obj$f))
    confdf <- data.frame(upper = obj$ul, lower = obj$ll, t = seq_along(testU))
    dfl <- list(preddf, testdf)
    .testPredPlot(dfl) + geom_line(data = confdf, aes(x = t, y = lower, alpha = 0.2), 
        linetype = 3003) + geom_line(data = confdf, aes(x = t, y = upper, alpha = 0.2), 
        linetype = 3003) + guides(alpha = FALSE)
}
```

And then we do our analysis

## A note

ARUMA, ARIMA, airline, and ARMA models were tested. We are only presenting here the best ARUMA model we could make, as none of these models were as appropriate, in my eyes, as the model presented. Please see `analysis/hourly/classicalHourly.R for more`

## Analysis

```r
# data loading
bj <- preprocess("data/")
```

```
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
#> Error in na.interpolation(v, "spline") : Input x is not numeric
```

```r
bj <- bj$BeijingPM_ %>>% as.data.frame
uvar <- bj$PM_US.Post
uvar <- uvar %>>% forecast::tsclean() %>>% abs
trainU <- window(uvar, start = 3, end = c(6, 8760 - 48))
testU <- window(uvar, start = c(6, 8760 - 48))[-1]

# no smoke and mirrors, just a wrapper for phi.tr
difference
```

```
#> function (type, x, n) 
#> {
#>     szn_trans <- function(x, n) {
#>         artrans.wge(x, phi.tr = c(rep(0, n - 1), 1))
#>     }
#>     arima_trans <- function(x, n) {
#>         f <- artrans.wge(x, phi.tr = 1)
#>         if (n == 1) {
#>             res <- f
#>             return(res)
#>         }
#>         else {
#>             arima_trans(f, n - 1)
#>         }
#>     }
#>     if (is.character(enexpr(type)) == F) {
#>         type <- as.character(enexpr(type))
#>     }
#>     if (type %in% c("arima", "ARIMA", "Arima")) {
#>         return((arima_trans(x, n)))
#>     }
#>     if (type %in% c("ARUMA", "Aruma", "aruma", "Seasonal", "seasonal")) {
#>         szn_trans(x, n)
#>     }
#> }
#> <bytecode: 0x1f279030>
#> <environment: namespace:tswgewrapped>
```

```r
train7 <- difference(seasonal, trainU, 24 * 7)
```

<img src="fig/unnamed-chunk-7-1.png" style="display: block; margin: auto;" />

It does not appear to do much, but it is the most useful model we can make, we will model the rest of the data as a high order AR model. The data appears to exhibit heteroskedasticity, or still a large seasonal pattern. Could not find a way to make it stationary




```r
aic72 <- aic5.wge(train7, p = 0:8, q = 0:5, type = "aicc")
```


```r
aic72
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":[""],"name":["_rn_"],"type":[""],"align":["left"]},{"label":["   p"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["   q"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["       aicc"],"name":[3],"type":["dbl"],"align":["right"]}],"data":[{"1":"1","2":"4","3":"7.816267","_rn_":"11"},{"1":"5","2":"1","3":"7.816274","_rn_":"32"},{"1":"1","2":"5","3":"7.816286","_rn_":"12"},{"1":"3","2":"1","3":"7.816305","_rn_":"20"},{"1":"6","2":"0","3":"7.816310","_rn_":"37"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

Next we estimate the parameters:


```r
# a wrapper for basically est.ar and est.arma
estimate
```

```
#> function (xs, p, q = 0, type = "mle", ...) 
#> {
#>     if (q > 0) {
#>         return(est.arma.wge(xs, p, q, ...))
#>     }
#>     else {
#>         return(est.ar.wge(xs, p, type, ...))
#>     }
#> }
#> <bytecode: 0xa412548>
#> <environment: namespace:tswgewrapped>
```

```r
est7 <- estimate(train7, 6, 0)
```

```
#> 
#> Coefficients of Original polynomial:  
#> 1.1139 -0.1659 0.0165 -0.0136 0.0199 -0.0085 
#> 
#> Factor                 Roots                Abs Recip    System Freq 
#> 1-0.9560B              1.0460               0.9560       0.0000
#> 1-0.3675B+0.1548B^2    1.1872+-2.2473i      0.3934       0.1727
#> 1+0.5829B+0.1548B^2   -1.8834+-1.7073i      0.3934       0.3828
#> 1-0.3733B              2.6791               0.3733       0.0000
#>   
#> 
```

```r
phis <- est7$phi
data.frame(t(data.frame(phis, row.names = sapply(1:6, function(x) paste0("phi", 
    x)))), row.names = NULL)
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["phi1"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["phi2"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["phi3"],"name":[3],"type":["dbl"],"align":["right"]},{"label":["phi4"],"name":[4],"type":["dbl"],"align":["right"]},{"label":["phi5"],"name":[5],"type":["dbl"],"align":["right"]},{"label":["phi6"],"name":[6],"type":["dbl"],"align":["right"]}],"data":[{"1":"1.113926","2":"-0.1658656","3":"0.01647402","4":"-0.01362379","5":"0.01994103","6":"-0.008548799"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

Next, we try to see if we can generate similar time series to ours:


```r
# combines all the gen....wges
generate
```

```
#> function (type, ...) 
#> {
#>     phrase <- paste0("gen.", enexpr(type), ".wge")
#>     func <- parse_expr(phrase)
#>     eval(call2(func, ...))
#> }
#> <bytecode: 0xba1c1d0>
#> <environment: namespace:tswgewrapped>
```

```r
recreate <- generate(arma, length(trainU), phi = phis, plot = F, sn = 34541)
par(mfrow = c(1, 2))
plot(recreate, type = "l")
plot(train7, type = "l")
```

<img src="fig/unnamed-chunk-12-1.png" style="display: block; margin: auto;" />

Checking for model completeness


```r
acf(est7$res)
```

<img src="fig/unnamed-chunk-13-1.png" style="display: block; margin: auto;" />

ACF is not outrageous, but not brillant either


```r
# follows the recommend values of K
ljung_box
```

```
#> function (x, p, q, k_val = c(24, 48)) 
#> {
#>     ljung <- function(k) {
#>         hush(ljung.wge(x = x, p = p, q = q, K = k))
#>     }
#>     sapply(k_val, ljung)
#> }
#> <bytecode: 0xa5009d8>
#> <environment: namespace:tswgewrapped>
```

```r
ljung_box(est7$res, 6, 0)
```

```
#>            [,1]             [,2]            
#> test       "Ljung-Box test" "Ljung-Box test"
#> K          24               48              
#> chi.square 35.99452         103.2838        
#> df         18               42              
#> pval       0.007067426      0.0000004477291
```

Our model is not a very appropriate one, but it is the absolute best we can do with this dataset. Lets Forecast now:


```r
# wrapper for fore.....wge
fcst
```

```
#> function (type, ...) 
#> {
#>     phrase <- paste0("fore.", enexpr(type), ".wge")
#>     func <- parse_expr(phrase)
#>     eval(expr((!!func)(...)))
#> }
#> <bytecode: 0x90cb100>
#> <environment: namespace:tswgewrapped>
```

```r
seafor <- fcst(aruma, s = 24 * 7, phi = est7$phi, theta = 0, n.ahead = 72, x = trainU)
```

<img src="fig/unnamed-chunk-15-1.png" style="display: block; margin: auto;" />

Next lets assess the model:


```r
seaF <- as.wge(seafor)
autoplot(seaF)
```

<img src="fig/unnamed-chunk-16-1.png" style="display: block; margin: auto;" />

```r
as.data.frame(t(scores(seaF)))
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["ASE"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["MAPE"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["Conf.Score"],"name":[3],"type":["dbl"],"align":["right"]}],"data":[{"1":"18167.88","2":"353.7521","3":"18.05556"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

It did astoundingly well given the horrible residuals of our estimation. This will be an excellent benchmark.

# Autocorrelated errors regression

First, we will write some functions and make some S3 methods, as well as convert our data to MSTS:


```r
library(forecast)
# convert to MSTS
toMsts <- function(x) {
    msts(x, seasonal.periods = c(24, 24 * 7, 8760))
}

trainU <- toMsts(trainU)
testU <- toMsts(testU)

# casting and methods
as.fore <- function(x) structure(x, class = "fore")

autoplot.fore <- function(obj) {
    testdf <- data.frame(type = "actual", t = seq_along(testU), ppm = (testU))
    preddf <- data.frame(type = "predicted", t = seq_along(testU), ppm = (obj$mean[1:length(testU)]))
    
    confdf <- data.frame(upper = obj$upper[1:length(testU), 2], lower = obj$lower[1:length(testU), 
        2], t = seq_along(testU))
    dfl <- list(preddf, testdf)
    .testPredPlot(dfl) + geom_line(data = confdf, aes(x = t, y = lower, alpha = 0.2), 
        linetype = 3003) + geom_line(data = confdf, aes(x = t, y = upper, alpha = 0.2), 
        linetype = 3003) + guides(alpha = FALSE)
}

scores.fore <- function(obj) {
    mape <- MAPE(as.numeric(obj$mean[1:length(testU)]), as.numeric(testU))
    ase <- ASE(as.numeric(obj$mean[1:length(testU)]), as.numeric(testU))
    confs <- confScore(as.numeric(obj$upper[1:length(testU), 2]), as.numeric(obj$lower[1:length(testU), 
        2]), as.numeric(testU))
    c(ASE = ase, MAPE = mape, Conf.Score = confs)
}
```

## Fourier expansion

Here we can demonstrate the fourier expansion of the series, to show that it is appropriate.


```r
library(tidyverse)

exampleFourier <- fourier(testU, K = c(10, 20, 100))
library(dplyr)
shortEx <- exampleFourier %>% data.frame %>% dplyr::select(ends_with("24"))
midEx <- exampleFourier %>% data.frame %>% dplyr::select(ends_with("168"))
longEx <- exampleFourier %>% data.frame %>% dplyr::select(ends_with("8760"))
pls <- data.frame(rowSums(shortEx), rowSums(midEx), rowSums(longEx), testU)
par(mfrow = c(2, 2))
walk(pls, plot, type = "l")
```

<img src="fig/unnamed-chunk-18-1.png" style="display: block; margin: auto;" />

It is somewhat clear (at least to me) that some combination of these could represent our series really well. Next, lets fit the model and make a forecast ***WARNING*** This takes about 6 hours on my computer (with 12 cores). Do not run on your own. 

## Analysis



Lets first do a fourier expansion of our test set:


```r
trainExpand <- fourier(trainU, K = c(10, 20, 100))
```

Next lets run our model. Please note that we set seasonal = FALSE, as we are actually using our seasonal patterns as a regressor. We also set lambda = 0, which indicates we are going to do a Box-Cox transformation with lambda = 0, AKA a log transform. This keeps our forecasts from being negative ever.


```r
mseaDay <- auto.arima(trainU, xreg = trainExpand, seasonal = FALSE, lambda = 0)

# these indexes just work because of our split
mseasFor <- forecast(mseaDay, xreg = trainExpand[(nrow(trainExpand) - 71):(nrow(trainExpand)), 
    ])
```

We can check out the chosen unit roots with autoplot


```r
autoplot(mseaDay)
```

<img src="fig/unnamed-chunk-22-1.png" style="display: block; margin: auto;" />

Looks like we have AR(5) errors, with our multiple seasonalities. The roots are well within the unit circle, which is great

Lets take a look


```r
autoplot(mseasFor)
```

<img src="fig/unnamed-chunk-23-1.png" style="display: block; margin: auto;" />

This is not visible at all, lets make a longer forecast to see how it looks



```r
forecast(mseaDay, xreg = trainExpand[1:(8760/2), ]) %>% autoplot
```

<img src="fig/unnamed-chunk-24-1.png" style="display: block; margin: auto;" />

Wow, this looks really good. Lets assess our actual forecast now:


```r
autoplot(as.fore(mseasFor))
```

<img src="fig/unnamed-chunk-25-1.png" style="display: block; margin: auto;" />

```r
data.frame(t(scores(as.fore(mseasFor))))
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["ASE"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["MAPE"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["Conf.Score"],"name":[3],"type":["dbl"],"align":["right"]}],"data":[{"1":"18419.81","2":"117.2476","3":"27.77778"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>


It appears that we did not capture as much of the variance as ARUMA, but we captured a lot more the smooth parts of the time series, especialy that lower trough. The ASE is super comparable to the ARUMA model, but this one is far more appropriate.

# TBATS

[what is tbats](https://medium.com/intive-developers/forecasting-time-series-with-multiple-seasonalities-using-tbats-in-python-398a00ac0e8a)

## Analysis

Note we do not need any new s3 methods for tbats



```r
cores <- parallel::detectCores()
# > [1] 12
bjbats <- tbats(trainU, use.parallel = TRUE, num.cores = cores - 1)
```




```r
autoplot(bjbats)
```

<img src="fig/unnamed-chunk-28-1.png" style="display: block; margin: auto;" />


This is interesting too, so we see the I think trend that the algorithm smoothed out of the time series, as well as the seasonal fourier series it made. Lets check out that forecast:



```r
batF <- forecast(bjbats, h = 72)
autoplot(batF)
```

<img src="fig/unnamed-chunk-29-1.png" style="display: block; margin: auto;" />

Looks like we cant see our forecast at all. Lets try zooming in on this with our autoplot method:
 


```r
autoplot(as.fore(batF))
```

<img src="fig/unnamed-chunk-30-1.png" style="display: block; margin: auto;" />

This is an interesting forecast. Looks like it kind of smoothed out the trend of the time series as a whole. Lets look to assess how it did:



```r
scores(as.fore(batF)) %>% t %>% data.frame
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["ASE"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["MAPE"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["Conf.Score"],"name":[3],"type":["dbl"],"align":["right"]}],"data":[{"1":"22434.52","2":"255.8904","3":"18.05556"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

This also seems to be on par with the past few models we've made. It does the poorest job of capturing the variance of the data however (albeit with extremely nice confidence limits).

# Univariate model comparison



```r
batCast <- as.fore(batF)
arumaCast <- as.wge(seafor)
harmonCast <- as.fore(mseasFor)
autoplot(harmonCast)
```

<img src="fig/unnamed-chunk-32-1.png" style="display: block; margin: auto;" />

```r
seriesL <- list(tbats = batCast, aruma = arumaCast, harmonic = harmonCast)
vapply(seriesL, scores, FUN.VALUE = double(3)) %>% t %>% data.frame
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":[""],"name":["_rn_"],"type":[""],"align":["left"]},{"label":["ASE"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["MAPE"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["Conf.Score"],"name":[3],"type":["dbl"],"align":["right"]}],"data":[{"1":"22434.52","2":"255.8904","3":"18.05556","_rn_":"tbats"},{"1":"18167.88","2":"353.7521","3":"18.05556","_rn_":"aruma"},{"1":"18419.81","2":"117.2476","3":"27.77778","_rn_":"harmonic"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>


We can see the results of our performance very nicely in the table above. We see that the TBATS forecast performed worse than the the ARUMA and harmonic forecasts, but the prediction interval of the harmonic forecast was the least useful. The harmonic forecast was off by the least in general (MAPE), but did not capture the big movements (ASE), so it missed big a few times. In contrast, the ARUMA forecast was in general less close to the truth (MAPE), but captured the extreme cases much more nicely(ASE). 

To get closer to the truth, let us now examine the time series with other variables, to see if they can help us make an even better forecast:


# Multivariate Setup


```r
# first we grab the data columns we want, we only want PM_US.Post of the
# PM25 variables
bj <- bj %>% dplyr::select(-c(No, starts_with("PM_D"), starts_with("PM_N")))

# Next, we write a function which makes a multivariate time series data
# frame we cannot just do [,] indexing, as it will break the `ts` class
# instead we have a function which assigns the values we want to our global
# environment:

splitMvar <- function(start = 3, end = c(6, 8760 - 48)) {
    startInd <- length(window(bj$PM_US, end = start))
    endInd <- length(window(bj$PM_US, end = end))
    bjnots <- purrr::discard(bj, is.ts)
    bjnots <- data.frame(lapply(bjnots, as.factor))
    bjts <- purrr::keep(bj, is.ts)
    bjts$PM_US.Post <- uvar  # just because it is already cleaned
    notstrain <- bjnots[startInd:endInd, ]
    tsTrain <- data.frame(lapply(bjts, function(x) window(x, start = start, 
        end = end)))
    trainM <<- cbind(tsTrain, notstrain)
    notstest <- bjnots[(endInd + 1):nrow(bjnots), ]
    tsTest <- data.frame(lapply(bjts, function(x) window(x, start = c(6, 8760 - 
        47))))
    testM <<- cbind(tsTest, notstest)
}
splitMvar()
trainM
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
  </script>
</div>

```r
testM
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":[""],"name":["_rn_"],"type":[""],"align":["left"]},{"label":["PM_US.Post"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["DEWP"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["HUMI"],"name":[3],"type":["dbl"],"align":["right"]},{"label":["PRES"],"name":[4],"type":["dbl"],"align":["right"]},{"label":["TEMP"],"name":[5],"type":["dbl"],"align":["right"]},{"label":["Iws"],"name":[6],"type":["dbl"],"align":["right"]},{"label":["precipitation"],"name":[7],"type":["dbl"],"align":["right"]},{"label":["Iprec"],"name":[8],"type":["dbl"],"align":["right"]},{"label":["year"],"name":[9],"type":["fctr"],"align":["left"]},{"label":["month"],"name":[10],"type":["fctr"],"align":["left"]},{"label":["day"],"name":[11],"type":["fctr"],"align":["left"]},{"label":["hour"],"name":[12],"type":["fctr"],"align":["left"]},{"label":["season"],"name":[13],"type":["fctr"],"align":["left"]},{"label":["cbwd"],"name":[14],"type":["fctr"],"align":["left"]}],"data":[{"1":"294.0000","2":"-8","3":"79","4":"1032","5":"-5","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"0","13":"4","14":"NE","_rn_":"52513"},{"1":"301.0000","2":"-8","3":"85","4":"1032","5":"-6","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"1","13":"4","14":"SE","_rn_":"52514"},{"1":"315.0000","2":"-7","3":"85","4":"1032","5":"-5","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"2","13":"4","14":"cv","_rn_":"52515"},{"1":"326.0000","2":"-6","3":"85","4":"1032","5":"-4","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"3","13":"4","14":"NW","_rn_":"52516"},{"1":"316.0000","2":"-7","3":"85","4":"1031","5":"-5","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"4","13":"4","14":"NE","_rn_":"52517"},{"1":"337.0000","2":"-8","3":"92","4":"1030","5":"-7","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"5","13":"4","14":"NW","_rn_":"52518"},{"1":"301.0000","2":"-8","3":"92","4":"1030","5":"-7","6":"1.78","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"6","13":"4","14":"NW","_rn_":"52519"},{"1":"215.0000","2":"-8","3":"92","4":"1030","5":"-7","6":"0.45","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"7","13":"4","14":"cv","_rn_":"52520"},{"1":"201.0000","2":"-7","3":"92","4":"1030","5":"-6","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"8","13":"4","14":"NW","_rn_":"52521"},{"1":"179.0000","2":"-7","3":"85","4":"1030","5":"-5","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"9","13":"4","14":"NE","_rn_":"52522"},{"1":"165.0000","2":"-5","3":"79","4":"1030","5":"-2","6":"2.68","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"10","13":"4","14":"NE","_rn_":"52523"},{"1":"196.0000","2":"-7","3":"59","4":"1029","5":"0","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"11","13":"4","14":"cv","_rn_":"52524"},{"1":"236.0000","2":"-8","3":"50","4":"1028","5":"1","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"12","13":"4","14":"NW","_rn_":"52525"},{"1":"245.0000","2":"-7","3":"51","4":"1027","5":"2","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"13","13":"4","14":"cv","_rn_":"52526"},{"1":"264.0000","2":"-6","3":"51","4":"1026","5":"3","6":"1.34","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"14","13":"4","14":"cv","_rn_":"52527"},{"1":"318.0000","2":"-6","3":"51","4":"1026","5":"3","6":"2.23","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"15","13":"4","14":"cv","_rn_":"52528"},{"1":"360.0000","2":"-6","3":"51","4":"1026","5":"3","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"16","13":"4","14":"SE","_rn_":"52529"},{"1":"407.0000","2":"-6","3":"68","4":"1026","5":"-1","6":"1.78","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"17","13":"4","14":"SE","_rn_":"52530"},{"1":"447.0000","2":"-5","3":"74","4":"1027","5":"-1","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"18","13":"4","14":"cv","_rn_":"52531"},{"1":"494.1835","2":"-6","3":"79","4":"1027","5":"-3","6":"1.78","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"19","13":"4","14":"cv","_rn_":"52532"},{"1":"516.0732","2":"-6","3":"79","4":"1028","5":"-3","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"20","13":"4","14":"SE","_rn_":"52533"},{"1":"499.0000","2":"-6","3":"85","4":"1028","5":"-4","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"21","13":"4","14":"cv","_rn_":"52534"},{"1":"472.0000","2":"-4","3":"86","4":"1028","5":"-2","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"22","13":"4","14":"SE","_rn_":"52535"},{"1":"470.0000","2":"-7","3":"92","4":"1028","5":"-6","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"29","12":"23","13":"4","14":"cv","_rn_":"52536"},{"1":"462.1651","2":"-7","3":"92","4":"1028","5":"-6","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"0","13":"4","14":"NE","_rn_":"52537"},{"1":"418.0000","2":"-6","3":"92","4":"1028","5":"-5","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"1","13":"4","14":"NW","_rn_":"52538"},{"1":"460.0000","2":"-7","3":"92","4":"1028","5":"-6","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"2","13":"4","14":"NE","_rn_":"52539"},{"1":"331.0000","2":"-7","3":"92","4":"1028","5":"-6","6":"3.58","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"3","13":"4","14":"NE","_rn_":"52540"},{"1":"228.0000","2":"-6","3":"92","4":"1028","5":"-5","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"4","13":"4","14":"NW","_rn_":"52541"},{"1":"173.0000","2":"-6","3":"85","4":"1028","5":"-4","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"5","13":"4","14":"cv","_rn_":"52542"},{"1":"45.0000","2":"-6","3":"92","4":"1029","5":"-5","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"6","13":"4","14":"NW","_rn_":"52543"},{"1":"13.0000","2":"-5","3":"74","4":"1029","5":"-1","6":"5.81","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"7","13":"4","14":"NW","_rn_":"52544"},{"1":"10.0000","2":"-6","3":"85","4":"1030","5":"-4","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"8","13":"4","14":"NE","_rn_":"52545"},{"1":"8.0000","2":"-6","3":"63","4":"1031","5":"0","6":"3.13","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"9","13":"4","14":"NW","_rn_":"52546"},{"1":"12.0000","2":"-7","3":"47","4":"1031","5":"3","6":"7.15","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"10","13":"4","14":"NW","_rn_":"52547"},{"1":"13.0000","2":"-11","3":"30","4":"1031","5":"5","6":"16.09","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"11","13":"4","14":"NW","_rn_":"52548"},{"1":"9.0000","2":"-11","3":"28","4":"1031","5":"6","6":"23.24","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"12","13":"4","14":"NW","_rn_":"52549"},{"1":"14.0000","2":"-11","3":"26","4":"1030","5":"7","6":"31.29","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"13","13":"4","14":"NW","_rn_":"52550"},{"1":"14.0000","2":"-11","3":"28","4":"1030","5":"6","6":"38.44","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"14","13":"4","14":"NW","_rn_":"52551"},{"1":"11.0000","2":"-11","3":"28","4":"1030","5":"6","6":"46.49","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"15","13":"4","14":"NW","_rn_":"52552"},{"1":"8.0000","2":"-11","3":"30","4":"1030","5":"5","6":"53.64","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"16","13":"4","14":"NW","_rn_":"52553"},{"1":"6.0000","2":"-11","3":"32","4":"1031","5":"4","6":"57.66","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"17","13":"4","14":"NW","_rn_":"52554"},{"1":"15.0000","2":"-11","3":"34","4":"1031","5":"3","6":"61.68","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"18","13":"4","14":"NW","_rn_":"52555"},{"1":"17.0000","2":"-11","3":"46","4":"1032","5":"-1","6":"63.47","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"19","13":"4","14":"NW","_rn_":"52556"},{"1":"20.0000","2":"-10","3":"54","4":"1033","5":"-2","6":"66.60","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"20","13":"4","14":"NW","_rn_":"52557"},{"1":"22.0000","2":"-10","3":"50","4":"1034","5":"-1","6":"70.62","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"21","13":"4","14":"NW","_rn_":"52558"},{"1":"33.0000","2":"-11","3":"58","4":"1034","5":"-4","6":"73.75","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"22","13":"4","14":"NW","_rn_":"52559"},{"1":"26.0000","2":"-11","3":"53","4":"1034","5":"-3","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"30","12":"23","13":"4","14":"NE","_rn_":"52560"},{"1":"28.0000","2":"-11","3":"62","4":"1034","5":"-5","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"0","13":"4","14":"NW","_rn_":"52561"},{"1":"27.0000","2":"-9","3":"73","4":"1034","5":"-5","6":"3.58","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"1","13":"4","14":"NW","_rn_":"52562"},{"1":"24.0000","2":"-11","3":"73","4":"1034","5":"-7","6":"5.37","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"2","13":"4","14":"NW","_rn_":"52563"},{"1":"23.0000","2":"-11","3":"67","4":"1034","5":"-6","6":"8.50","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"3","13":"4","14":"NW","_rn_":"52564"},{"1":"19.0000","2":"-11","3":"73","4":"1034","5":"-7","6":"10.29","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"4","13":"4","14":"NW","_rn_":"52565"},{"1":"14.0000","2":"-11","3":"73","4":"1034","5":"-7","6":"12.08","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"5","13":"4","14":"NW","_rn_":"52566"},{"1":"19.0000","2":"-12","3":"72","4":"1034","5":"-8","6":"15.21","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"6","13":"4","14":"NW","_rn_":"52567"},{"1":"25.0000","2":"-11","3":"73","4":"1034","5":"-7","6":"18.34","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"7","13":"4","14":"NW","_rn_":"52568"},{"1":"22.0000","2":"-11","3":"67","4":"1034","5":"-6","6":"20.13","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"8","13":"4","14":"NW","_rn_":"52569"},{"1":"25.0000","2":"-8","3":"68","4":"1035","5":"-3","6":"23.26","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"9","13":"4","14":"NW","_rn_":"52570"},{"1":"29.0000","2":"-9","3":"50","4":"1035","5":"0","6":"26.39","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"10","13":"4","14":"NW","_rn_":"52571"},{"1":"31.0000","2":"-10","3":"43","4":"1035","5":"1","6":"28.18","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"11","13":"4","14":"NW","_rn_":"52572"},{"1":"40.0000","2":"-10","3":"37","4":"1033","5":"3","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"12","13":"4","14":"cv","_rn_":"52573"},{"1":"43.0000","2":"-11","3":"34","4":"1032","5":"3","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"13","13":"4","14":"NW","_rn_":"52574"},{"1":"48.0000","2":"-10","3":"35","4":"1031","5":"4","6":"1.79","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"14","13":"4","14":"SE","_rn_":"52575"},{"1":"58.0000","2":"-11","3":"32","4":"1031","5":"4","6":"3.58","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"15","13":"4","14":"SE","_rn_":"52576"},{"1":"69.0000","2":"-10","3":"37","4":"1031","5":"3","6":"4.47","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"16","13":"4","14":"SE","_rn_":"52577"},{"1":"91.0000","2":"-10","3":"43","4":"1030","5":"1","6":"5.36","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"17","13":"4","14":"SE","_rn_":"52578"},{"1":"114.0000","2":"-10","3":"58","4":"1030","5":"-3","6":"6.25","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"18","13":"4","14":"SE","_rn_":"52579"},{"1":"133.0000","2":"-8","3":"68","4":"1031","5":"-3","6":"7.14","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"19","13":"4","14":"SE","_rn_":"52580"},{"1":"169.0000","2":"-8","3":"63","4":"1030","5":"-2","6":"8.03","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"20","13":"4","14":"SE","_rn_":"52581"},{"1":"203.0000","2":"-10","3":"73","4":"1030","5":"-6","6":"0.89","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"21","13":"4","14":"NE","_rn_":"52582"},{"1":"212.0000","2":"-10","3":"73","4":"1030","5":"-6","6":"1.78","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"22","13":"4","14":"NE","_rn_":"52583"},{"1":"235.0000","2":"-9","3":"79","4":"1029","5":"-6","6":"2.67","7":"0","8":"0","9":"2015","10":"12","11":"31","12":"23","13":"4","14":"NE","_rn_":"52584"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

```r
# double check

sum(abs(as.numeric(testM$PM_US) - as.numeric(testU)))
```

```
#> [1] 0
```

```r
nrow(testM) - length(testU)
```

```
#> [1] 0
```

```r
sum(abs(trainM$PM_US - trainU))
```

```
#> [1] 0
```

```r
nrow(trainM) - length(trainU)
```

```
#> [1] 0
```

# Multivariate EDA



```r
library(tidyverse)
plotAllTs <- function(df) {
    tsdf <- keep(df, is.ts)
    lapply(tsdf, function(x) autoplot(x) + theme_hc())
}
plot_grid(plotlist = plotAllTs(trainM), labels = names(keep(trainM, is.ts)))
```

<img src="fig/unnamed-chunk-34-1.png" style="display: block; margin: auto;" />


We already learned some interesting stuff. First of all, precipitation does not seem to be in any way similar to our dataset. So, we probably wont use it as a predictor, as it also does not make logical sense. Of all the time seres here, humidity appears to have a similar trend pattern to our target. Pressure looks similar but a bit off, so at a high lag maybe it is interesting. Dewpoint has the wrong period, and is a function of temperature and humidity, so most likely it does not provide us with any interesting information.


## Analysis of Categorical Variables:

Lets see how each of the categorical variables affect our air content:


```r
catTable <- function(cat) {
    trainM %>% arrange(!!sym(cat)) %>% group_by(!!sym(cat)) %>% summarise(meanPPM = mean(PM_US.Post))
}

map(names(discard(trainM, is.ts)), catTable) %>% pander
```



  *

    ----------------
     year   meanPPM
    ------ ---------
     2012    89.91

     2013    100.6

     2014    97.36

     2015    81.27
    ----------------

  *

    -----------------
     month   meanPPM
    ------- ---------
       1      130.4

       2      118.9

       3      104.2

       4      81.41

       5      76.97

       6      80.25

       7      72.96

       8      63.12

       9      66.98

      10      103.3

      11      101.8

      12      108.8
    -----------------

  *

    ---------------
     day   meanPPM
    ----- ---------
      1     87.95

      2     76.97

      3     72.52

      4     70.35

      5     81.04

      6     96.23

      7     95.08

      8     95.85

      9     92.02

     10     77.49

     11     76.2

     12     73.52

     13     94.75

     14     88.16

     15     104.5

     16     99.5

     17     96.24

     18     93.28

     19      103

     20     99.4

     21     102.9

     22     99.53

     23     106.7

     24     96.15

     25     101.9

     26     104.2

     27     96.54

     28     106.6

     29     94.39

     30     86.17

     31     91.18
    ---------------

  *

    ----------------
     hour   meanPPM
    ------ ---------
      0      103.6

      1      103.1

      2      101.4

      3      98.01

      4      94.18

      5      91.81

      6      90.26

      7      90.15

      8      88.86

      9      87.51

      10     86.73

      11     84.65

      12     83.38

      13     81.49

      14     80.67

      15     80.91

      16     82.57

      17     86.23

      18     91.22

      19     96.89

      20     100.9

      21     102.8

      22     103.8

      23     104.1
    ----------------

  *

    ------------------
     season   meanPPM
    -------- ---------
       1       87.6

       2       72.02

       3       90.83

       4       119.5
    ------------------

  *

    ----------------
     cbwd   meanPPM
    ------ ---------
      cv     118.4

      NE     82.67

      NW     62.05

      SE     104.6

      NA     75.6
    ----------------


<!-- end of list -->

There are a few variables which stand out, namely hour, season, and cbwd (wind direction). Hour appears to be a daily and nigthly pattern, where at night, the average PPM goes up. Lets create a new categorical variable for daytime nighttime:


```r
library(forcats)
hoursToDayNight <- function(df) {
    df[["hour"]] %>% fct_collapse(night = c(as.character(18:23), as.character(0:5)), 
        day = as.character(6:17))
}

trainM$dayNight <- hoursToDayNight(trainM)
testM$dayNight <- hoursToDayNight(testM)
catTable("dayNight")
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["dayNight"],"name":[1],"type":["fctr"],"align":["left"]},{"label":["meanPPM"],"name":[2],"type":["dbl"],"align":["right"]}],"data":[{"1":"night","2":"99.31249"},{"1":"day","2":"85.28434"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

A noticeable change between day and night (you could say a day and night difference). This will be an absolutely useful factor. As for wind direction, we have a lot of NAs, and we only have 4 wind directions, so that may not be so useful.

## Another look at numeric variables



```r
plotVsResponse <- function(x) {
    plot(trainM$PM_US.Post ~ trainM[[x]], xlab = x)
    lw1 <- loess(trainM$PM_US.Post ~ trainM[[x]])
    j <- order(trainM[[x]])
    lines(trainM[[x]][j], lw1$fitted[j], col = "red", lwd = 3)
}
trainM2 <- trainM %>% select(-contains("prec"))  # precipitation is all 0
trainM2 %>% keep(is.numeric) %>% names %>% walk(plotVsResponse)
```

<img src="fig/unnamed-chunk-37-1.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-37-2.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-37-3.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-37-4.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-37-5.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-37-6.png" style="display: block; margin: auto;" />

All of these variables appear to have some sort of relationship with the air quality, so we will have to investigate further. 

## CCF analysis

Lets look at the cross correlation between all the useful variables next:


```r
ppm <- trainM$PM_US.Post
ccfPlot <- function(x) {
    ccf(ppm, trainM[[x]], main = x)
}
trainM2 %>% keep(is.numeric) %>% names %>% walk(ccfPlot)
```

<img src="fig/unnamed-chunk-38-1.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-38-2.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-38-3.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-38-4.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-38-5.png" style="display: block; margin: auto;" /><img src="fig/unnamed-chunk-38-6.png" style="display: block; margin: auto;" />

It appears we have a very complex correlation structure. It is important to note that the cross correlation of dewpoint does look like some sort of combination of humidity and temperature, which makes physical sense. Now that we have completed our EDA, lets go ahead and get to doing analysis.

# Multivariate data prep

Lets update our split funciton to include encoded day night factors, and we can get rid of the dummies, which werent that helpful as seasonality is already encoded in the `ts` class (which all of our models that care use)


```r
hoursToDayNight <- function(df) {
    df[["hour"]] %>% fct_collapse(night = c(as.character(18:23), as.character(0:5)), 
        day = as.character(6:17)) %>% as.numeric %>% -1
}

splitMvar <- function(start = 3, end = c(6, 8760 - 48)) {
    startInd <- length(window(bj$PM_US, end = start))
    endInd <- length(window(bj$PM_US, end = end))
    bjnots <- purrr::discard(bj, is.ts)
    bjnots <- data.frame(lapply(bjnots, as.factor))
    bjnots$dayNight <- hoursToDayNight(bjnots)
    bjts <- purrr::keep(bj, is.ts)
    bjts$PM_US.Post <- uvar  # just because it is already cleaned
    notstrain <- bjnots[startInd:endInd, ]
    tsTrain <- data.frame(lapply(bjts, function(x) window(x, start = start, 
        end = end)))
    trainM <<- cbind(tsTrain, notstrain)
    notstest <- bjnots[(endInd + 1):nrow(bjnots), ]
    tsTest <- data.frame(lapply(bjts, function(x) window(x, start = c(6, 8760 - 
        47))))
    testM <<- cbind(tsTest, notstest)
    trainM[c(7:14)] <<- NULL
    testM[c(7:14)] <<- NULL
}
splitMvar()
head(trainM)
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":[""],"name":["_rn_"],"type":[""],"align":["left"]},{"label":["PM_US.Post"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["DEWP"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["HUMI"],"name":[3],"type":["dbl"],"align":["right"]},{"label":["PRES"],"name":[4],"type":["dbl"],"align":["right"]},{"label":["TEMP"],"name":[5],"type":["dbl"],"align":["right"]},{"label":["Iws"],"name":[6],"type":["dbl"],"align":["right"]},{"label":["dayNight"],"name":[7],"type":["dbl"],"align":["right"]}],"data":[{"1":"303","2":"-12","3":"72","4":"1030","5":"-8","6":"0.89","7":"0","_rn_":"17521"},{"1":"215","2":"-13","3":"78","4":"1031","5":"-10","6":"1.79","7":"0","_rn_":"17522"},{"1":"222","2":"-13","3":"72","4":"1032","5":"-9","6":"3.58","7":"0","_rn_":"17523"},{"1":"85","2":"-13","3":"72","4":"1033","5":"-9","6":"6.71","7":"0","_rn_":"17524"},{"1":"38","2":"-13","3":"49","4":"1033","5":"-4","6":"4.92","7":"0","_rn_":"17525"},{"1":"23","2":"-14","3":"45","4":"1034","5":"-4","6":"10.73","7":"0","_rn_":"17526"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

# VAR

Our first multivariate technique is going to be VAR. One of the interesting properties of VAR is what it means about the data. VAR is telling us that our time series are *endogenous*, that is to say that they all affect each other. Now with our weather patterns, this makes sense. Weather is a complex, dynamic system of complex feedback cycles, so it is natural that all the weather variables are endogenous to each other. However; let us discuss how our air particulate content relates to the weather, to determine weather a VAR model is physically appropriate

## Discussion of Appropriateness

In this section we will discuss how each of our weather patterns interplay (or dont) with our air quality.

### Humidity

#### Effect of Humidity on Air Quality

Humidity has a strong effect on air quality. On a humid day, water in the air will "stick" to air particulates. This weighs them down and makes them much larger, causing them to stick around for longer. This will cause the air particulates to hang around in one place. 

It is also important to note that this has an effect on temperature. When water sticks to air pollution, energy from the sun is reflected off of the water, imparting a bit of energy to the water and spreading the sunlight some more. This adds to the haze appearance when there is a lot of pollution. Over time, due to water's high specific heat, should cause the air to heat up, as the small amount of extra energy stored in the water attached to the pollution will be transferred quickly to the air. (Note this should also have an affect on pressure slowly, because $P \propto T$).

#### Effect of Air Quality on Humidity

Surprisingly, air quality also has a positive feedback with humidity. This is due to the "sticking" effect we discussed earlier. Heavy air particulate matter with water attached causes other air particles to stick around too. You can actually prove this with an experiment.


Supplies

* An aerosol, such as hairspray

* A jar with a lid

* Ice

* Warm water

Take the warm water and put it in your jar. Put the lid on upside down with ice in it, for about 30 seconds (this causes the evaporated water in the air to condense a bit, raising the humidity). 

Now, lift the ice lid up, and spray a tiny bit of your aerosol in the jar and quickly replave the ice lid. 

Then, watch a cloud form inside the jar.

You can do this experiment with any air particulate, for example smoke from a match also works nicely.

It is safe to say that a bidirectional VAR forecast of humidity is appropriate.



### Temperature

#### How Temperature Relates to Surface Air Quality

Temperature has a clear effect on air quality. Not only does cold air cause air particulates to stick around more (slower moving), but also more condensed water particles in the air, which we already know about.

Similarly, in the winter, tropospheric temperature inversions can occur when the stratosphere (think, up high) gets heated more than the troposphere (we live here). Normally, the air naturally convects because it is hotter in the troposphere than the stratosphere, but in the winter, with our long nights, the opposite can occur. This is a temerature inversion. This causes the convection to stop, and the air to remain trapped there. This traps the pollutants in the same place, and causes them to accumilate

#### Effect of air quality on temperature

The air quality + humidity should have a slight effect on temperature, but global weather patterns are more powerful

#### Discussion

It is appropriate to say that temperature affects air quality, but only slightly appropriate to say the opposite

### Dewpoint

Dewppoint is simply a function of humidity and temperature, so this variable will not be included in our model, as it is simply overfitting.

### Wind speed

This is a complex relationship, but basically it does have some sort of complex relationship with air pollution. When it is windy, pollutants can be moved thousands of miles, while when it is still, they can accumulate. So it depends, but it does have an effect. Heavy particulte matter may also slow the wind down, but it is unclear and unproven. Overall though, it is somewhat appropriate to do this bidirectional forecast.

### Pressure

Given the complex relationship between pressure, temperature, humidity, and wind speed, which are all somewhat related to air quality in interesting ways, it is safe to include pressure in our VAR models as well

### Precipitation

No matter how much i think, I cannot say that you can predict surface air quality with precipitation or vice versa, so this is likely inappropriate

## Analysis

First, we break our sets into exogenous and endogenous sets, as well as write a dummy variable generator for our prediction


```r
vartrain <- trainM %>% dplyr::select(-c(dayNight))
varexo <- matrix(trainM$dayNight, dimnames = list(NULL, "dayNight"))

makeExo <- function(n) {
    day <- c(rep(0, 5), rep(1, 12), rep(0, 7))
    return(matrix(rep(day, n), dimnames = list(NULL, "dayNight")))
}
```

Next, lets write our standard s3 methods for VAR


```r
as.var <- function(x) structure(x, class = "var")
autoplot.var <- function(obj) {
    us <- obj$fcst$PM_US.Post
    testdf <- data.frame(type = "actual", t = seq_along(testM$PM_US.Post), ppm = as.numeric(testM$PM_US.Post))
    preddf <- data.frame(type = "predicted", t = seq_along(testM$PM_US.Post), 
        ppm = (obj$fcst$PM_US.Post[, 1]))
    dfl <- list(testdf, preddf)
    confdf <- data.frame(t = seq_along(testM$PM_US.Post), upper = us[, 3], lower = us[, 
        2])
    .testPredPlot(dfl) + geom_line(data = confdf, aes(x = t, y = lower, alpha = 0.2), 
        linetype = 3003) + geom_line(data = confdf, aes(x = t, y = upper, alpha = 0.2), 
        linetype = 3003) + guides(alpha = FALSE)
}
scores.var <- function(obj) {
    mape <- MAPE(obj$fcst$PM[, 1], testM$PM_US)
    ase <- ASE(obj$fcst$PM[, 1], testM$PM_US)
    us <- obj$fcst$PM_US.Post
    conf <- confScore(upper = us[, 3], lower = us[, 2], testM$PM_US)
    c(ASE = ase, MAPE = mape, Conf.Score = conf)
}
```


### Linear Modeling

An important part in choosing variables for VAR is to model our data in the same fassion as a linear model (for reference on how to choose variables for VAR, please refer to [this link](https://cran.r-project.org/web/packages/vars/vignettes/vars.pdf)).


```r
summary(lm(trainM))
```

```
#> 
#> Call:
#> lm(formula = trainM)
#> 
#> Residuals:
#>     Min      1Q  Median      3Q     Max 
#> -175.56  -50.10  -13.17   32.25  378.96 
#> 
#> Coefficients:
#>                Estimate  Std. Error t value             Pr(>|t|)    
#> (Intercept) 1312.562192   79.364221  16.538 < 0.0000000000000002 ***
#> DEWP           0.573997    0.171460   3.348             0.000816 ***
#> HUMI           0.971564    0.054544  17.812 < 0.0000000000000002 ***
#> PRES          -1.212078    0.077217 -15.697 < 0.0000000000000002 ***
#> TEMP          -3.171640    0.169294 -18.734 < 0.0000000000000002 ***
#> Iws           -0.281490    0.009344 -30.125 < 0.0000000000000002 ***
#> dayNight       8.395574    0.868097   9.671 < 0.0000000000000002 ***
#> ---
#> Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
#> 
#> Residual standard error: 76.63 on 34985 degrees of freedom
#> Multiple R-squared:  0.2303,	Adjusted R-squared:  0.2302 
#> F-statistic:  1745 on 6 and 34985 DF,  p-value: < 0.00000000000000022
```

We see here that while all our predictors are important, dewpoint isnt that important. So, lets remove that from our function, so that next time we dont even look at it, and lets remove it from the sets we are about to use:


```r
splitMvar <- function(start = 3, end = c(6, 8760 - 48)) {
    startInd <- length(window(bj$PM_US, end = start))
    endInd <- length(window(bj$PM_US, end = end))
    bjnots <- purrr::discard(bj, is.ts)
    bjnots <- data.frame(lapply(bjnots, as.factor))
    bjnots$dayNight <- hoursToDayNight(bjnots)
    bjts <- purrr::keep(bj, is.ts)
    bjts$PM_US.Post <- uvar  # just because it is already cleaned
    notstrain <- bjnots[startInd:endInd, ]
    tsTrain <- data.frame(lapply(bjts, function(x) window(x, start = start, 
        end = end)))
    trainM <<- cbind(tsTrain, notstrain)
    notstest <- bjnots[(endInd + 1):nrow(bjnots), ]
    tsTest <- data.frame(lapply(bjts, function(x) window(x, start = c(6, 8760 - 
        47))))
    testM <<- cbind(tsTest, notstest)
    trainM[c(7:14)] <<- NULL
    testM[c(7:14)] <<- NULL
    trainM$DEWP <<- NULL
    testM$DEWP <<- NULL
}
splitMvar()
vartrain$DEWP <- NULL
```


## Selecting the order

Next, we must select the order of our VAR forecast:




```r
ord <- VARselect(vartrain, lag.max = 100, exogen = varexo)
```

Lets check it out:


```r
data.frame(ord$selection)
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":[""],"name":["_rn_"],"type":[""],"align":["left"]},{"label":["ord.selection"],"name":[1],"type":["int"],"align":["right"]}],"data":[{"1":"78","_rn_":"AIC(n)"},{"1":"30","_rn_":"HQ(n)"},{"1":"26","_rn_":"SC(n)"},{"1":"78","_rn_":"FPE(n)"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

Looks like we have three candidate values. First, the AIC value of 78 is clearly too big, that would be overfitting. The BIC/SC and HQ values both appear reasonable. Through trial and error (see analysis/daily/VAR.R), it was found that the HQ order of 30 performed better. This was all validated through a massive grid search of VAR forecasts in `analysis/hourly/VAR.R`, which will not be discussed here

### Prediction

Next we get to finally make our model:


```r
Var <- VAR(vartrain, p = 30, exogen = varexo, type = "both")
```

Lets confirm our exogenous variable maker works before we use it:


```r
ht <- function(d, m = 5, n = m) {
    list(HEAD = head(d, m), TAIL = tail(d, n))
}
ht(makeExo(1), 10)
```

```
#> $HEAD
#>       dayNight
#>  [1,]        0
#>  [2,]        0
#>  [3,]        0
#>  [4,]        0
#>  [5,]        0
#>  [6,]        1
#>  [7,]        1
#>  [8,]        1
#>  [9,]        1
#> [10,]        1
#> 
#> $TAIL
#>       dayNight
#> [15,]        1
#> [16,]        1
#> [17,]        1
#> [18,]        0
#> [19,]        0
#> [20,]        0
#> [21,]        0
#> [22,]        0
#> [23,]        0
#> [24,]        0
```

That'll do


```r
varpred <- predict(Var, n.ahead = 72, dumvar = makeExo(3))
plot(varpred)
```

<img src="fig/unnamed-chunk-49-1.png" style="display: block; margin: auto;" />

We can barely see the prediction, lets zoom in a bit:


```r
varp <- as.var(varpred)
autoplot(varp)
```

<img src="fig/unnamed-chunk-50-1.png" style="display: block; margin: auto;" />
And the scores:


```r
data.frame(t(scores(varp)))
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["ASE"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["MAPE"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["Conf.Score"],"name":[3],"type":["dbl"],"align":["right"]}],"data":[{"1":"19661.58","2":"350.2365","3":"22.22222"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

This is pretty good, it performed slightly worse than the aruma and harmonic methods, but still much better than the TBATS forecast. It is about the same as the harmonic forecast, but with less variance. Lets move on to the simple `nnetar` model now

# NNETAR

## S3 Methods


```r
as.nfor <- function(x) structure(x, class = "nfor")

scores.nfor <- function(obj) {
    mape <- MAPE(obj$mean, testM[[1]])
    ase <- ASE(obj$mean, testM[[1]])
    confs <- confScore(upper = obj$upper[, 2], lower = obj$lower[, 2], testM[[1]])
    c(ASE = ase, MAPE = mape, Conf.Score = confs)
}
autoplot.nfor <- function(obj) {
    testdf <- data.frame(type = "actual", t = seq_along(testM[[1]]), ppm = as.numeric(testM[[1]]))
    preddf <- data.frame(type = "predicted", t = seq_along(testM[[1]]), ppm = as.numeric(obj$mean))
    confdf <- data.frame(t = seq_along(testM[[1]]), upper = obj$upper[, 2], 
        lower = obj$lower[, 2])
    dfl <- list(testdf, preddf)
    .testPredPlot(dfl) + geom_line(data = confdf, aes(x = t, y = lower, alpha = 0.2), 
        linetype = 3003) + geom_line(data = confdf, aes(x = t, y = upper, alpha = 0.2), 
        linetype = 3003) + guides(alpha = FALSE)
}
```

## Analysis

This is fast and simple. One note, we want to scale our inputs before feeding them in to a neural network, if we have multiple features with different magnitudes. Otherwise, the model will weight the big attributes higher than the little attributes, imbalancing the model. We also again set lambda = 0:





```r
nnet <- nnetar(y = trainM[[1]], xreg = trainM[-1], scale.inputs = TRUE, repeats = 100, 
    lambda = 0)
```

Lets check out the AR order it chose, as well as the number of lags:


```r
nnet$p
```

```
#> [1] 25
```

```r
nnet$lags
```

```
#>  [1]    1    2    3    4    5    6    7    8    9   10   11   12   13   14
#> [15]   15   16   17   18   19   20   21   22   23   24   25 8760
```



***NOTE*** This will take a few hours. To calculate the confidence interval, it runs the neural net 1000 times.


```r
nnetfor <- forecast(nnet, xreg = trainM[(nrow(trainM) - 71):nrow(trainM), -1], 
    PI = TRUE)
```

Lets check out that forecast:


```r
autoplot(nnetfor)
```

<img src="fig/unnamed-chunk-57-1.png" style="display: block; margin: auto;" />

Lets view the forecast with a higher granularity and check out the scores


```r
autoplot(as.nfor(nnetfor))
```

<img src="fig/unnamed-chunk-58-1.png" style="display: block; margin: auto;" />

```r
scores(as.nfor(nnetfor)) %>% t %>% data.frame
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["ASE"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["MAPE"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["Conf.Score"],"name":[3],"type":["dbl"],"align":["right"]}],"data":[{"1":"19847.95","2":"214.3944","3":"8.333333"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

This model is pretty much par for the course. All of them seem to have trouble getting that big peak, other than ARUMA. Lets see if deep learning is the answer to our troubles:

# LSTM

## Data Prep
First, we must prepare our data:



As LSTMs rely on the sigmoid function, which is between zero and one, we must prepare our data accordingly (aka center and scale it). It also has to be in matrix form (for now). We will save our scale values for later as well. In the following chunk, we prepare our dataset, our test values, as well as a validation set for use in the next section.


```r
library(keras)

Ktrain <- data.matrix(trainM)
Ktest <- data.matrix(testM)
dat <- rbind(Ktrain, Ktest)
str(dat)
```

```
#>  num [1:35064, 1:6] 303 215 222 85 38 23 19 14 16 21 ...
#>  - attr(*, "dimnames")=List of 2
#>   ..$ : chr [1:35064] "17521" "17522" "17523" "17524" ...
#>   ..$ : chr [1:6] "PM_US.Post" "HUMI" "PRES" "TEMP" ...
```

```r
nrow(dat)
```

```
#> [1] 35064
```

```r
x <- nrow(dat)
mn <- apply(dat, 2, mean)
std <- apply(dat, 2, sd)
# train test split of our dataset
maxt <- floor(nrow(dat) * 4/5)
maxv <- floor(nrow(dat))
val <- dat[(maxv - 72 * 2):(maxv - 72), ]
val <- scale(val, center = apply(val, 2, mean), scale = apply(val, 2, sd))
valScale <- attr(val, "scaled:scale")[1]
valCent <- attr(val, "scaled:center")[1]
val <- array(val, c(nrow(val), 10, 6))
dat <- scale(dat, center = mn, scale = std)
test2 <- scale(Ktest, center = apply(Ktest, 2, mean), scale = apply(Ktest, 2, 
    sd))
test3 <- array(test2, c(nrow(test2), 10, 6))
```

## Sampling
Next, we want to define a data generator for the LSTM. When we train it, we dont want to feed it our entire dataset over and over again, that will lead to overfitting pretty quickly. Instead, we are going to do `rolling origin resamples`. How this will work is we will write a generator function, which pulls from our dataset a certain number of observations, samples a certain number of observations from that, and compares observations a certain number of steps ahead. It will do this a certain number of times per epoch. The values are defined as follows:

* lookback: How many timesteps back we sample

* delay: How far forward our targets are

* steps: The number of timesteps between samples

* batch size: The number of times you sample per movement of our sliding window.

We will first define a function to produce data generator functions, as we will need one for both our test and validation sets. All this code is based off code from `deep learning with R` (the book)


```r
generator <- function(data, lookback, delay, min_index, max_index, shuffle = FALSE, 
    batch_size = 128, step = 6) {
    if (is.null(max_index)) 
        max_index <- nrow(data) - delay - 1
    i <- min_index + lookback
    function() {
        if (shuffle) {
            rows <- sample(c((min_index + lookback):max_index), size = batch_size)
        } else {
            if (i + batch_size >= max_index) 
                i <<- min_index + lookback
            rows <- c(i:min(i + batch_size - 1, max_index))
            i <<- i + length(rows)
        }
        
        samples <- array(0, dim = c(length(rows), lookback/step, dim(data)[[-1]]))
        targets <- array(0, dim = c(length(rows)))
        
        for (j in 1:length(rows)) {
            indices <- seq(rows[[j]] - lookback, rows[[j]] - 1, length.out = dim(samples)[[2]])
            samples[j, , ] <- data[indices, ]
            targets[[j]] <- data[rows[[j]] + delay, 2]
        }
        list(samples, targets)
    }
}
```

Next lets define our lookback and steps etc:


```r
lookback <- 60 * 24  # look back 60 days
step <- 6 * 24  # samle one data point every 6 days
delay <- 1 * 24  # look forward one day
batch_size <- 150 * 24  # Do this 3600 times, in other sample each window 2.5 times
```

Next lets define our training and validation generators


```r
train_gen <- generator(dat, lookback = lookback, delay = delay, min_index = 1, 
    max_index = maxt, shuffle = TRUE, step = step, batch_size = batch_size)

val_gen = generator(dat, lookback = lookback, delay = delay, min_index = maxt + 
    1, max_index = maxv, step = step, batch_size = batch_size)

# How many steps to draw from val_gen in order to see the entire validation
# set
val_steps <- ((maxv - maxt + 1 - lookback)/batch_size)
```

## Model building

Finally, we can define our model. Note what we did with the dropout. Dropout basically does exactly what it sounds like, when we are training our model, `n` percent of our data points that go in are randomly dropped, in the input an output nodes of each layer. This prevents the model from seeing too much of our training set, preventing overfitting. When predicting and running on our validation set, dropout is removed, as those are less data. This is why when we look at our loss curves in a moment, we see that the validation curve is lower than the train curve. Recurrent dropout is the same thing, except it occurs in the hidden layer, also preventing overfitting inthe LSTMs famous `forget gate`. We use the sigmoid activation function because it is most appropriate for time series data, especially our dataset.


```r
model_lstm <- keras_model_sequential() %>% layer_lstm(units = 10, dropout = 0.1, 
    recurrent_dropout = 0.1, activation = "sigmoid", input_shape = list(NULL, 
        dim(dat)[[-1]]), return_sequences = TRUE) %>% bidirectional(layer_lstm(units = 40, 
    dropout = 0.1, recurrent_dropout = 0.1, activation = "sigmoid", return_sequences = TRUE)) %>% 
    bidirectional(layer_lstm(units = 80, dropout = 0.1, recurrent_dropout = 0.1, 
        activation = "sigmoid")) %>% layer_dense(units = 1)
```


Next, we need to compile the model, or define how it trains. We will want to train it to optimize ASE, as we have been using that metric on the rest of the models. For our optimizing method, we will use `rmsprop`. To learn how RMSprop works, please see[this link](https://blog.paperspace.com/intro-to-optimization-momentum-rmsprop-adam/). It is typically a good optimizer for RNNs. We set the learning rate to 0.001 (chosen with great trial and error).



```r
model_lstm %>% compile(optimizer = optimizer_rmsprop(lr = 0.001), loss = "mse")
```

Finally, we get to train our model:



```r
Lhist <- model_lstm %>% fit_generator(train_gen, steps_per_epoch = 40, epochs = 40, 
    validation_data = val_gen, validation_steps = val_steps)
```

## Model Diagnostics

Lets look at our history to diagnose our model


```r
plot(Lhist)
```

<img src="fig/unnamed-chunk-67-1.png" style="display: block; margin: auto;" />

There are two things to know about the loss curve:

1. If the validation loss is greater than training loss, you are severly overfitting.

2. If the loss is too steep, learning rate is too high. If it is too flat, the learning rate is too low

For an amazing guide about tuning this, please refer to [this link](http://cs231n.github.io/neural-networks-3/#baby)

Knowing this, lets evaluate ours. It is apparent we are not overfitting. The loss curve is a little steeper than desirable, but it is definitely acceptable, and this is the best result of at least 1000 models. I also tried adding convolutional layers at the beginning, which is done frequently in related literature, as a tool for feature extraction, but we do not have near enough data (features) for that to work and overfit immediately. This model is a good balance of fast learning rate and not overfitting, and it is nice and stable.

## Prediction


As with the NNETAR, there is no explicit way of calculting the prediction interval of an LSTM, as it is not a stochastic, parametric tool, it is simply a composition of lots of functions.
However, we can use our dropout to approximate a prediction interval. For an in depth explanation, please refer to [this paper](https://arxiv.org/pdf/1506.02142.pdf). Basically, dropout is a random process. So each time we run our model on the training set, with our dropout, and our random generator function, we will get slightly different results (whereas when we make a prediction, we will not have the dropout, so it will be the same result every time). So, what we can do, is we can evaluate how well our model represents the training set, and use that to calculte a prediction interval. First, we will write a function that gets the mean error of our model over the training set:




```r
getMeanError <- function(mod) {
    # 28 is simply the integer number of times we need to sample from our
    # training set to represent the whole thing.  In this case it will run
    # through the entire set 4 times
    sqerror <- evaluate_generator(mod, train_gen, 28)
    return(sqrt(sqerror))
}
GetErrors <- function(n) {
    errors <- double(n)
    for (i in seq_along(errors)) {
        cat("evaluation number", i, "\n")
        errors[i] <- getMeanError(model_lstm)
    }
    return(errors)
}
trainErr <- GetErrors(1000)

errorMean <- mean((trainErr))
errorStd <- sd((trainErr))

makeInterval <- function(prediction) {
    pm <- errorMean + errorStd
    upper <- prediction + pm
    lower <- prediction - pm
    return(data.frame(fitted = prediction, upper = upper, lower = lower))
}
```

Now we can define scoring methods and autoplot methods:

## S3 Methods


```r
as.keras <- function(x) structure(x, class = "keras")
autoplot.keras <- function(obj) {
    testdf <- data.frame(type = "actual", t = seq_along(testM[, 1]), ppm = as.numeric(testM[, 
        1]))
    preddf <- data.frame(type = "predicted", t = seq_along(testM[, 1]), ppm = as.numeric(obj$fitted))
    confdf <- data.frame(t = seq_along(testM[, 1]), upper = obj$upper, lower = obj$lower)
    dfl <- list(testdf, preddf)
    .testPredPlot(dfl) + geom_line(data = confdf, aes(x = t, y = lower, alpha = 0.2), 
        linetype = 3003) + geom_line(data = confdf, aes(x = t, y = upper, alpha = 0.2), 
        linetype = 3003) + guides(alpha = FALSE)
}
scores.keras <- function(obj) {
    c(ase = ASE(obj$fitted, testM[, 1]), mape = MAPE(obj$fitted, testM[, 1]), 
        Conf.Score = confScore(upper = obj$upper, lower = obj$lower, testM[, 
            1]))
}
```

Next, lets make predictions:


```r
lstmTest <- predict(model_lstm, test3, n.ahead = nrow(test2))
lstmTest <- makeInterval(lstmTest)
descaleTest <- function(x) {
    x * attr(test2, "scaled:scale")[1] + attr(test2, "scaled:center")[1]
}
lstmTest <- lapply(lstmTest, descaleTest)
```

## Evaluation


```r
autoplot(as.keras(lstmTest))
```

<img src="fig/unnamed-chunk-72-1.png" style="display: block; margin: auto;" />

```r
data.frame(t(scores(as.keras(lstmTest))))
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["ase"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["mape"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["Conf.Score"],"name":[3],"type":["dbl"],"align":["right"]}],"data":[{"1":"15725.99","2":"275.8838","3":"0"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

This is an insanely good model. The hard work seriously paid off. It nearly matches the test set.

Lets go ahead and make a forecast on the validation (secondary training set) for our next model:


```r
lstmVal <- predict(model_lstm, val, n.ahead = nrow(test2))
lstmVal <- lstmVal * valScale + valCent
lstmVal <- as.keras(lstmVal)
```

# Ensembling

We will now walk through the setup of our ensemble model. We are going to make forecasts on the 72 hours ***BEFORE*** our test set (after retraining models that need to be retrained), and then train our gradient boosting machine on that. Then, we are going to feed our predictions on the test set into the gradient boosted model to assess. This type of ensembling is known as ***stacked ensembling***

## Data setup


```r
library(magrittr)
N <- nrow(trainM)
n <- 72

# set up a df to train our model on
valdf <- data.frame(ppm = trainM[(N - n + 1):(N), 1])
nrow(valdf)
```

```
#> [1] 72
```

```r
# set up the validation set data frame, all the data up to the last 144
# observations Note this is with trainM
valstack <- trainM[1:(N - n), ]
valstack <- trainM[1:(N - n), ]

nrow(valstack) - nrow(trainM)
```

```
#> [1] -72
```

```r
# make the columns a time series that need to be
ncol(valstack)
```

```
#> [1] 6
```

```r
valstack[, 1:5] %<>% lapply(function(x) ts(x, frequency = 8760))
str(valstack)
```

```
#> 'data.frame':	34920 obs. of  6 variables:
#>  $ PM_US.Post: Time-Series  from 1 to 4.99: 303 215 222 85 38 23 19 14 16 21 ...
#>  $ HUMI      : Time-Series  from 1 to 4.99: 72 78 72 72 49 45 45 45 42 35 ...
#>  $ PRES      : Time-Series  from 1 to 4.99: 1030 1031 1032 1033 1033 ...
#>  $ TEMP      : Time-Series  from 1 to 4.99: -8 -10 -9 -9 -4 -4 -4 -5 -4 -3 ...
#>  $ Iws       : Time-Series  from 1 to 4.99: 0.89 1.79 3.58 6.71 4.92 ...
#>  $ dayNight  : num  0 0 0 0 0 0 1 1 1 1 ...
```

```r
# create our test set
teststack <- data.frame(ppm = as.numeric(testM$PM))
```

Next, we make a tswge forecast on our validation set:


```r
teststack$ARUMA <- seafor$f
valwge <- fcst(type = aruma, x = valstack[[1]], theta = 0, s = 24 * 7, n.ahead = 72, 
    phi = est7$phi, plot = FALSE)
valdf$ARUMA <- valwge$f

plot(valdf[[1]], type = "l")
lines(valdf[[2]], col = "blue")
```

<img src="fig/unnamed-chunk-75-1.png" style="display: block; margin: auto;" />

Looks like tswge is still doing a good job, its got the shape down, but in this case it is a bit too high. This is the probably overfitting we did showing. Lets do nnetar next:


```r
teststack$nnet <- nnetfor$mean
newnet <- nnetar(model = nnet, y = valstack[[1]], xreg = valstack[-1])
netval <- forecast(newnet, xreg = valstack[(N - 2 * n + 1):(nrow(valstack)), 
    -1])
valdf$nnet <- netval$mean
plot(valdf[[1]], type = "l")
lines(as.numeric(netval$mean), col = "red")
```

<img src="fig/unnamed-chunk-76-1.png" style="display: block; margin: auto;" />

This is an excellent model.

Next, lets setup VAR


```r
teststack$VAR <- opred$fcst$PM_US.Post[, 1]
valVar <- VAR(valstack[-6], exogen = matrix(valstack[[6]], dimnames = list(NULL, 
    "dayNight")), type = "both", p = 30)
valVarP <- predict(valVar, n.ahead = 72, dumvar = makeExo(3))
valdf$VAR <- valVarP$fcst$PM_US.Post[, 1]
plot(valdf[[1]], type = "l")
lines(valdf$VAR, col = "blue")
```

<img src="fig/unnamed-chunk-77-1.png" style="display: block; margin: auto;" />

This is another good forecast, not as good as nnetar but still solid.

Next, it is time for tbats


```r
teststack$TBATS <- as.numeric(batF$mean)
valbat <- tbats(model = bjbats, y = msts(valstack$PM, seasonal.periods = c(24, 
    24 * 7, 8760)))
valbatfor <- forecast(valbat, h = 72)
valdf$TBATS <- valbatfor$mean

plot(valdf[[1]], type = "l")
lines(as.numeric(valdf$TBATS), col = "red")
```

<img src="fig/unnamed-chunk-78-1.png" style="display: block; margin: auto;" />

Another solid fit.

Lets try our other multiseasonal series next:


```r
teststack$harmonic <- mseasFor$mean
expansion3 <- fourier(msts(valstack$PM, seasonal.periods = c(24, 24 * 7, 8760)), 
    K = c(10, 20, 100))

mseaval <- Arima(model = mseaDay, y = msts(valstack$PM, seasonal.periods = c(24, 
    24 * 7, 8760)), xreg = expansion3)
mseavalF <- forecast(mseaval, xreg = fourier(msts(valdf$ppm, seasonal.periods = c(24, 
    24 * 7, 8760)), K = c(10, 20, 100)))
valdf$harmonic <- mseavalF$mean


plot(valdf[[1]], type = "l")
lines(as.numeric(valdf$harmonic), col = "blue")
```

<img src="fig/unnamed-chunk-79-1.png" style="display: block; margin: auto;" />

Interesting. This took a different approach, and should help balance out our models.

Finally, lets look at the LSTM


```r
valdf$LSTM <- lstmVal[-1]
teststack$LSTM <- lstmTest$f

plot(valdf[[1]], type = "l")
lines(as.numeric(valdf$LSTM), col = "red")
```

<img src="fig/unnamed-chunk-80-1.png" style="display: block; margin: auto;" />

This is an absurdly good model, and apparently not overfit.

Next, lets view all of the models in one plot:


```r
valdf %<>% lapply(as.numeric) %>% data.frame
teststack %<>% lapply(as.numeric) %>% data.frame
plot(valdf[[1]], t = "l")
lines(valdf[[2]], col = "red")
lines(valdf[[3]], col = "blue")
lines(valdf[[4]], col = "green")
lines(valdf[[5]], col = "purple")
lines(valdf[[6]], col = "orange")
lines(valdf[[7]], col = "turquoise")
legend(60, 500, legend = names(valdf), col = c("black", "red", "blue", "green", 
    "purple", "orange", "turquoise"), lty = 1:7)
```

<img src="fig/unnamed-chunk-81-1.png" style="display: block; margin: auto;" />

We have an ensemble of awesome forecasts.

## Ensembling

### S3 methods and Helper functions

Along with our standard S3 methods, we will make a function to help us quickly evaluate predictions:


```r
as.ens <- function(x) structure(x, class = "ens")
scores.ens <- function(obj) {
    ase <- ASE(obj, teststack[[1]])
    mape <- MAPE(obj, teststack[[1]])
    c(ASE = ase, MAPE = mape)
}

testModel <- function(model, ...) {
    scores(as.ens(predict(model, ..., newdata = teststack)))
}
```

### Baseline Mean

If we cant do better than the common sense mean of all models, what are we doing?


```r
modmean <- rowMeans(teststack[-1])
scores(as.ens(modmean))
```

```
#>        ASE       MAPE 
#> 15213.9805   255.0673
```

Already, this is a very good model, on par if not better than the LSTM. Lets plot it


```r
plot(teststack[[1]], t = "l")
lines(modmean, col = "blue")
```

<img src="fig/unnamed-chunk-84-1.png" style="display: block; margin: auto;" />

This is by no means a bad model, in fact its our best yet, but we can do better.

### Gradient Boosting: Feature Selection

We will try different combinations of features to see which ones are the most important. I have a feeling ARUMA is going to have to go.


```r
set.seed(503)  # for reproducibility
library(gbm)
b1 <- gbm(ppm ~ ., data = valdf)
```

```
#> Distribution not specified, assuming gaussian ...
```

```r
testModel(b1, n.trees = 100)
```

```
#>        ASE       MAPE 
#> 12535.2876   108.3212
```

Already we beat the baseline mean by a lot, lets see if we can drive it down even further:


```r
valdf2 <- valdf %>% dplyr::select(-ARUMA)

set.seed(503)  # for reproducibility
b2 <- gbm(ppm ~ ., data = valdf2)
```

```
#> Distribution not specified, assuming gaussian ...
```

```r
testModel(b2, n.trees = 100)
```

```
#>       ASE      MAPE 
#> 11323.177   162.094
```

Wow, we have now broken the 10,000 mark. Lets next do a grid search, in order to optimize our model. First, lets set up our grid:


```r
n.trees <- 100 * (1:10)
interaction.depth <- 1:6
shrinkage <- 10^(-(1:3))
gbmGrid <- expand.grid(list(n.trees, interaction.depth, shrinkage))
```

Then lets do the grid search (note this is super quick)


```r
search <- foreach(g = 1:nrow(gbmGrid), .combine = rbind) %do% {
    set.seed(503)
    model <- gbm(ppm ~ ., data = valdf2, n.trees = gbmGrid[g, 1], interaction.depth = gbmGrid[g, 
        2], shrinkage = gbmGrid[g, 3], distribution = "gaussian")
    score <- testModel(model, n.trees = gbmGrid[g, 1])
    return(data.frame(ASE = score[1], MAPE = score[2], n.trees = gbmGrid[g, 
        1], interaction.depth = gbmGrid[g, 2], shrinkage = gbmGrid[g, 3]))
}
```

Lets view the results:


```r
search %>% as_tibble %>% arrange(ASE)
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["ASE"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["MAPE"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["n.trees"],"name":[3],"type":["dbl"],"align":["right"]},{"label":["interaction.depth"],"name":[4],"type":["int"],"align":["right"]},{"label":["shrinkage"],"name":[5],"type":["dbl"],"align":["right"]}],"data":[{"1":"5405.314","2":"141.5800","3":"700","4":"2","5":"0.100"},{"1":"5405.314","2":"141.5800","3":"700","4":"3","5":"0.100"},{"1":"5405.314","2":"141.5800","3":"700","4":"4","5":"0.100"},{"1":"5405.314","2":"141.5800","3":"700","4":"5","5":"0.100"},{"1":"5405.314","2":"141.5800","3":"700","4":"6","5":"0.100"},{"1":"5480.970","2":"142.3529","3":"800","4":"2","5":"0.100"},{"1":"5480.970","2":"142.3529","3":"800","4":"3","5":"0.100"},{"1":"5480.970","2":"142.3529","3":"800","4":"4","5":"0.100"},{"1":"5480.970","2":"142.3529","3":"800","4":"5","5":"0.100"},{"1":"5480.970","2":"142.3529","3":"800","4":"6","5":"0.100"},{"1":"5529.252","2":"143.9740","3":"600","4":"2","5":"0.100"},{"1":"5529.252","2":"143.9740","3":"600","4":"3","5":"0.100"},{"1":"5529.252","2":"143.9740","3":"600","4":"4","5":"0.100"},{"1":"5529.252","2":"143.9740","3":"600","4":"5","5":"0.100"},{"1":"5529.252","2":"143.9740","3":"600","4":"6","5":"0.100"},{"1":"5568.126","2":"140.4900","3":"1000","4":"2","5":"0.100"},{"1":"5568.126","2":"140.4900","3":"1000","4":"3","5":"0.100"},{"1":"5568.126","2":"140.4900","3":"1000","4":"4","5":"0.100"},{"1":"5568.126","2":"140.4900","3":"1000","4":"5","5":"0.100"},{"1":"5568.126","2":"140.4900","3":"1000","4":"6","5":"0.100"},{"1":"5586.463","2":"141.0543","3":"900","4":"2","5":"0.100"},{"1":"5586.463","2":"141.0543","3":"900","4":"3","5":"0.100"},{"1":"5586.463","2":"141.0543","3":"900","4":"4","5":"0.100"},{"1":"5586.463","2":"141.0543","3":"900","4":"5","5":"0.100"},{"1":"5586.463","2":"141.0543","3":"900","4":"6","5":"0.100"},{"1":"5673.356","2":"147.1089","3":"500","4":"2","5":"0.100"},{"1":"5673.356","2":"147.1089","3":"500","4":"3","5":"0.100"},{"1":"5673.356","2":"147.1089","3":"500","4":"4","5":"0.100"},{"1":"5673.356","2":"147.1089","3":"500","4":"5","5":"0.100"},{"1":"5673.356","2":"147.1089","3":"500","4":"6","5":"0.100"},{"1":"5925.538","2":"145.4495","3":"400","4":"2","5":"0.100"},{"1":"5925.538","2":"145.4495","3":"400","4":"3","5":"0.100"},{"1":"5925.538","2":"145.4495","3":"400","4":"4","5":"0.100"},{"1":"5925.538","2":"145.4495","3":"400","4":"5","5":"0.100"},{"1":"5925.538","2":"145.4495","3":"400","4":"6","5":"0.100"},{"1":"6162.100","2":"156.7154","3":"300","4":"2","5":"0.100"},{"1":"6162.100","2":"156.7154","3":"300","4":"3","5":"0.100"},{"1":"6162.100","2":"156.7154","3":"300","4":"4","5":"0.100"},{"1":"6162.100","2":"156.7154","3":"300","4":"5","5":"0.100"},{"1":"6162.100","2":"156.7154","3":"300","4":"6","5":"0.100"},{"1":"6916.483","2":"156.9754","3":"200","4":"2","5":"0.100"},{"1":"6916.483","2":"156.9754","3":"200","4":"3","5":"0.100"},{"1":"6916.483","2":"156.9754","3":"200","4":"4","5":"0.100"},{"1":"6916.483","2":"156.9754","3":"200","4":"5","5":"0.100"},{"1":"6916.483","2":"156.9754","3":"200","4":"6","5":"0.100"},{"1":"7354.109","2":"179.5043","3":"100","4":"2","5":"0.100"},{"1":"7354.109","2":"179.5043","3":"100","4":"3","5":"0.100"},{"1":"7354.109","2":"179.5043","3":"100","4":"4","5":"0.100"},{"1":"7354.109","2":"179.5043","3":"100","4":"5","5":"0.100"},{"1":"7354.109","2":"179.5043","3":"100","4":"6","5":"0.100"},{"1":"8098.077","2":"153.0522","3":"1000","4":"1","5":"0.100"},{"1":"8399.900","2":"139.9457","3":"600","4":"1","5":"0.100"},{"1":"8562.382","2":"146.3333","3":"700","4":"1","5":"0.100"},{"1":"8586.670","2":"145.1485","3":"800","4":"1","5":"0.100"},{"1":"8887.935","2":"149.4850","3":"900","4":"1","5":"0.100"},{"1":"9378.696","2":"131.9705","3":"500","4":"1","5":"0.100"},{"1":"9991.307","2":"122.3199","3":"400","4":"1","5":"0.100"},{"1":"10472.818","2":"136.3276","3":"300","4":"1","5":"0.100"},{"1":"10716.486","2":"170.9298","3":"700","4":"1","5":"0.010"},{"1":"10753.061","2":"172.4200","3":"500","4":"1","5":"0.010"},{"1":"10800.796","2":"166.7360","3":"800","4":"1","5":"0.010"},{"1":"10806.784","2":"172.4220","3":"600","4":"1","5":"0.010"},{"1":"10823.114","2":"177.9055","3":"400","4":"1","5":"0.010"},{"1":"10936.492","2":"163.1068","3":"1000","4":"1","5":"0.010"},{"1":"10975.835","2":"165.7877","3":"900","4":"1","5":"0.010"},{"1":"11109.493","2":"176.3500","3":"500","4":"2","5":"0.010"},{"1":"11109.493","2":"176.3500","3":"500","4":"3","5":"0.010"},{"1":"11109.493","2":"176.3500","3":"500","4":"4","5":"0.010"},{"1":"11109.493","2":"176.3500","3":"500","4":"5","5":"0.010"},{"1":"11109.493","2":"176.3500","3":"500","4":"6","5":"0.010"},{"1":"11143.084","2":"183.4093","3":"400","4":"2","5":"0.010"},{"1":"11143.084","2":"183.4093","3":"400","4":"3","5":"0.010"},{"1":"11143.084","2":"183.4093","3":"400","4":"4","5":"0.010"},{"1":"11143.084","2":"183.4093","3":"400","4":"5","5":"0.010"},{"1":"11143.084","2":"183.4093","3":"400","4":"6","5":"0.010"},{"1":"11204.402","2":"166.5001","3":"700","4":"2","5":"0.010"},{"1":"11204.402","2":"166.5001","3":"700","4":"3","5":"0.010"},{"1":"11204.402","2":"166.5001","3":"700","4":"4","5":"0.010"},{"1":"11204.402","2":"166.5001","3":"700","4":"5","5":"0.010"},{"1":"11204.402","2":"166.5001","3":"700","4":"6","5":"0.010"},{"1":"11263.325","2":"187.5292","3":"300","4":"1","5":"0.010"},{"1":"11284.181","2":"165.1709","3":"800","4":"2","5":"0.010"},{"1":"11284.181","2":"165.1709","3":"800","4":"3","5":"0.010"},{"1":"11284.181","2":"165.1709","3":"800","4":"4","5":"0.010"},{"1":"11284.181","2":"165.1709","3":"800","4":"5","5":"0.010"},{"1":"11284.181","2":"165.1709","3":"800","4":"6","5":"0.010"},{"1":"11323.177","2":"162.0940","3":"100","4":"1","5":"0.100"},{"1":"11353.043","2":"172.0868","3":"600","4":"2","5":"0.010"},{"1":"11353.043","2":"172.0868","3":"600","4":"3","5":"0.010"},{"1":"11353.043","2":"172.0868","3":"600","4":"4","5":"0.010"},{"1":"11353.043","2":"172.0868","3":"600","4":"5","5":"0.010"},{"1":"11353.043","2":"172.0868","3":"600","4":"6","5":"0.010"},{"1":"11406.813","2":"161.3347","3":"900","4":"2","5":"0.010"},{"1":"11406.813","2":"161.3347","3":"900","4":"3","5":"0.010"},{"1":"11406.813","2":"161.3347","3":"900","4":"4","5":"0.010"},{"1":"11406.813","2":"161.3347","3":"900","4":"5","5":"0.010"},{"1":"11406.813","2":"161.3347","3":"900","4":"6","5":"0.010"},{"1":"11483.286","2":"196.7273","3":"300","4":"2","5":"0.010"},{"1":"11483.286","2":"196.7273","3":"300","4":"3","5":"0.010"},{"1":"11483.286","2":"196.7273","3":"300","4":"4","5":"0.010"},{"1":"11483.286","2":"196.7273","3":"300","4":"5","5":"0.010"},{"1":"11483.286","2":"196.7273","3":"300","4":"6","5":"0.010"},{"1":"11485.636","2":"160.8885","3":"1000","4":"2","5":"0.010"},{"1":"11485.636","2":"160.8885","3":"1000","4":"3","5":"0.010"},{"1":"11485.636","2":"160.8885","3":"1000","4":"4","5":"0.010"},{"1":"11485.636","2":"160.8885","3":"1000","4":"5","5":"0.010"},{"1":"11485.636","2":"160.8885","3":"1000","4":"6","5":"0.010"},{"1":"11504.517","2":"146.3666","3":"200","4":"1","5":"0.100"},{"1":"12461.439","2":"207.0917","3":"200","4":"1","5":"0.010"},{"1":"12744.084","2":"219.1690","3":"200","4":"2","5":"0.010"},{"1":"12744.084","2":"219.1690","3":"200","4":"3","5":"0.010"},{"1":"12744.084","2":"219.1690","3":"200","4":"4","5":"0.010"},{"1":"12744.084","2":"219.1690","3":"200","4":"5","5":"0.010"},{"1":"12744.084","2":"219.1690","3":"200","4":"6","5":"0.010"},{"1":"16126.146","2":"263.4539","3":"1000","4":"1","5":"0.001"},{"1":"16306.233","2":"270.0404","3":"1000","4":"2","5":"0.001"},{"1":"16306.233","2":"270.0404","3":"1000","4":"3","5":"0.001"},{"1":"16306.233","2":"270.0404","3":"1000","4":"4","5":"0.001"},{"1":"16306.233","2":"270.0404","3":"1000","4":"5","5":"0.001"},{"1":"16306.233","2":"270.0404","3":"1000","4":"6","5":"0.001"},{"1":"16447.061","2":"267.7146","3":"100","4":"2","5":"0.010"},{"1":"16447.061","2":"267.7146","3":"100","4":"3","5":"0.010"},{"1":"16447.061","2":"267.7146","3":"100","4":"4","5":"0.010"},{"1":"16447.061","2":"267.7146","3":"100","4":"5","5":"0.010"},{"1":"16447.061","2":"267.7146","3":"100","4":"6","5":"0.010"},{"1":"16784.127","2":"271.3976","3":"900","4":"1","5":"0.001"},{"1":"16942.527","2":"277.0997","3":"900","4":"2","5":"0.001"},{"1":"16942.527","2":"277.0997","3":"900","4":"3","5":"0.001"},{"1":"16942.527","2":"277.0997","3":"900","4":"4","5":"0.001"},{"1":"16942.527","2":"277.0997","3":"900","4":"5","5":"0.001"},{"1":"16942.527","2":"277.0997","3":"900","4":"6","5":"0.001"},{"1":"17055.268","2":"262.7740","3":"100","4":"1","5":"0.010"},{"1":"17525.618","2":"279.7907","3":"800","4":"1","5":"0.001"},{"1":"17615.508","2":"284.3916","3":"800","4":"2","5":"0.001"},{"1":"17615.508","2":"284.3916","3":"800","4":"3","5":"0.001"},{"1":"17615.508","2":"284.3916","3":"800","4":"4","5":"0.001"},{"1":"17615.508","2":"284.3916","3":"800","4":"5","5":"0.001"},{"1":"17615.508","2":"284.3916","3":"800","4":"6","5":"0.001"},{"1":"18315.734","2":"292.5518","3":"700","4":"2","5":"0.001"},{"1":"18315.734","2":"292.5518","3":"700","4":"3","5":"0.001"},{"1":"18315.734","2":"292.5518","3":"700","4":"4","5":"0.001"},{"1":"18315.734","2":"292.5518","3":"700","4":"5","5":"0.001"},{"1":"18315.734","2":"292.5518","3":"700","4":"6","5":"0.001"},{"1":"18323.578","2":"289.2858","3":"700","4":"1","5":"0.001"},{"1":"19098.305","2":"301.4923","3":"600","4":"2","5":"0.001"},{"1":"19098.305","2":"301.4923","3":"600","4":"3","5":"0.001"},{"1":"19098.305","2":"301.4923","3":"600","4":"4","5":"0.001"},{"1":"19098.305","2":"301.4923","3":"600","4":"5","5":"0.001"},{"1":"19098.305","2":"301.4923","3":"600","4":"6","5":"0.001"},{"1":"19209.371","2":"298.8006","3":"600","4":"1","5":"0.001"},{"1":"19997.084","2":"311.2179","3":"500","4":"2","5":"0.001"},{"1":"19997.084","2":"311.2179","3":"500","4":"3","5":"0.001"},{"1":"19997.084","2":"311.2179","3":"500","4":"4","5":"0.001"},{"1":"19997.084","2":"311.2179","3":"500","4":"5","5":"0.001"},{"1":"19997.084","2":"311.2179","3":"500","4":"6","5":"0.001"},{"1":"20231.390","2":"309.2233","3":"500","4":"1","5":"0.001"},{"1":"20997.649","2":"321.0972","3":"400","4":"2","5":"0.001"},{"1":"20997.649","2":"321.0972","3":"400","4":"3","5":"0.001"},{"1":"20997.649","2":"321.0972","3":"400","4":"4","5":"0.001"},{"1":"20997.649","2":"321.0972","3":"400","4":"5","5":"0.001"},{"1":"20997.649","2":"321.0972","3":"400","4":"6","5":"0.001"},{"1":"21376.987","2":"320.3206","3":"400","4":"1","5":"0.001"},{"1":"22278.382","2":"332.3351","3":"300","4":"2","5":"0.001"},{"1":"22278.382","2":"332.3351","3":"300","4":"3","5":"0.001"},{"1":"22278.382","2":"332.3351","3":"300","4":"4","5":"0.001"},{"1":"22278.382","2":"332.3351","3":"300","4":"5","5":"0.001"},{"1":"22278.382","2":"332.3351","3":"300","4":"6","5":"0.001"},{"1":"22681.493","2":"332.4207","3":"300","4":"1","5":"0.001"},{"1":"23622.578","2":"345.0661","3":"200","4":"2","5":"0.001"},{"1":"23622.578","2":"345.0661","3":"200","4":"3","5":"0.001"},{"1":"23622.578","2":"345.0661","3":"200","4":"4","5":"0.001"},{"1":"23622.578","2":"345.0661","3":"200","4":"5","5":"0.001"},{"1":"23622.578","2":"345.0661","3":"200","4":"6","5":"0.001"},{"1":"23883.538","2":"345.2225","3":"200","4":"1","5":"0.001"},{"1":"25168.668","2":"359.7843","3":"100","4":"2","5":"0.001"},{"1":"25168.668","2":"359.7843","3":"100","4":"3","5":"0.001"},{"1":"25168.668","2":"359.7843","3":"100","4":"4","5":"0.001"},{"1":"25168.668","2":"359.7843","3":"100","4":"5","5":"0.001"},{"1":"25168.668","2":"359.7843","3":"100","4":"6","5":"0.001"},{"1":"25532.703","2":"359.3627","3":"100","4":"1","5":"0.001"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

These are absurd models. We more than halved our mean ensemble. Lets go ahead and implement this new model:


```r
set.seed(503)
finalModel <- gbm(ppm ~ ., data = valdf2, n.trees = 700, interaction.depth = 2, 
    distribution = "gaussian")
testModel(finalModel, n.trees = 700)
```

```
#>      ASE     MAPE 
#> 5405.314  141.580
```

```r
plot(teststack$ppm, type = "l")
lines(predict(finalModel, newdata = teststack, n.trees = 700), col = "blue")
```

<img src="fig/unnamed-chunk-90-1.png" style="display: block; margin: auto;" />

```r
finalPred <- predict(finalModel, newdata = teststack, n.trees = 700)
scores(as.ens(finalPred))
```

```
#>      ASE     MAPE 
#> 5405.314  141.580
```

Oh man...
This is an incredible model. Next, lets calculate a prediction interval. For this, we will run it 10000 times and bootstrap the residuals. Because forecasts get worse with time, we will then order them sequentially (I am making a lot assumptions here). Because we included 5 models here, and each of those have uncertainty too, lets assume (generously) that we will have 5 times the uncertainty in our ensembled model:


```r
load("analysis/hourly/pint.Rda")
```


```r
library(boot)
set.seed(NULL)

bootfun <- function(data, indices) {
    data <- data[indices, ]
    tr <- gbm(ppm ~ ., data = data, n.trees = 700, interaction.depth = 2, distribution = "gaussian")
    predict(tr, newdata = teststack, n.trees = 700)
}

# bootstrap residuals
b <- boot(data = valdf2, statistic = bootfun, R = 10000, parallel = "multicore")
# 95% limits
lims <- t(apply(b$t, 2, FUN = function(x) quantile(x, c(0.05, 0.975))))
# order them
pm <- (lims[, 2] - lims[, 1])[order(lims[, 2] - lims[, 1])]/2
```

Again because we have 5 models, lets say theres 5 times the uncertainty. This is safe enough for me, although it assumes a lot


```r
pm <- pm * (ncol(valdf2) - 1)
makeInterval <- function(x) {
    upper <- x + pm
    lower <- x - pm
    return(data.frame(predicted = x, upper = upper, lower = lower))
}
```

Now lets make an interval on our prediction:


```r
finalPred <- makeInterval(finalPred)
```

And final S3 methods update:


```r
autoplot.ens <- function(obj) {
    testdf <- data.frame(type = "actual", t = seq_along(testM[, 1]), ppm = as.numeric(testM[, 
        1]))
    preddf <- data.frame(type = "predicted", t = seq_along(testM[, 1]), ppm = as.numeric(obj$predicted))
    confdf <- data.frame(t = seq_along(testM[, 1]), upper = obj$upper, lower = obj$lower)
    dfl <- list(testdf, preddf)
    .testPredPlot(dfl) + geom_line(data = confdf, aes(x = t, y = lower, alpha = 0.2), 
        linetype = 3003) + geom_line(data = confdf, aes(x = t, y = upper, alpha = 0.2), 
        linetype = 3003) + guides(alpha = FALSE)
}
scores.ens <- function(obj) {
    c(ase = ASE(obj$predicted, testM[, 1]), mape = MAPE(obj$predicted, testM[, 
        1]), Conf.Score = confScore(upper = obj$upper, lower = obj$lower, testM[, 
        1]))
}
```

## Assessing the model


```r
autoplot(as.ens(finalPred))
```

<img src="fig/unnamed-chunk-96-1.png" style="display: block; margin: auto;" />

```r
scores(as.ens(finalPred)) %>% t %>% data.frame
```

<div data-pagedtable="false">
  <script data-pagedtable-source type="application/json">
{"columns":[{"label":["ase"],"name":[1],"type":["dbl"],"align":["right"]},{"label":["mape"],"name":[2],"type":["dbl"],"align":["right"]},{"label":["Conf.Score"],"name":[3],"type":["dbl"],"align":["right"]}],"data":[{"1":"5405.314","2":"141.58","3":"0"}],"options":{"columns":{"min":{},"max":[10]},"rows":{"min":[10],"max":[10]},"pages":{}}}
  </script>
</div>

Well, this is a miraculous model. There is not much left to say. Lets now forecast *into the future*

# Forecasting

We will now make 72 hour ahead predictions of PM2.5 content in Beijing.

First, lets get rid of that train test split and have our whole dataset, and transform it just as we had in the previous sections


```r
processData <- function() {
    bjnots <- purrr::discard(bj, is.ts)
    bjnots <- data.frame(lapply(bjnots, as.factor))
    bjts <- purrr::keep(bj, is.ts)
    bjts$PM_US.Post <- uvar  # just because it is already cleaned
    alldata <<- cbind(bjts, bjnots)
}

processData()
str(alldata)
```

```
#> 'data.frame':	52584 obs. of  14 variables:
#>  $ PM_US.Post   : Time-Series  from 1 to 7: 129 148 159 181 138 ...
#>  $ DEWP         : Time-Series  from 1 to 7: -21 -21 -21 -21 -20 -19 -19 -19 -19 -20 ...
#>  $ HUMI         : Time-Series  from 1 to 7: 43 47 43 55 51 47 44 44 44 37 ...
#>  $ PRES         : Time-Series  from 1 to 7: 1021 1020 1019 1019 1018 ...
#>  $ TEMP         : Time-Series  from 1 to 7: -11 -12 -11 -14 -12 -10 -9 -9 -9 -8 ...
#>  $ Iws          : Time-Series  from 1 to 7: 1.79 4.92 6.71 9.84 12.97 ...
#>  $ precipitation: Time-Series  from 1 to 7: 0 0 0 0 0 0 0 0 0 0 ...
#>  $ Iprec        : Time-Series  from 1 to 7: 0 0 0 0 0 0 0 0 0 0 ...
#>  $ year         : Factor w/ 6 levels "2010","2011",..: 1 1 1 1 1 1 1 1 1 1 ...
#>  $ month        : Factor w/ 12 levels "1","2","3","4",..: 1 1 1 1 1 1 1 1 1 1 ...
#>  $ day          : Factor w/ 31 levels "1","2","3","4",..: 1 1 1 1 1 1 1 1 1 1 ...
#>  $ hour         : Factor w/ 24 levels "0","1","2","3",..: 1 2 3 4 5 6 7 8 9 10 ...
#>  $ season       : Factor w/ 4 levels "1","2","3","4": 4 4 4 4 4 4 4 4 4 4 ...
#>  $ cbwd         : Factor w/ 4 levels "cv","NE","NW",..: 3 3 3 3 3 3 3 3 3 3 ...
```

```r
alldata$dayNight <- hoursToDayNight(alldata)

alldata[c(7:14)] <- NULL
alldata$DEWP <- NULL
```

## VAR

Next, lets make the VAR forecast, so we have all our exogenous predictors.


```r
varmod <- VAR(alldata[-6], exogen = matrix(alldata[[6]], dimnames = list(NULL, 
    "dayNight")), type = "both", p = 30)
varpred <- predict(varmod, n.ahead = 72, dumvar = makeExo(3))

# make an xvar data frame
xvars <- rapply(varpred$fcst, function(x) x[, 1], how = "list") %>% data.frame %>% 
    dplyr::select(-PM_US.Post)
xvars$dayNight <- c(makeExo(3))
predictions <- data.frame(VAR = varpred$fcst$PM[, 1])
```

## Classical

Next, lets use tswge to make an ARUMA forecast (this is still important)


```r
aruma <- fore.aruma.wge(x = alldata[[1]], theta = 0, s = 24 * 7, n.ahead = 72, 
    phi = est7$phi, plot = FALSE)$f
predictions$ARUMA <- aruma
```

## Autocorrelated errors


```r
expansion3 <- fourier(msts(alldata$PM, seasonal.periods = c(24, 24 * 7, 8760)), 
    K = c(10, 20, 100))

mseamod <- Arima(model = mseaDay, y = msts(alldata$PM, seasonal.periods = c(24, 
    24 * 7, 8760)), xreg = expansion3)


harmonF <- forecast(mseamod, xreg = fourier(msts(tail(alldata$PM, 72), seasonal.periods = c(24, 
    24 * 7, 8760)), K = c(10, 20, 100)))

predictions$harmonic <- harmonF$mean
```

## TBATS


```r
tbatmod <- tbats(model = bjbats, y = msts(alldata$PM, seasonal.periods = c(24, 
    24 * 7, 8760)))
batfor <- forecast(tbatmod, h = 72)

predictions$TBATS <- batfor$mean
```

## NNETAR


```r
newnet <- nnetar(model = nnet, y = alldata[[1]], xreg = alldata[-1])
netfor <- forecast(newnet, xreg = xvars)

predictions$nnet <- netfor$mean
```

## LSTM

This one is tricky. LSTM requires us to put a target variable into it, because it csn only accept arrays of that shape. The first column of the array is our target set, which does not affect the prediction. We could put only zeroes there and it would be fine. However, we need to pick something with a good scale. For that, the standard deviation of our ARUMA forecasts was very good relative to our real data. So we will use that as our scaling variable:


```r
input <- data.matrix(data.frame(aruma, xvars))
# data setup
input <- scale(input, center = apply(input, 2, mean), scale = apply(input, 2, 
    sd))
inputScale <- attr(input, "scaled:scale")[1]
inputCent <- attr(input, "scaled:center")[1]
# descaler
descaleInput <- function(x) {
    x * inputScale + inputCent
}
# to array
input <- array(input, c(nrow(input), 10, 6))
lstmPred <- predict(model_lstm, input, n.ahead = 72)
lstmPred <- descaleInput(lstmPred)
predictions$LSTM <- lstmPred
```

## Ensembling

Lets combine our forecasts: 

```r
futuresight <- predict(finalModel, newdata = predictions, n.trees = 700)
futuresight <- makeInterval(futuresight)
plot(futuresight$predicted, type = "o")
lines(futuresight$upper, type = "b", col = "blue")
lines(futuresight$lower, type = "b", col = "blue")
```

<img src="fig/unnamed-chunk-104-1.png" style="display: block; margin: auto;" />

looks like the shape is on point. Lets plot a few days back so we can see if weve made a realistic forecast:


```r
numbah <- 72 * 2
past <- as.numeric(tail(alldata$PM, numbah))
tAdd <- length(past)
library(lubridate)
library(stringr)

# convert month, day, and hour columns to date
col2date <- function(v) {
    v <- as.character(v)
    v %<>% vapply(function(x) str_pad(x, 2, pad = "0"), character(1))
    dte <- paste(v[1], v[2], v[3], sep = "-")
    tme <- paste0(v[4], ":00:00")
    dtme <- paste(dte, tme)
    as_datetime(dtme)
}

start <- tail(bj[1:4], tAdd) %>% head(1) %>% col2date
pastdates <- start + hours(x = seq.int(0, tAdd - 1))
futuredates <- tail(pastdates, 1) + hours(x = seq.int(1, 72))

pastdf <- data.frame(time = pastdates, PM2.5 = past)
futuredf <- data.frame(time = futuredates, futuresight)
models <- data.frame(time = futuredates, predictions)
modeldf <- models %>% gather_(key = "model", value = "value", gather_cols = names(models)[-1])
connectdf <- data.frame(time = c(tail(pastdf$time, 1), head(futuredf$time, 1)), 
    val = c(tail(pastdf$PM2.5, 1), head(futuredf$predicted, 1)))
library(ggthemes)
ggplot() + geom_line(data = pastdf, aes(x = time, y = PM2.5), color = "black") + 
    geom_line(data = futuredf, aes(x = time, y = predicted), color = "blue") + 
    geom_line(data = futuredf, aes(x = time, y = upper), linetype = 3003) + 
    geom_line(data = futuredf, aes(x = time, y = lower), linetype = 3003) + 
    geom_line(data = connectdf, aes(x = time, y = val), color = "blue") + geom_ribbon(data = futuredf, 
    aes(x = time, ymin = lower, ymax = upper), alpha = 0.1) + theme_hc() + scale_color_calc() + 
    ggtitle("Gradient Boosted Ensemble Forecast of PM2.5 in Beijing") + theme(plot.title = element_text(hjust = 0.5))
```

<img src="fig/unnamed-chunk-105-1.png" style="display: block; margin: auto;" />

And there we go :)
Last but no least, lets write our forecast to a csv:


```r
write.csv(futuredf, "predictions.csv")
```

