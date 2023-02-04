README
================
2023-01-20

# Purpose

The purpose of this README is to provide sanitized code for my Advanced
Financial Econometrics 871 project. The purpose of the project is to
investigate time-varying conditional correlations between
FTSE/JSE-listed property securities and the broader FTSE/JSE assets by
adopting the Dynamic Conditional Correlation (DCC) Multivariate
Generalized Autoregressive Conditional Heteroskedasticity (MV GARCH)
modelling procedure. See the PDF document in the repository for the
final project.

The DCC MV-GARCH model’s results suggest that, on aggregate, the
comovement between PROP, SWIX(J433), and SMLC(J202) is amplified during
periods of heightened global economic uncertainty. Although the PROP
index is the most volatile of the indexes and the SMLC(J202) index the
least volatile, it exhibits a substantially lower time-varying
conditional correlation with the SWIX(J433) index compared to the
SMLC(J202) index. However, since 2016, this difference has shrunk
marginally. On the other hand, the dynamic conditional correlation
between the SMLC(J202) and SWIX(J433) indexes is the highest and the
more stable among the three index pairs considered. These results infer
that an investor holding large proportions of SWIX(J433) constituents
will achieve superior portfolio diversification in purchasing listed
property compared to SMLC(J202) constituents.

``` r
pacman::p_load("MTS", "robustbase")
pacman::p_load("tidyverse", "devtools", "rugarch", "rmgarch", 
    "forecast", "tbl2xts", "lubridate", "PerformanceAnalytics", 
    "ggthemes")
```

# Data import

``` r
T40 <- read_rds("data/data_from_exam/T40.rds") # There are 92 stocks in this tbl

RebDays <- read_rds("data/data_from_exam/Rebalance_days.rds")

Capped_SWIX <- read_rds("data/data_from_exam/Capped_SWIX.rds") # This is the Monthly Capped and Weighted Portf Returns for SWIX Index (J433)


## I separately load the data for the property index (PROP)

"C:/Users/tianc/Rproj/FinMetrics871_Project/19025831/data/Alsi_Returns.rds" |>  read_rds() -> df



# I first shrink the dataframe to include only what in needed

T40_a <- df |> select(date, Tickers, Return, Index_Name, J433, J200, J202) |> 
    
    mutate(Tickers = gsub(" SJ Equity", "", Tickers))  # Remove clutter in Tickers names


T40_b <- T40 |> select(-Short.Name) |> 
    
    mutate(Tickers = gsub(" SJ Equity", "", Tickers))  # Remove clutter in Tickers names
```

# Portfolio Construction and Sector Calculations

## SWIX, ALSI and PROP weighted portfolio cumulative returns

I first plot the ALSI and SWIX weighted portfolio cumulative returns,
following some tedious data wrangling.

``` r
# I generate a tbl calculating both Indexes weighted returns by hand

df_Port_ret <- T40_a |> 
    
    mutate(J433 = coalesce(J433, 0)) |> 
    
    mutate(J200 = coalesce(J200, 0)) |>
    
    mutate(J202 = coalesce(J202, 0)) |>
    
    mutate(ALSI_wret = Return*J200) |> 
    
    mutate(SWIX_wret = Return*J433) |>
    
    mutate(SMALLCAP_wret = Return*J202) |>
    
    arrange(date) |> 
    
    group_by(date) |> 
    
    mutate(ALSI_pret = sum(ALSI_wret, na.rm = T)) |> 
    
    mutate(SWIX_pret = sum(SWIX_wret, na.rm = T)) |> 
    
    mutate(SMALLCAP_pret = sum(SMALLCAP_wret, na.rm = T))

df_Port_ret_B <-  T40_b |> 
    
    mutate(J400 = coalesce(J400, 0)) |> 
    
    mutate(J400 = coalesce(J400, 0)) |> 
    
    mutate(ALSI_wret = Return*J200) |> 
    
    mutate(SWIX_wret = Return*J400) |> 
    
    arrange(date) |> 
    
    group_by(date) |> 
    
    mutate(ALSI_pret = sum(ALSI_wret)) |> 
    
    mutate(SWIX_pret = sum(SWIX_wret)) 

 




## Calculate the weight of each property security on each day. And then calculate the UNCAPPED PORTFOLIO RETURN FOR THIS PROPERTY INDEX.

df_property <- df  |>  
    filter(Sector %in% "Property") |> 
    select(date, Tickers, Return, Index_Name, Market.Cap) |> 
    group_by(date) |> mutate(weight = Market.Cap / sum(Market.Cap)) |> 
    arrange(desc(weight)) |> ungroup() |> 
    mutate(weight = coalesce(weight, 0)) |>
    mutate(PROP_wret = Return*weight) |>            # Here I calculate the weighted returns for each porperty security
    arrange(date) |> 
    group_by(date) |> 
    mutate(PROP_pret = sum(PROP_wret, na.rm = T)) |>
    mutate(Tickers = gsub(" SJ Equity", "", Tickers)) 
  

    
## NB ---> Test if the weights on each day sums to one:  df_property |> group_by(date) |> summarise(sum(weight)). (Success)


# And now I merge the (uncapped) weighted portfolio return

df_Port_ret1 <- df_Port_ret |> select(date, ALSI_pret, SWIX_pret, SMALLCAP_pret) |> unique() |> left_join(df_property |> select(date, PROP_pret) |> unique(), by  = "date" )
```

After double checking the weighted portfolio returns of the SWIX, ALSI,
and PROP indexes using the Safe.Return function, I merge the ALSI and
SWIX, and the PROP indexes weighted portfolio return tibbles.

``` r
# Lets calculate the weighted portfolio daily return for ALSI and SWIX using Safe_Returns to verify my by hand calculation

Wghts_ALSI_xts <- T40_a |> select(date, Tickers , J200) |> tbl_xts(cols_to_xts = J200, spread_by = Tickers)

Wghts_SWIX_xts <- T40_a |> select(date, Tickers , J433) |> tbl_xts(cols_to_xts = J433, spread_by = Tickers)

Returns_xts <- T40_a |> select(date, Tickers , Return) |> tbl_xts(cols_to_xts = Return, spread_by = Tickers)

# Set NA's to null to use PA's Safe_returns.Portfolio command

Wghts_ALSI_xts[is.na(Wghts_ALSI_xts)] <- 0
Wghts_SWIX_xts[is.na(Wghts_SWIX_xts)] <- 0
Returns_xts[is.na(Returns_xts)] <- 0

# Now I calculate the weighed (uncapped) portfolio returns

Port_Ret_ALSI <- rmsfuns::Safe_Return.portfolio(R = Returns_xts, weights = Wghts_ALSI_xts, lag_weights = T) |> 
    
                 xts_tbl() |> rename(ALSI_Ret = portfolio.returns)

Port_Ret_SWIX <- rmsfuns::Safe_Return.portfolio(R = Returns_xts, weights = Wghts_SWIX_xts, lag_weights = T) |> 
    
                 xts_tbl() |> rename(SWIX_Ret = portfolio.returns)

# Now I combine the above two weighted portfolio returns

Merged_Port_Ret <- inner_join(Port_Ret_ALSI, Port_Ret_SWIX, by= "date") |> inner_join(df_property |> select(date, PROP_pret) |> unique(), by = "date")

# I verify that my by hand calc and the Safe_return calc is the same:
# df_Port_ret |> select(date, ALSI_pret, SWIX_pret) |> group_by(date, ALSI_pret, SWIX_pret ) |> summarise()
# Happy ---> They are the same
```

``` r
# Now I proceed to calculate the Portfolios' cumulative return and plot it

Cum_ret <- df_Port_ret1 |> arrange(date) |> as_tibble() |> 
    
    mutate(across(.cols = -date, .fns = ~cumprod(1 + .))) |> 
    
    mutate(across(.cols = -date, .fns = ~./first(.))) |> # Start at 1
    
    rename("Top40(J200)" = ALSI_pret, "SWIX(J433)" = SWIX_pret, "SMLC(J202)" = SMALLCAP_pret ,PROP = PROP_pret) |> 
    
    pivot_longer(cols=-date, names_to = "Index", values_to = "Cumret")
    
# And finally the plot

Indexes_Cum_ret_plot <-    Cum_ret |> 
       
       ggplot() + 
  
  geom_line(aes(date, Cumret , color = Index), size = 0.6, alpha = 0.7) +
    
    
    
    annotate("rect", xmin = lubridate::ymd("20130301"), xmax = lubridate::ymd("20161201"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
    
     annotate("rect", xmin = lubridate::ymd("20071201"), xmax = lubridate::ymd("20090801"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
     annotate("rect", xmin = lubridate::ymd("20190101"), xmax = lubridate::ymd("20200701"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
  
   fmxdat::theme_fmx(title.size = fmxdat::ggpts(30), 
                    subtitle.size = fmxdat::ggpts(0),
                    caption.size = fmxdat::ggpts(25),
                    CustomCaption = T) + 
    
  fmxdat::fmx_cols() + 
  
  labs(x = "", y = "%", caption = "Notes: The blue shaded areas reflect SA's economic recessions as defined by the OECD.",
       title = "Cumulative Returns of ALSI and SWIX Indexes",
       subtitle = "")

# Finplot for finishing touches:

fmxdat::finplot(Indexes_Cum_ret_plot, x.vert = T, x.date.type = "%Y", x.date.dist = "1 year", darkcol = T)
```

    ## Scale for colour is already present.
    ## Adding another scale for colour, which will replace the existing scale.

![](README_files/figure-gfm/Portfolios%20cumulative%20returns-1.png)<!-- -->

From the plot above, the cumulative weighted returns for the ALSI and
SWIX indexes are strickingly similar, however, since the onset of
COVID-19, the ALSI has achieved higher returns.

## Weighted return contribition of each sector

I now create the final tbl that includes the weighted return
contribution of each sector to the overall weighted portfolio return
(daily). From the plots produced below, the ALSI has a over time applied
a larger weight to the Resources sector than the SWIX, where the SWIX
applies relatively larger weights to Financials and Industrials.

The weighted contribution per sector confirms that the additional weight
the ALSI applies to the Resources sector compared to the SWIX is what
generated its higher returns since the onset of COVID-19.

``` r
# Calculated the weighted contribution of each sector by summing the individual security weights over Financials, Industrials, and Resources. 

df_Port_ret_final <- df_Port_ret_B |> arrange(date) |> 
    
    group_by(date, Sector) |>       # Group by Market Cap to calc each category's contr to wieghted portf return
    
    mutate(ALSI_wght_sector = coalesce(J400, 0)) |> # Make NA's 0 to use PA later
    
    mutate(SWIX_wght_sector = coalesce(J200, 0)) |>
    
    mutate(ALSI_wret = coalesce(ALSI_wret, 0)) |> 
    
    mutate(SWIX_wret = coalesce(SWIX_wret, 0)) |>
    
    mutate(ALSI_ret_sector = sum(ALSI_wret, na.rm = T)) |>  # The weight-contribution of each sector on each day
    
    mutate(SWIX_ret_sector = sum(SWIX_wret, na.rm = T)) |>

    mutate(ALSI_wght_sector = sum(J400, na.rm = T)) |>  # The weighted return contribution of each sector on each day
    
    mutate(SWIX_wght_sector = sum(J200, na.rm = T)) |>
    
    ungroup()

ALSI_wght_sector <-  df_Port_ret_final |> select(date, Sector ,ALSI_wght_sector) |> group_by(Sector) |> unique() |> 
    tbl_xts(cols_to_xts = ALSI_wght_sector, spread_by = Sector)

SWIX_wght_sector <- df_Port_ret_final |> select(date, Sector ,SWIX_wght_sector) |> group_by(Sector) |> unique() |> 
    tbl_xts(cols_to_xts = SWIX_wght_sector, spread_by = Sector)

ALSI_ret_sector <- df_Port_ret_final |> select(date, Sector ,ALSI_ret_sector) |> group_by(Sector) |> unique() |> 
    tbl_xts(cols_to_xts = ALSI_ret_sector, spread_by = Sector)

SWIX_ret_sector <- df_Port_ret_final |> select(date, Sector ,SWIX_ret_sector) |> group_by(Sector) |> unique() |> 
    tbl_xts(cols_to_xts = SWIX_ret_sector, spread_by = Sector)



 ALSI_RetPort_sector <- 
      rmsfuns::Safe_Return.portfolio(ALSI_ret_sector, 
                                     
                       weights = ALSI_wght_sector, lag_weights = TRUE,
                       
                       verbose = TRUE, contribution = TRUE, 
                       
                       value = 1000, geometric = TRUE) 

 SWIX_RetPort_sector <- 
      rmsfuns::Safe_Return.portfolio(SWIX_ret_sector, 
                                     
                       weights = SWIX_wght_sector, lag_weights = TRUE,
                       
                       verbose = TRUE, contribution = TRUE, 
                       
                       value = 1000, geometric = TRUE)
    

    
 
    
 
 

 ALSI_RetPort_sector$BOP.Weight  %>% .[endpoints(.,'months')] %>% chart.StackedBar()
```

![](README_files/figure-gfm/Portfolio%20Returns%20subdivided%20into%20Market%20Caps%20Contributions-1.png)<!-- -->

``` r
  SWIX_RetPort_sector$BOP.Weight  %>% .[endpoints(.,'months')] %>% chart.StackedBar()
```

![](README_files/figure-gfm/Portfolio%20Returns%20subdivided%20into%20Market%20Caps%20Contributions-2.png)<!-- -->

``` r
  ALSI_RetPort_sector$contribution |> chart.CumReturns(legend.loc = "bottom")
```

![](README_files/figure-gfm/Portfolio%20Returns%20subdivided%20into%20Market%20Caps%20Contributions-3.png)<!-- -->

``` r
  SWIX_RetPort_sector$contribution |> chart.CumReturns(legend.loc = "bottom")
```

![](README_files/figure-gfm/Portfolio%20Returns%20subdivided%20into%20Market%20Caps%20Contributions-4.png)<!-- -->

Pull the rebalance days,

``` r
# I firt pull the effective rebalance dates

Rebalance_Days <-RebDays |> filter(Date_Type %in% c("Effective Date")) |> pull(date)
    
# And now for both Indexes I create a capped weights tbl for rebalancing purposes

rebalance_col_ALSI <- df_Port_ret |> 
    
    filter(date %in% Rebalance_Days) |> 
    
    select(date, Tickers, J200) |> 
    
    rename(weight = J200) |> 
    
    mutate(RebalanceTime = format(date, "%Y_%b")) |> 
    
    mutate(weight= coalesce(weight, 0))
    
 rebalance_col_SWIX <- df_Port_ret |> 
    
    filter(date %in% Rebalance_Days) |> 
    
    select(date, Tickers, J433) |> 
     
     rename(weight = J433) |> 
     
     mutate(RebalanceTime = format(date, "%Y_%b")) |> 
     
      mutate(weight= coalesce(weight, 0))
 
## For SMALL CAPS
 
 rebalance_col_SMALLCAP <- df_Port_ret |> 
    
    filter(date %in% Rebalance_Days) |> 
    
    select(date, Tickers, J202) |> 
     
     rename(weight = J202) |> 
     
     mutate(RebalanceTime = format(date, "%Y_%b")) |> 
     
      mutate(weight= coalesce(weight, 0))
 
 
## And for the df_property
 
  rebalance_col_PROP <- df_property |> 
    
    filter(date %in% Rebalance_Days) |> 
    
    select(date, Tickers, weight) |> 
     
     mutate(RebalanceTime = format(date, "%Y_%b")) |> 
     
      mutate(weight= coalesce(weight, 0))
```

``` r
###### This function applies a capping on the weights. #########

Proportional_Cap_Foo <- function(df_Cons, W_Cap = 0.08){
  
  # Let's require a specific form from the user... Alerting when it does not adhere this form
  if( !"weight" %in% names(df_Cons)) stop("... for Calc capping to work, provide weight column called 'weight'")
  
  if( !"date" %in% names(df_Cons)) stop("... for Calc capping to work, provide date column called 'date'")
  
  if( !"Tickers" %in% names(df_Cons)) stop("... for Calc capping to work, provide id column called 'Tickers'")

  # First identify the cap breachers...
  Breachers <- 
    df_Cons %>% filter(weight > W_Cap) %>% pull(Tickers)
  
  # Now keep track of breachers, and add to it to ensure they remain at 10%:
  if(length(Breachers) > 0) {
    
    while( df_Cons %>% filter(weight > W_Cap) %>% nrow() > 0 ) {
      
      
      df_Cons <-
        
        bind_rows(
          
          df_Cons %>% filter(Tickers %in% Breachers) %>% mutate(weight = W_Cap),
          
          df_Cons %>% filter(!Tickers %in% Breachers) %>% 
            mutate(weight = (weight / sum(weight, na.rm=T)) * (1-length(Breachers)*W_Cap) )
          
        )
      
      Breachers <- c(Breachers, df_Cons %>% filter(weight > W_Cap) %>% pull(Tickers))
      
    }

    if( sum(df_Cons$weight, na.rm=T) > 1.001 | sum(df_Cons$weight, na.rm=T) < 0.999 | max(df_Cons$weight, na.rm = T) > W_Cap) {
      
      stop( glue::glue("For the Generic weight trimming function used: the weight trimming causes non unit 
      summation of weights for date: {unique(df_Cons$date)}...\n
      The restriction could be too low or some dates have extreme concentrations...") )
      
    }
    
  } else {
    
  }
  
  df_Cons
  
  }
  

# Now, to map this across all the dates, I purrr::map_df 
Capped_ALSI_10 <- 
    
    rebalance_col_ALSI |> 

    group_split(RebalanceTime) |> 
    
    map_df(~Proportional_Cap_Foo(., W_Cap = 0.1) ) |>  select(-RebalanceTime)
  
# Now I do the same for a 6% cap:

Capped_ALSI_6 <- 
    
    rebalance_col_ALSI |> 

    group_split(RebalanceTime) |> 
    
    map_df(~Proportional_Cap_Foo(., W_Cap = 0.06) ) |>  select(-RebalanceTime)

Capped_SWIX_10 <- 
    
    rebalance_col_ALSI |> 

    group_split(RebalanceTime) |> 
    
    map_df(~Proportional_Cap_Foo(., W_Cap = 0.1) ) |>  select(-RebalanceTime)
  
Capped_SWIX_6 <- 
    
    rebalance_col_ALSI |> 

    group_split(RebalanceTime) |> 
    
    map_df(~Proportional_Cap_Foo(., W_Cap = 0.06) ) |>  select(-RebalanceTime)


Capped_PROP_10 <- 
    
    rebalance_col_PROP |> 

    group_split(RebalanceTime) |> 
    
    map_df(~Proportional_Cap_Foo(., W_Cap = 0.15) ) |>  select(-RebalanceTime)


Capped_SMALLCAP_10 <- 
    
    rebalance_col_SMALLCAP |> 

    group_split(RebalanceTime) |> 
    
    map_df(~Proportional_Cap_Foo(., W_Cap = 0.1) ) |>  select(-RebalanceTime)



# # Testing if the max weight is correct for all 4 tbl above

Capped_ALSI_10 %>% pull(weight) %>% max(.) 
```

    ## [1] 0.1

``` r
Capped_ALSI_6 %>% pull(weight) %>% max(.) 
```

    ## [1] 0.06

``` r
Capped_SWIX_10 %>% pull(weight) %>% max(.) 
```

    ## [1] 0.1

``` r
Capped_SWIX_6 %>% pull(weight) %>% max(.) # Success!!
```

    ## [1] 0.06

``` r
Capped_PROP_10 %>% pull(weight) %>% max(.)
```

    ## [1] 0.15

## 6% and 10% Capped indexes weighted returns

I now analyse the impact different capping levels would have had on both
the SWIX and ALSI (6% and 10%).

From the ALSI capped figure, a capping of 10% has virtually the same
cumulative returns as when uncapped, with significantly lower returns
when capped at 6%. This is likely due to this 6% capping most of the
relatively higher returns generated from the Resources sector.

On the other hand, from the SWIX capped figure, the 10% cap would have
improved its return since the onset of COVID, likely due to a reduction
in weighted contribution from the weaker performing Industrial and
Financial sectors.

``` r
####For ALSI capped at 10%#####

wghts_ALSI_10 <- 
  Capped_ALSI_10 %>% 
  tbl_xts(cols_to_xts = weight, spread_by = Tickers)

ret_ALSI_10 <- 
  df_Port_ret %>% 
  
  filter(Tickers %in% unique(Capped_ALSI_10$Tickers) ) %>% 
  
  tbl_xts(cols_to_xts = Return, spread_by = Tickers)

wghts_ALSI_10[is.na(wghts_ALSI_10)] <- 0

ret_ALSI_10[is.na(ret_ALSI_10)] <- 0

ALSI_10_Idx <- 
  rmsfuns::Safe_Return.portfolio(R = ret_ALSI_10, weights = wghts_ALSI_10, lag_weights = T) |> 
  
  # Then I make it a tibble:
  xts_tbl() |>  
  
  rename(ALSI_10_Idx = portfolio.returns)

####For ALSI capped at 6%#####

wghts_ALSI_6 <- 
  Capped_ALSI_6 %>% 
  tbl_xts(cols_to_xts = weight, spread_by = Tickers)

ret_ALSI_6 <- 
  df_Port_ret %>% 
  
  filter(Tickers %in% unique(Capped_ALSI_6$Tickers) ) %>% 
  
  tbl_xts(cols_to_xts = Return, spread_by = Tickers)

wghts_ALSI_6[is.na(wghts_ALSI_6)] <- 0

ret_ALSI_6[is.na(ret_ALSI_6)] <- 0

ALSI_6_Idx <- 
  rmsfuns::Safe_Return.portfolio(R = ret_ALSI_6, weights = wghts_ALSI_6, lag_weights = T) |> 
  
  # Then I make it a tibble:
  xts_tbl() |>  
  
  rename(ALSI_6_Idx = portfolio.returns)

####For SWIX capped at 10%#####

wghts_SWIX_10 <- 
  Capped_SWIX_10 %>% 
  tbl_xts(cols_to_xts = weight, spread_by = Tickers)

ret_SWIX_10 <- 
  df_Port_ret %>% 
  
  filter(Tickers %in% unique(Capped_SWIX_10$Tickers) ) %>% 
  
  tbl_xts(cols_to_xts = Return, spread_by = Tickers)

wghts_SWIX_10[is.na(wghts_SWIX_10)] <- 0

ret_SWIX_10[is.na(ret_SWIX_10)] <- 0

SWIX_10_Idx <- 
  rmsfuns::Safe_Return.portfolio(R = ret_SWIX_10, weights = wghts_SWIX_10, lag_weights = T) |> 
  
  # Then I make it a tibble:
  xts_tbl() |>  
  
  rename(SWIX_10_Idx = portfolio.returns)



####For SWIX capped at 6%#####

wghts_SWIX_6 <- 
  Capped_SWIX_6 %>% 
  tbl_xts(cols_to_xts = weight, spread_by = Tickers)

ret_SWIX_6 <- 
  df_Port_ret %>% 
  
  filter(Tickers %in% unique(Capped_SWIX_6$Tickers) ) %>% 
  
  tbl_xts(cols_to_xts = Return, spread_by = Tickers)

wghts_SWIX_6[is.na(wghts_SWIX_6)] <- 0

ret_SWIX_6[is.na(ret_SWIX_6)] <- 0

SWIX_6_Idx <- 
  rmsfuns::Safe_Return.portfolio(R = ret_SWIX_6, weights = wghts_SWIX_6, lag_weights = T) |> 
  
  # Then I make it a tibble:
  xts_tbl() |>  
  
  rename(SWIX_6_Idx = portfolio.returns)

##### For PROP capped at 10% #####

wghts_PROP_10 <- 
  Capped_PROP_10 %>% 
  tbl_xts(cols_to_xts = weight, spread_by = Tickers)

ret_PROP_10 <- 
  df_Port_ret %>% 
  
  filter(Tickers %in% unique(Capped_PROP_10$Tickers) ) %>% 
  
  tbl_xts(cols_to_xts = Return, spread_by = Tickers)

wghts_PROP_10[is.na(wghts_PROP_10)] <- 0

ret_PROP_10[is.na(ret_PROP_10)] <- 0

PROP_10_Idx <- 
  rmsfuns::Safe_Return.portfolio(R = ret_PROP_10, weights = wghts_PROP_10, lag_weights = T) |> 
  
  # Then I make it a tibble:
  xts_tbl() |>  
  
  rename(PROP_10_Idx = portfolio.returns)


### For J202 SMALLCAP capped at 10%##

wghts_SMALLCAP_10 <- 
  Capped_SMALLCAP_10 %>% 
  tbl_xts(cols_to_xts = weight, spread_by = Tickers)

ret_SMALLCAP_10 <- 
  df_Port_ret %>% 
  
  filter(Tickers %in% unique(Capped_SMALLCAP_10$Tickers) ) %>% 
  
  tbl_xts(cols_to_xts = Return, spread_by = Tickers)

wghts_SMALLCAP_10[is.na(wghts_SMALLCAP_10)] <- 0

ret_SMALLCAP_10[is.na(ret_SMALLCAP_10)] <- 0

SMALLCAP_10_Idx <- 
  rmsfuns::Safe_Return.portfolio(R = ret_SMALLCAP_10, weights = wghts_SMALLCAP_10, lag_weights = T) |> 
  
  # Then I make it a tibble:
  xts_tbl() |>  
  
  rename(SMALLCAP_10_Idx = portfolio.returns)
```

``` r
Capped_df_final <- ALSI_10_Idx |> 
    inner_join(PROP_10_Idx, by ="date") |> 
    inner_join(SMALLCAP_10_Idx, by ="date") |> 
    arrange(date) |> 
    mutate(across(.cols = -date, .fns = ~cumprod(1+.))) |> # cumulative returns
    mutate(across(.cols = -date, .fns = ~./first(.))) |>   # Start at 1
    left_join(Cum_ret |> filter(Index %in% "SWIX(J433)") |> pivot_wider(names_from = "Index", values_from = "Cumret"), by = "date") |> 
    
    
    
    rename("Top40(J200)" = ALSI_10_Idx , "PROP" = PROP_10_Idx , "SMLC(J202)" = SMALLCAP_10_Idx) |> # rename for clarity
    
    pivot_longer(cols = -date, names_to = "Description", values_to = "Values")

# And now, at last, for the plot


capping_plot_all <- Capped_df_final |>  ggplot() + 
  
  geom_line(aes(date, Values , color = Description), size = 0.6, alpha = 0.7) +
    
    annotate("rect", xmin = lubridate::ymd("20130301"), xmax = lubridate::ymd("20161201"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
    
     annotate("rect", xmin = lubridate::ymd("20071201"), xmax = lubridate::ymd("20090801"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
     annotate("rect", xmin = lubridate::ymd("20190101"), xmax = lubridate::ymd("20200701"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
  
   fmxdat::theme_fmx(title.size = fmxdat::ggpts(30), 
                    subtitle.size = fmxdat::ggpts(0),
                    caption.size = fmxdat::ggpts(25),
                    CustomCaption = T) + 
    
  fmxdat::fmx_cols() + 
  
  labs(x = "", y = "%", caption = "Note: Calculation own. The blue shaded areas reflect SA's economic recessions as defined by the OECD.",
       title = "Cumulative Returns of SWIX, ALSI, and PROP Indices Uncapped and Capped at 10%",
       subtitle = "")

# Finplot for finishing touches:

fmxdat::finplot(capping_plot_all, x.vert = T, x.date.type = "%Y", x.date.dist = "1 year", darkcol = F)
```

![](README_files/figure-gfm/Plot%20the%20capped%20indexes-1.png)<!-- -->

# The DCC GARCH Model

## Loading data

``` r
pacman::p_load("MTS", "robustbase","fGarch")
pacman::p_load("tidyverse", "devtools", "rugarch", "rmgarch", 
    "forecast", "tbl2xts", "lubridate", "PerformanceAnalytics", 
    "ggthemes", "MTS")



df_DCC <- Capped_df_final |> 
    pivot_wider(names_from = "Description", values_from = "Values") |> 
    pivot_longer(cols = -date,names_to = "Index", values_to = "Cum_ret")
```

## Sample SD comparison of selected indexes

To get an initial comparison of volatility of the selected indexes, I
plot their sample standard deviation in log growth. I also filter the
dates to consider the period from 2006 onwards to remove extremely
volatile periods.

``` r
SD_plot_data <- df_DCC |>  arrange(date) |> 
    
    group_by(Index) |> 
    
    mutate(Growth = log(Cum_ret) - lag(log(Cum_ret))) |> 
    
    filter(date > dplyr::first(date)) |>  
    
    mutate(scaledgrowth = Growth - mean(Growth, rm.na = T)) |>     # Scale the Growth by demeaning
    
    mutate(SampleSD = (sqrt(scaledgrowth^2))) |> 
    
    ungroup() 




Scaledgrowth_plot_df <-  SD_plot_data |> 
 
    ggplot() + 
  
  geom_line(aes(date, scaledgrowth , color = Index), size = 0.6, alpha = 0.7) +
    
    facet_wrap(~Index, scales = "free_y")+
    
    
    
    
    
    annotate("rect", xmin = lubridate::ymd("20130301"), xmax = lubridate::ymd("20161201"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
    
     annotate("rect", xmin = lubridate::ymd("20071201"), xmax = lubridate::ymd("20090801"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
     annotate("rect", xmin = lubridate::ymd("20190101"), xmax = lubridate::ymd("20200701"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
  
   fmxdat::theme_fmx(title.size = fmxdat::ggpts(30), 
                    subtitle.size = fmxdat::ggpts(0),
                    caption.size = fmxdat::ggpts(25),
                    CustomCaption = T) + 
    
  fmxdat::fmx_cols() + 
  
  labs(x = "", y = "%", caption = "Note: Calculation own. The blue shaded areas reflect SA's economic recessions as defined by the OECD.",
       title = "Scaled (demeaned) Log Growth of Respective Currencies to USD since 2005.",
       subtitle = "")

SD_plot <- SD_plot_data |>  

  ggplot() + 
  
   geom_line(aes(date, SampleSD , color = Index), size = 0.6, alpha = 0.7) +
    
    
    annotate("rect", xmin = lubridate::ymd("20130301"), xmax = lubridate::ymd("20161201"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
    
     annotate("rect", xmin = lubridate::ymd("20071201"), xmax = lubridate::ymd("20090801"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
     annotate("rect", xmin = lubridate::ymd("20190101"), xmax = lubridate::ymd("20200701"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
    
    facet_wrap(~Index, scales = "free_y")+
  
   fmxdat::theme_fmx(title.size = fmxdat::ggpts(30), 
                    subtitle.size = fmxdat::ggpts(0),
                    caption.size = fmxdat::ggpts(25),
                    CustomCaption = T) + 
    
  fmxdat::fmx_cols() + 
  
  labs(x = "", y = "%", caption = "Note: Calculation own. The blue shaded areas reflect SA's economic recessions as defined by the OECD.",
       title = "Sample Standard Deviation of Respective Currencies to USD since 2005.",
       subtitle = "")

# Finplot for finishing touches:



fmxdat::finplot(Scaledgrowth_plot_df, x.vert = T, x.date.type = "%Y", x.date.dist = "2 years", darkcol = T)
```

    ## Scale for colour is already present.
    ## Adding another scale for colour, which will replace the existing scale.

![](README_files/figure-gfm/Scaledgrowth%20and%20standard%20deviation%20plots-1.png)<!-- -->

``` r
fmxdat::finplot(SD_plot, x.vert = T, x.date.type = "%Y", x.date.dist = "2 years", darkcol = T)
```

    ## Scale for colour is already present.
    ## Adding another scale for colour, which will replace the existing scale.

![](README_files/figure-gfm/Scaledgrowth%20and%20standard%20deviation%20plots-2.png)<!-- -->

Table reports the coefficients $a$ and $b$’s estimates and corresponding
p-values. From @katzke2013, these estimates signify mean reversion of
the time-varying correlations since $a + b < 1$. The impact of lagged
standardised shocks on dynamic conditional correlations is measured by
the coefficient $a$. In contrast, the measure of the past effect of the
dynamic conditional correlations on present dynamic conditional
correlations is given by $b$. These parameters are, additionally,
statistically significant at the 5$\%$ level, except for the $a$ and $b$
coefficients for SMLC(J202), which is significant at the 10$\%$ level.
Again following @katzke2013, this indicates significant deviations over
time, reaffirming that a DCC model is more fitting than a CCC model.

The corresponding estimated DCC model diagnostics, checking for
conditional heteroscedasticity through testing for serial correlation,
is reported in Table below. As stated by @tsay2013, when the shocks are
heavy-tailed, the parameters $Q(m)$ and $Q_k(m)$ often fail to detect
the presence of conditional heteroscedasticity, and the $Q_k^r (m)$
robustness parameter is desirable. Consequently, the fitted DCC model
fails to reject the null of no autocorrelation when considering the
rank-based test and the robustness parameter $Q_k^r (m)$.

The model’s estimated volatility for each index (Figure ) shows that
PROP is the most volatile in comparison to the SMLC(J202) and SWIX(J433)
indexes, especially in the past decade, possessing substantially larger
jumps in volatility during recessionary periods (blue shaded areas).
Moreover, the SMLC(J202) index is the least volatile.

In analysing the estimated dynamic conditional correlations across index
pairs (Figure ), the PROP index exhibits a substantially lower
time-varying conditional correlation with the SWIX(J433) index compared
to the SMLC(J202) index. However, since 2016, this difference has shrunk
marginally. On the other hand, the dynamic conditional correlation
between the SMLC(J202) and SWIX(J433) indexes is the highest and the
more stable among the three index pairs considered.

## MV Conditional Heteroskedasticity tests

I conduct MV Portmanteau tests using MarchTest from MTS package.

``` r
######   ########

gwt1 <- SD_plot_data |> select(date, Index, Growth) |> filter(Index %in% c("SWIX(J433)", "PROP", "SMLC(J202)"))



# Change to xts format


gwt_xts1 <- gwt1 |> 
    
    tbl_xts(cols_to_xts = Growth, spread_by = Index)

# MV Portmanteau tests


MarchTest(gwt_xts1)
```

    ## Q(m) of squared series(LM test):  
    ## Test statistic:  5699.07  p-value:  0 
    ## Rank-based Test:  
    ## Test statistic:  2452.686  p-value:  0 
    ## Q_k(m) of squared series:  
    ## Test statistic:  9682.785  p-value:  0 
    ## Robust Test(5%) :  2047.903  p-value:  0

The MARCH test indicates that all the MV portmanteau tests reject the
null of no conditional heteroskedasticity, motivating our use of MVGARCH
models.

## DCC MV-GARCH MODEL

I decide to use a DCC MVGARCH Model; DCC models offer a simple and more
parsimonious means of doing MV-vol modelling. In particular, it relaxes
the constraint of a fixed correlation structure (assumed by the CCC
model), to allow for estimates of time-varying correlation.

``` r
# As in the tut, I select a VAR order of zero for the mean equation, and simply use the mean of each series.
# The mean equation is thus in our case simply: Growth = mean(Growth) + et

# Then, for every series, a standard univariate GARCH(1,1) is run - giving us:
# et and sigmat, which is then used to calculate the standardized resids, zt, which is used in DCC calcs after.

DCCPre <- dccPre(gwt_xts1, include.mean = T, p=0) # Find a nice way to put this in a table
```

    ## Sample mean of the returns:  0.0004078085 0.0005823262 0.0004718333 
    ## Component:  1 
    ## Estimates:  2e-06 0.147346 0.84056 
    ## se.coef  :  0 0.012458 0.011918 
    ## t-value  :  6.47192 11.8278 70.52736 
    ## Component:  2 
    ## Estimates:  2e-06 0.135447 0.83566 
    ## se.coef  :  0 0.014118 0.016533 
    ## t-value  :  5.724064 9.593628 50.54456 
    ## Component:  3 
    ## Estimates:  3e-06 0.099673 0.881356 
    ## se.coef  :  1e-06 0.009429 0.011151 
    ## t-value  :  5.07514 10.57106 79.04112

I now have the estimates of volatility for each series. Follow my lead
below in changing the output to a usable Xts series for each column in
xts_rtn:

``` r
Vol <- DCCPre$marVol

colnames(Vol) <- colnames(gwt_xts1)

Vol <- 
  data.frame( cbind( date = index(gwt_xts1), Vol)) |>  # Add date column which dropped away...
  mutate(date = as.Date(date)) |>  tibble::as_tibble()  # make date column a date column...



TidyVol <- Vol |>  pivot_longer(names_to = "Indexes", values_to =  "Sigma", cols =  -date)

TidyVol_plot <- TidyVol |> ggplot() + 
  
  geom_line(aes(date, Sigma , color = Indexes), size = 0.9, alpha = 0.6) +
    
    annotate("rect", xmin = lubridate::ymd("20130301"), xmax = lubridate::ymd("20161201"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
    
     annotate("rect", xmin = lubridate::ymd("20071201"), xmax = lubridate::ymd("20090801"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
     annotate("rect", xmin = lubridate::ymd("20190101"), xmax = lubridate::ymd("20200701"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
    
  
   fmxdat::theme_fmx(title.size = fmxdat::ggpts(30), 
                    subtitle.size = fmxdat::ggpts(0),
                    caption.size = fmxdat::ggpts(25),
                    CustomCaption = T) + 
    
  fmxdat::fmx_cols() + 
  
  labs(x = "", y = "Sigma", caption = "Note: Calculation own. The blue shaded areas reflect SA's economic recessions as defined by the OECD.",
       title = "DCC GARCH: Estimated Volatility (Sigma) for Each Currency",
       subtitle = "Notice this includes the Bloomberg Dollar Spot Index (BBDXY)")
    
# And finally touches with finplot    

fmxdat::finplot(TidyVol_plot, x.vert = T, x.date.type = "%Y", x.date.dist = "1 years", darkcol = F)
```

![](README_files/figure-gfm/unnamed-chunk-1-1.png)<!-- -->

## Calculating the DCC Model

I will now extract the estimated standardized residuals to calculate the
DCC Model.

``` r
# The standardized residuals

StdRes <- DCCPre$sresi

# I first do the detach trick from the tut:

pacman::p_load(tidyverse,fmxdat, rmsfuns, tbl2xts, tidyr, ggpubr, broom,rstatix, modelr )

detach("package:tidyverse", unload=TRUE)
detach("package:fmxdat", unload=TRUE)
detach("package:rmsfuns", unload=TRUE)
detach("package:tbl2xts", unload=TRUE)
detach("package:ggpubr", unload=TRUE)
detach("package:rstatix", unload=TRUE)
detach("package:modelr", unload=TRUE)
detach("package:broom", unload=TRUE)
detach("package:tidyr", unload=TRUE)
detach("package:dplyr", unload=TRUE)

DCC <- dccFit(StdRes,type = "Engle") 
```

    ## Estimates:  0.9357251 0.02997446 8.69082 
    ## st.errors:  0.01243048 0.004531957 0.4955076 
    ## t-values:   75.27664 6.614021 17.53923

``` r
pacman::p_load(tidyverse,fmxdat, rmsfuns, tbl2xts, tidyr, ggpubr, broom,rstatix, modelr )
```

``` r
Rhot <- DCC$rho.t
# Right, so it gives us all the columns together in the form:
# X1,X1 ; X1,X2 ; X1,X3 ; ....

# So, let's be clever about defining more informative col names. 
# I will create a renaming function below:
ReturnSeries = gwt_xts1
DCC.TV.Cor = Rhot

renamingdcc <- function(ReturnSeries, DCC.TV.Cor) {
  
ncolrtn <- ncol(ReturnSeries)
namesrtn <- colnames(ReturnSeries)
paste(namesrtn, collapse = "_")

nam <- c()
xx <- mapply(rep, times = ncolrtn:1, x = namesrtn)
# Now let's be creative in designing a nested for loop to save the names corresponding to the columns of interest.. 

# TIP: draw what you want to achieve on a paper first. Then apply code.

# See if you can do this on your own first.. Then check vs my solution:

nam <- c()
for (j in 1:(ncolrtn)) {
for (i in 1:(ncolrtn)) {
  nam[(i + (j-1)*(ncolrtn))] <- paste(xx[[j]][1], xx[[i]][1], sep="_")
}
}

colnames(DCC.TV.Cor) <- nam

# So to plot all the time-varying correlations wrt SBK:
 # First append the date column that has (again) been removed...
DCC.TV.Cor <- 
    data.frame( cbind( date = index(ReturnSeries), DCC.TV.Cor)) %>% # Add date column which dropped away...
    mutate(date = as.Date(date)) %>%  tbl_df() 

DCC.TV.Cor <- DCC.TV.Cor %>% gather(Pairs, Rho, -date)

DCC.TV.Cor

}

# Let's see if our function works! Excitement!
Rhot <- 
  renamingdcc(ReturnSeries = gwt_xts1, DCC.TV.Cor = Rhot)

head(Rhot %>% arrange(date))
```

    ## # A tibble: 6 × 3
    ##   date       Pairs                   Rho
    ##   <date>     <chr>                 <dbl>
    ## 1 2005-03-23 PROP_PROP             1    
    ## 2 2005-03-23 PROP_SMLC.J202.       0.528
    ## 3 2005-03-23 PROP_SWIX.J433.       0.458
    ## 4 2005-03-23 SMLC.J202._PROP       0.528
    ## 5 2005-03-23 SMLC.J202._SMLC.J202. 1    
    ## 6 2005-03-23 SMLC.J202._SWIX.J433. 0.590

The figure below points towards heterogeneity in the correlations
between the sector pairs over time and reveals that static estimates of
comovement (in modeling terms, the Constant Conditional Correlations or
CCC) might be misleading.

``` r
# Let's now create a plot for all the stocks relative to the other stocks...


DCC_plot_PROP <- Rhot|>  filter(!grepl("SWIX.J433._", Pairs ), !grepl("_PROP", Pairs),  !grepl("SMLC.J202._SMLC.J202.", Pairs) ) |>
    
    ggplot() +
    
    geom_line(aes(x = date, y = Rho, colour = Pairs),size = 0.4, alpha = 1 ) + 
    
    
    annotate("rect", xmin = lubridate::ymd("20130301"), xmax = lubridate::ymd("20161201"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
    
     annotate("rect", xmin = lubridate::ymd("20071201"), xmax = lubridate::ymd("20090801"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
     annotate("rect", xmin = lubridate::ymd("20190101"), xmax = lubridate::ymd("20200701"), ymin = -Inf, ymax = Inf,alpha = .2, fill='steelblue', alpha= 0.05) +
    
    facet_wrap(~Pairs) +
    
    fmxdat::theme_fmx(title.size = fmxdat::ggpts(30), 
                    subtitle.size = fmxdat::ggpts(0),
                    caption.size = fmxdat::ggpts(25),
                    CustomCaption = T) + 
    
  fmxdat::fmx_cols() + 
  
  labs(x = "", y = "Sigma", caption = "Note: Calculation own. The blue shaded areas reflect SA's economic recessions as defined by the OECD.",
       title = "Dynamic Conditional Correlations",
       subtitle = "")
    
# And finally touches with finplot    

fmxdat::finplot(DCC_plot_PROP, x.vert = T, x.date.type = "%Y", x.date.dist = "2 years", darkcol = T, legend_pos = "none", col.hue = 40)
```

    ## Scale for colour is already present.
    ## Adding another scale for colour, which will replace the existing scale.
    ## Scale for colour is already present.
    ## Adding another scale for colour, which will replace the existing scale.

![](README_files/figure-gfm/Plotting%20the%20dynamic%20conditional%20correlations%20-1.png)<!-- -->

Gather the coefficients $a$ and $b$’s estimates and standard error. From
Katzke (2013), indicates that the time-varying correlations are mean
reverting since a + b \< 1, for three indexes. The coefficient $a$
measures the effect of past standardised innovations on dynamic
conditional correlations, while $b$ report the impact of lagged dynamic
conditional correlations on the current dynamic conditional correlations
(Katzke 2013). In addition to this, the parameters are mostly
significant, indicating significant variation over the specified period.
More specifically, the statistical significance of a and b indicates
that a DCC model vis-a-vis CCC model is more suitable.

``` r
xtable(DCCPre$est) # Order is SWIX, SMALLCAP, and PROP
```

    ## % latex table generated in R 4.2.2 by xtable 1.8-4 package
    ## % Sat Feb  4 11:52:14 2023
    ## \begin{table}[ht]
    ## \centering
    ## \begin{tabular}{rrrr}
    ##   \hline
    ##  & omega & alpha1 & beta1 \\ 
    ##   \hline
    ## 1 & 0.00 & 0.15 & 0.84 \\ 
    ##   2 & 0.00 & 0.14 & 0.84 \\ 
    ##   3 & 0.00 & 0.10 & 0.88 \\ 
    ##    \hline
    ## \end{tabular}
    ## \end{table}

``` r
xtable(DCCPre$se.coef)
```

    ## % latex table generated in R 4.2.2 by xtable 1.8-4 package
    ## % Sat Feb  4 11:52:14 2023
    ## \begin{table}[ht]
    ## \centering
    ## \begin{tabular}{rrrr}
    ##   \hline
    ##  & omega & alpha1 & beta1 \\ 
    ##   \hline
    ## 1 & 0.00 & 0.01 & 0.01 \\ 
    ##   2 & 0.00 & 0.01 & 0.02 \\ 
    ##   3 & 0.00 & 0.01 & 0.01 \\ 
    ##    \hline
    ## \end{tabular}
    ## \end{table}
