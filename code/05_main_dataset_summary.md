Summary Statistics
================
Martin Kosík
3 dubna 2019

``` r
knitr::opts_chunk$set(echo = TRUE, fig.show = 'hide')
library(scales)
library(knitr)
library(kableExtra)
library(tidyverse)
library(here)
source(here::here("code/functions.R"))
```

``` r
min_by_year <-  read_csv(here::here("data/min_by_year_preds.csv")) %>% 
  mutate(log_n = log(1 + label ),
         log_n_pred_full = log(1 + label + pred_adj_full_scaled),
         log_n_pred = log(1 + label + prediction),
         log_n_imp_date = log(1 + label_imp_date),
         log_n_pred_full_imp_date = log(1 + label_imp_date + pred_adj_full_scaled_imp_date))
```

    ## Parsed with column specification:
    ## cols(
    ##   .default = col_integer(),
    ##   ethnicity = col_character(),
    ##   log_n = col_double(),
    ##   log_n_imp_date = col_double(),
    ##   log_n_rehab = col_double()
    ## )

    ## See spec(...) for full column specifications.

``` r
opts_current$set(label = "total_arrests_by_ethnicity")
min_by_year %>% 
  group_by(ethnicity) %>% 
  summarize(Arrests_label = sum(label_imp_date), Arrests_preds = sum(label_imp_date + prediction_imp_date),
            Arrests_preds_adj = sum(label_imp_date + pred_adj_scaled_imp_date),
            Arrests_preds_adj_full = sum(label_imp_date + pred_adj_full_scaled_imp_date)) %>% 
  arrange(desc(Arrests_label)) %>% 
  kable("latex", booktabs = T, caption = "Total arrest by ethnicity, 1921-1960", linesep = "",
        col.names = c("Ethnicity", "Only Labeled", "Labeled + Unadj. Imput.", 
                      "Labeled + Parsimon. Adj.", "Labeled + Full-matrix Adj."),format.args = list(big.mark = ""))%>%
  add_header_above(c(" " = 1, "Arrests" = 4)) %>% 
  kable_styling(font_size = 8, latex_options = c("hold_position")) %>%  
  write_file(here::here("tables/total_arrests_tab.tex"))
```

``` r
column_order <- c(1, seq(from = 2, by = 2, length.out = 5), seq(from = 3, by = 2, length.out = 5))

opts_current$set(label = "descr_stats_by_ethnicity")
min_by_year %>% 
  mutate(label_preds = label + prediction) %>% 
  group_by(ethnicity) %>% 
  summarize_at(dplyr::vars(label, label_preds), c("mean", "sd", "min", "max", "sum")) %>% 
  select(!!column_order) %>% 
  arrange(ethnicity) %>% 
  kable("latex", booktabs = T, caption = "Descriptive Statistics of Arrests from 1921 to 1960 by Ethnicity, Part 1",
        linesep = "",
        col.names = c("Ethnicity", "Mean", "St.dev.", "Min", "Max", "Total", 
                      "Mean", "St.dev.", "Min", "Max", "Total"), format.args = list(big.mark = " "), digits = 0) %>% 
  add_header_above(c(" " = 1, "Only labeled data" = 5, "Labels + Ethnicity imputations (no adj.)" = 5)) %>% 
  kable_styling(font_size = 8, latex_options = c("hold_position")) %>%  
  write_file(here::here("tables/descr_stats_by_ethnicity.tex"))
```

``` r
opts_current$set(label = "descr_stats_date_imp")
min_by_year %>% 
  mutate(label_preds_imp = label_imp_date + prediction_imp_date) %>% 
  group_by(ethnicity) %>% 
  summarize_at(dplyr::vars(label_imp_date, label_preds_imp), c("mean", "sd", "min", "max", "sum")) %>% 
  select(!!column_order) %>% 
  arrange(ethnicity) %>% 
  kable("latex", booktabs = T, caption = "Descriptive Statistics of Arrests from 1921 to 1960 by Ethnicity, Part 2",
        linesep = "",
        col.names = c("Ethnicity", "Mean", "St.dev.", "Min", "Max", "Total", 
                      "Mean", "St.dev.", "Min", "Max", "Total"), format.args = list(big.mark = " "), digits = 0) %>% 
  add_header_above(c(" " = 1, "Labels + Arrest date imputations" = 5,
                     "Labels + Arrest date + Ethnicity imput. (no adj.)" = 5)) %>% 
  kable_styling(font_size = 8, latex_options = c("hold_position")) %>%  
  write_file(here::here("tables/descr_stats_date_imp.tex"))
```

``` r
min_by_year %>% 
  transmute(label, pred_adj_full_scaled = label + pred_adj_full_scaled, 
            label_imp_date, pred_adj_full_scaled_imp_date = label_imp_date + pred_adj_full_scaled_imp_date) %>% 
  #dplyr::select(label,  pred_adj_full_scaled, label_imp_date, pred_adj_full_scaled_imp_date) %>% 
  gather(key = "key", value = "n") %>% 
  mutate(key = recode_factor(key, "label" = "no imputaion (only labeled data)",
                            "label_imp_date" =  "labels + arrest date imputation",
                            "pred_adj_full_scaled" = "labels + ethnicity imputation",
                            "pred_adj_full_scaled_imp_date" = "labels + ethnicity + arrest date imputation")) %>% 
  ggplot(aes(x = n)) + geom_histogram(col = "white") + theme_bw()+
  facet_wrap(~ key) + 
  scale_x_continuous(trans = "mylog10" , breaks = c(0,2,10,100,1000,10000))+
  labs(x = "Number arrests of each ethnic group for a given year"
       #caption = expression("Values on x-axis are plotted on the"~log[10](1 + x)~"scale.")
       ) + 
  theme(#axis.line = element_line(size = 1), 
       # panel.grid.major.x = element_blank(), 
        panel.grid.minor.x = element_blank(), 
       #strip.background = element_rect(colour="black"),
        text = element_text(size=12)) + expand_limits(x = -0.01)
```

    ## `stat_bin()` using `bins = 30`. Pick better value with `binwidth`.

``` r
ggsave(here::here("plots/arrests/facet_hist.pdf"))
```

    ## Saving 7 x 5 in image
    ## `stat_bin()` using `bins = 30`. Pick better value with `binwidth`.

Histogram of arrests by different imputations.

``` r
min_by_year %>% 
  dplyr::select(YEAR, ethnicity, label, label_imp_date) %>% 
  mutate(label_imp_date = label_imp_date -label) %>% 
  gather(key = "type", value = "arrests", -YEAR, -ethnicity) %>% 
  mutate(type = recode_factor(type, "label" = "Known year of arrest",
                                         "label_imp_date" = "Imputed year of arrest")) %>%   
  ggplot(aes(x = YEAR, y = arrests, col = type)) + geom_line() + facet_wrap(~ethnicity) +
  scale_y_continuous(trans = "mylog10" , breaks = c(0,10,100,1000,10000))+
  scale_x_continuous(breaks = c(1920, 1930, 1940, 1950)) + 
  labs(y = "Number of Arrests", x = "Year", col = "Type",
       caption = expression("Values on y-axis are plotted on the"~log[10](1 + y)~"scale.")) + theme_bw() +
  theme(#axis.line = element_line(size = 1), 
       # panel.grid.major.x = element_blank(), 
        panel.grid.minor.y = element_blank(), 
        legend.position="bottom", 
        text = element_text(size=12))
```

``` r
ggsave(here::here("plots/arrests/facet_by_year.pdf"))
```

    ## Saving 7 x 5 in image

Visualisation of arrests by year by prediction adjustment.

``` r
min_by_year %>% 
  dplyr::select(YEAR, ethnicity, prediction, pred_adj_scaled, pred_adj_full_scaled) %>% 
  gather(key = "prediction_type", value = "arrests", -YEAR, -ethnicity) %>% 
  mutate(prediction_type = recode_factor(prediction_type, "prediction" = "No adjustment",
                                         "pred_adj_scaled" = "Parsimonious adj.", 
                                         "pred_adj_full_scaled" = "Full matrix adj.")) %>% 
  ggplot(aes(x = YEAR, y = arrests, col = prediction_type)) + geom_line() + facet_wrap(~ethnicity) +
  scale_y_continuous(trans = "mylog10" , breaks = c(0,10,100,1000,10000))+
  scale_x_continuous(breaks = c(1920, 1930, 1940, 1950)) + 
  labs(y = "Number of Arrests", x = "Year", 
       caption = expression("Values on y-axis are plotted on the"~log[10](1 + y)~"scale."),
       col = "Prediction type") + theme_bw() +
  theme(#axis.line = element_line(size = 1), 
       # panel.grid.major.x = element_blank(), 
        panel.grid.minor.y = element_blank(), 
        legend.position="bottom", 
        text = element_text(size=12))
```

``` r
ggsave(here::here("plots/arrests/prediction_type_by_year.pdf"))
```

    ## Saving 7 x 5 in image
