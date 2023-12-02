---
title: "Portfolio Analysis: a beginning"
date: 2023-12-01 13:17:00 -0500
categories: [Analytics, R programming]
tags: [Finance]
render_with_liquid: false
---

This document is an exploration of R coding, applied to analyzing a financial portfolio. We will scrape ETF  stock data from web pages and APIs (via `quantmod`), calculate financial metrics, and produce tables. In a future post we will create graphs and charts.

## Librairies

The following libraries are used:

The `rvest` package in R is a popular and powerful tool designed for web scraping. It allows users to easily read and manipulate the data from web pages. The `tydiverse` package is a collection of packages useful for data science, including  `ggplot2` and `dyplr` which are necessary for the code used here. The package `flextable` is used to produce nice-looking HTML tables.

```r
#Run the libraries
    library(rvest)
    library(tidyverse)
    library(flextable)
```

## Scraping ticker data

The following code uses tools of the **rvest** library to create a function that scrapes data from finance.yahoo.com. This requires getting the XPATH to specific data in the web page. If the web page changes, it will break the script. It probably would be best  in the future to change this for getting info from a database (via API or other means).

```r
# This is a function that takes one argument: ticker. The function constructs a URL for the
# Yahoo Finance page of the given ticker, then uses read_html to download and parse the HTML
# content of that page. 
    get_financials <- function(ticker) {
      url <- paste0("https://finance.yahoo.com/quote/", ticker)
      page <- read_html(url)
      
      pe_ratio <- page %>%
        html_nodes(xpath = '/html/body/div[1]/div/div/div[1]/div/div[3]/div[1]/div/div[1]/div/div/div/div[2]/div[2]/table/tbody/tr[3]/td[2]') %>%
        html_text() %>%
        as.numeric()
      
#Earnings yield is the inverse of the P/E ratio. We can calculate this here.
      earnings_yield <- round(1 / pe_ratio, 4)
      

      expense_ratio_text <- page %>%
        html_nodes(xpath = '/html/body/div[1]/div/div/div[1]/div/div[3]/div[1]/div/div[1]/div/div/div/div[2]/div[2]/table/tbody/tr[7]/td[2]') %>%
        html_text()
      
      # Remove the '%' sign and convert to numeric
      expense_ratio <- as.numeric(gsub("%", "", expense_ratio_text))
      
      
      return(data.frame(Ticker = ticker, ExpenseRatio = expense_ratio, PE = pe_ratio, EarningsYield = earnings_yield))
    }

    tickers <- c("VUN.TO", "VCN.TO", "XEF.TO", "AVUV", "AVDV", "XEC.TO", "AVES")  # The tickers part of the portfolio

# Define the weights for each ticker in the portfolio. This will be used to calculate the weighted averages.
    portfolio_weights <- c("VUN.TO" = 0.315, "VCN.TO" = 0.23, "XEF.TO" = 0.165, 
                           "AVUV" = 0.115, "AVDV" = 0.075, "XEC.TO" = 0.05, "AVES" = 0.05)

# `lapply` is a function in R that applies a function over a list or vector. In this case,
# the function get_financials is applied to each element of the tickers vector. `bind_rows()`
# combines multiple data frames into one by binding them row-wise. This means that it takes
# data frames and stacks them on top of each other.
    financial_data <- lapply(tickers, get_financials) %>%
      bind_rows()
```

Since the Vanguard and Blackrock tickers' Expense Ratios are missing, the following code adjusts for this by manually inserting values. I will eventually add code to scrape the data from their respective websites.

```r
# Manually adjust 
    correct_expense_ratios <- c("VUN.TO" = 0.17, "VCN.TO" = 0.05, "XEF.TO" = 0.22, "XEC.TO" = 0.28)
    financial_data$ExpenseRatio <- ifelse(financial_data$ExpenseRatio == 0,
                                          correct_expense_ratios[financial_data$Ticker],
                                          financial_data$ExpenseRatio)
```

| Ticker | ExpenseRatio | PE    | EarningsYield |
|:-------|:-------------|:------|--------------:|
| VUN.TO | 0.17         | 21.78 | 0.0459        |
| VCN.TO | 0.05         | 12.27 | 0.0815        |
| XEF.TO | 0.22         | 13.65 | 0.0733        |
| AVUV   | 0.25         | 7.15  | 0.1399        |
| AVDV   | 0.36         | 7.25  | 0.1379        |
| XEC.TO | 0.38         | 11.37 | 0.0880        |
| AVES   | 0.36         | 7.53  | 0.1328        |


## Add weights and calculate averages

We manually add the weight of each tickers in the portfolio.

```r
# Add the weights to the data frame
financial_data$Weight <- portfolio_weights[financial_data$Ticker]

# Rearrange the columns
financial_data <- financial_data[c("Ticker", "Weight", "PE", "EarningsYield", "ExpenseRatio")]
```

The following blocks are to calculate the weighted averages. Here are the average Expense Ratios.

```r
# Calculate the weighted ER for each ticker
financial_data$WeightedER <- financial_data$ExpenseRatio * financial_data$Weight

# Calculate the sum of the weighted ER  to get the weighted average ER
weighted_average_er <- round(sum(financial_data$WeightedER),2)

print(paste("Weighted Average ER:", weighted_average_er))
```
`[1] "Weighted Average ER: 0.19"`

Here are the weighted average PE ratios.

```r
# Calculate the weighted PE for each ticker
financial_data$WeightedPE <- round(financial_data$PE * financial_data$Weight, 2)

# Calculate the sum of the weighted PEs to get the weighted average PE
weighted_average_pe <- round(sum(financial_data$WeightedPE),2)

print(paste("Weighted Average PE:", weighted_average_pe))

```
`[1] "Weighted Average PE: 14.22"`

Here are the sum of ER ratios. I want to print it out as a %. 
```r
# Calculate the weighted Earning Yields for each ticker
financial_data$WeightedEY <- round(financial_data$EarningsYield * financial_data$Weight, 4)

# Calculate the sum of the EarningYields
weighted_average_ey <- sum(financial_data$WeightedEY)

# Convert to percentage
weighted_average_ey_percent <- weighted_average_ey * 100

print(sprintf("Weighted Average Earning Yields: %.2f%%", weighted_average_ey_percent))
```
`[1] "Weighted Average Earning Yields: 8.29%"`

Here we create a summary data frame. 
```r
# Create a summary data frame
summary_data <- data.frame(
  Metric = c("Weighted Average PE", "Weighted Average ER", "Weighted Average EY"),
  Value = c(weighted_average_pe, weighted_average_er, weighted_average_ey_percent)
)
```

## Print out data tables
Now we print out the results in two `flextables` tables.

```r
flextable(financial_data)
```
<div class="tabwid"><style>.cl-bb5b6256{}.cl-bb540a24{font-family:'Arial';font-size:11pt;font-weight:normal;font-style:normal;text-decoration:none;color:rgba(0, 0, 0, 1.00);background-color:white;}.cl-bb56dca4{margin:0;text-align:left;border-bottom: 0 solid rgba(0, 0, 0, 1.00);border-top: 0 solid rgba(0, 0, 0, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);padding-bottom:5pt;padding-top:5pt;padding-left:5pt;padding-right:5pt;line-height: 1;background-color:white;}.cl-bb56dcae{margin:0;text-align:right;border-bottom: 0 solid rgba(0, 0, 0, 1.00);border-top: 0 solid rgba(0, 0, 0, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);padding-bottom:5pt;padding-top:5pt;padding-left:5pt;padding-right:5pt;line-height: 1;background-color:white;}.cl-bb56ed98{width:0.75in;background-color:white;vertical-align: middle;border-bottom: 1.5pt solid rgba(102, 102, 102, 1.00);border-top: 1.5pt solid rgba(102, 102, 102, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);margin-bottom:0;margin-top:0;margin-left:0;margin-right:0;}.cl-bb56ed99{width:0.75in;background-color:white;vertical-align: middle;border-bottom: 1.5pt solid rgba(102, 102, 102, 1.00);border-top: 1.5pt solid rgba(102, 102, 102, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);margin-bottom:0;margin-top:0;margin-left:0;margin-right:0;}.cl-bb56ed9a{width:0.75in;background-color:white;vertical-align: middle;border-bottom: 0 solid rgba(0, 0, 0, 1.00);border-top: 0 solid rgba(0, 0, 0, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);margin-bottom:0;margin-top:0;margin-left:0;margin-right:0;}.cl-bb56eda2{width:0.75in;background-color:white;vertical-align: middle;border-bottom: 0 solid rgba(0, 0, 0, 1.00);border-top: 0 solid rgba(0, 0, 0, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);margin-bottom:0;margin-top:0;margin-left:0;margin-right:0;}.cl-bb56eda3{width:0.75in;background-color:white;vertical-align: middle;border-bottom: 1.5pt solid rgba(102, 102, 102, 1.00);border-top: 0 solid rgba(0, 0, 0, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);margin-bottom:0;margin-top:0;margin-left:0;margin-right:0;}.cl-bb56eda4{width:0.75in;background-color:white;vertical-align: middle;border-bottom: 1.5pt solid rgba(102, 102, 102, 1.00);border-top: 0 solid rgba(0, 0, 0, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);margin-bottom:0;margin-top:0;margin-left:0;margin-right:0;}</style><table data-quarto-disable-processing='true' class='cl-bb5b6256'><thead><tr style="overflow-wrap:break-word;"><th class="cl-bb56ed98"><p class="cl-bb56dca4"><span class="cl-bb540a24">Ticker</span></p></th><th class="cl-bb56ed99"><p class="cl-bb56dcae"><span class="cl-bb540a24">Weight</span></p></th><th class="cl-bb56ed99"><p class="cl-bb56dcae"><span class="cl-bb540a24">PE</span></p></th><th class="cl-bb56ed99"><p class="cl-bb56dcae"><span class="cl-bb540a24">EarningsYield</span></p></th><th class="cl-bb56ed99"><p class="cl-bb56dcae"><span class="cl-bb540a24">ExpenseRatio</span></p></th><th class="cl-bb56ed99"><p class="cl-bb56dcae"><span class="cl-bb540a24">WeightedER</span></p></th><th class="cl-bb56ed99"><p class="cl-bb56dcae"><span class="cl-bb540a24">WeightedPE</span></p></th><th class="cl-bb56ed99"><p class="cl-bb56dcae"><span class="cl-bb540a24">WeightedEY</span></p></th></tr></thead><tbody><tr style="overflow-wrap:break-word;"><td class="cl-bb56ed9a"><p class="cl-bb56dca4"><span class="cl-bb540a24">VUN.TO</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.315</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">21.76</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.0460</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.17</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.05</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">6.85</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.0145</span></p></td></tr><tr style="overflow-wrap:break-word;"><td class="cl-bb56ed9a"><p class="cl-bb56dca4"><span class="cl-bb540a24">VCN.TO</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.230</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">12.23</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.0818</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.05</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.01</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">2.81</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.0188</span></p></td></tr><tr style="overflow-wrap:break-word;"><td class="cl-bb56ed9a"><p class="cl-bb56dca4"><span class="cl-bb540a24">XEF.TO</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.165</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">13.65</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.0733</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.22</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.04</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">2.25</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.0121</span></p></td></tr><tr style="overflow-wrap:break-word;"><td class="cl-bb56ed9a"><p class="cl-bb56dca4"><span class="cl-bb540a24">AVUV</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.115</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">7.12</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.1404</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.25</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.03</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.82</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.0161</span></p></td></tr><tr style="overflow-wrap:break-word;"><td class="cl-bb56ed9a"><p class="cl-bb56dca4"><span class="cl-bb540a24">AVDV</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.075</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">7.26</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.1377</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.36</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.03</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.54</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.0103</span></p></td></tr><tr style="overflow-wrap:break-word;"><td class="cl-bb56ed9a"><p class="cl-bb56dca4"><span class="cl-bb540a24">XEC.TO</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.050</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">11.31</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.0884</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.28</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.01</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.57</span></p></td><td class="cl-bb56eda2"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.0044</span></p></td></tr><tr style="overflow-wrap:break-word;"><td class="cl-bb56eda3"><p class="cl-bb56dca4"><span class="cl-bb540a24">AVES</span></p></td><td class="cl-bb56eda4"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.050</span></p></td><td class="cl-bb56eda4"><p class="cl-bb56dcae"><span class="cl-bb540a24">7.50</span></p></td><td class="cl-bb56eda4"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.1333</span></p></td><td class="cl-bb56eda4"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.36</span></p></td><td class="cl-bb56eda4"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.02</span></p></td><td class="cl-bb56eda4"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.38</span></p></td><td class="cl-bb56eda4"><p class="cl-bb56dcae"><span class="cl-bb540a24">0.0067</span></p></td></tr></tbody></table></div>


```r
flextable(summary_data)
```
<div class="tabwid"><style>.cl-bb66bc5a{}.cl-bb6042a8{font-family:'Arial';font-size:11pt;font-weight:normal;font-style:normal;text-decoration:none;color:rgba(0, 0, 0, 1.00);background-color:white;}.cl-bb62d130{margin:0;text-align:left;border-bottom: 0 solid rgba(0, 0, 0, 1.00);border-top: 0 solid rgba(0, 0, 0, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);padding-bottom:5pt;padding-top:5pt;padding-left:5pt;padding-right:5pt;line-height: 1;background-color:white;}.cl-bb62d13a{margin:0;text-align:right;border-bottom: 0 solid rgba(0, 0, 0, 1.00);border-top: 0 solid rgba(0, 0, 0, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);padding-bottom:5pt;padding-top:5pt;padding-left:5pt;padding-right:5pt;line-height: 1;background-color:white;}.cl-bb62e3e6{width:0.75in;background-color:white;vertical-align: middle;border-bottom: 1.5pt solid rgba(102, 102, 102, 1.00);border-top: 1.5pt solid rgba(102, 102, 102, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);margin-bottom:0;margin-top:0;margin-left:0;margin-right:0;}.cl-bb62e3f0{width:0.75in;background-color:white;vertical-align: middle;border-bottom: 1.5pt solid rgba(102, 102, 102, 1.00);border-top: 1.5pt solid rgba(102, 102, 102, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);margin-bottom:0;margin-top:0;margin-left:0;margin-right:0;}.cl-bb62e3f1{width:0.75in;background-color:white;vertical-align: middle;border-bottom: 0 solid rgba(0, 0, 0, 1.00);border-top: 0 solid rgba(0, 0, 0, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);margin-bottom:0;margin-top:0;margin-left:0;margin-right:0;}.cl-bb62e3fa{width:0.75in;background-color:white;vertical-align: middle;border-bottom: 0 solid rgba(0, 0, 0, 1.00);border-top: 0 solid rgba(0, 0, 0, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);margin-bottom:0;margin-top:0;margin-left:0;margin-right:0;}.cl-bb62e3fb{width:0.75in;background-color:white;vertical-align: middle;border-bottom: 1.5pt solid rgba(102, 102, 102, 1.00);border-top: 0 solid rgba(0, 0, 0, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);margin-bottom:0;margin-top:0;margin-left:0;margin-right:0;}.cl-bb62e3fc{width:0.75in;background-color:white;vertical-align: middle;border-bottom: 1.5pt solid rgba(102, 102, 102, 1.00);border-top: 0 solid rgba(0, 0, 0, 1.00);border-left: 0 solid rgba(0, 0, 0, 1.00);border-right: 0 solid rgba(0, 0, 0, 1.00);margin-bottom:0;margin-top:0;margin-left:0;margin-right:0;}</style><table data-quarto-disable-processing='true' class='cl-bb66bc5a'><thead><tr style="overflow-wrap:break-word;"><th class="cl-bb62e3e6"><p class="cl-bb62d130"><span class="cl-bb6042a8">Metric</span></p></th><th class="cl-bb62e3f0"><p class="cl-bb62d13a"><span class="cl-bb6042a8">Value</span></p></th></tr></thead><tbody><tr style="overflow-wrap:break-word;"><td class="cl-bb62e3f1"><p class="cl-bb62d130"><span class="cl-bb6042a8">Weighted Average PE</span></p></td><td class="cl-bb62e3fa"><p class="cl-bb62d13a"><span class="cl-bb6042a8">14.22</span></p></td></tr><tr style="overflow-wrap:break-word;"><td class="cl-bb62e3f1"><p class="cl-bb62d130"><span class="cl-bb6042a8">Weighted Average ER</span></p></td><td class="cl-bb62e3fa"><p class="cl-bb62d13a"><span class="cl-bb6042a8">0.19</span></p></td></tr><tr style="overflow-wrap:break-word;"><td class="cl-bb62e3fb"><p class="cl-bb62d130"><span class="cl-bb6042a8">Weighted Average EY</span></p></td><td class="cl-bb62e3fc"><p class="cl-bb62d13a"><span class="cl-bb6042a8">8.29</span></p></td></tr></tbody></table></div>

> If you have large tables, you can use the `DT` package, which utilizes the JavaScript library DataTables to create interactive and dynamic outputs. This allows for functions like search and sort.
{: .prompt-tip }
