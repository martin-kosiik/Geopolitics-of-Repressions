Creating the main dataset
================
Martin Kosík
February 27, 2019

``` r
knitr::opts_chunk$set(echo = TRUE, fig.show = 'hide')
library(data.table)
library(tidyverse)
library(here)
library(lubridate)
library(scales)
library(knitr)
library(kableExtra)
source(here::here("code/functions.R"))
```

Load, pre-process and join all necessary datasets

``` r
events_preds <- fread(here::here("data/events_predictions.csv"), encoding = "UTF-8") %>% 
  dplyr::select(person_id, ethnicity, prediction)


selected_vars <- c("person_id", "arest_date", "process_date", "rehabilitation_date", "nation")
events <- fread(here::here("memo_list/memorial_lists.tsv"), encoding="UTF-8", sep="\t", 
                        select = selected_vars, quote="")
events <- events %>% 
  left_join(events_preds, "person_id") %>% 
  mutate(rehabilitation = ifelse(rehabilitation_date == "None", 0, 1))


events <- events %>% 
  separate(arest_date, sep = "\\.", into = c("DAY", "MONTH", "YEAR"), remove = FALSE) %>% 
  separate(process_date, sep = "\\.", into = c("DAY_PROCESS", "MONTH_PROCESS", "YEAR_PROCESS"), remove = FALSE) %>%
  mutate_at(c("DAY", "MONTH", "YEAR", "DAY_PROCESS", "MONTH_PROCESS", "YEAR_PROCESS"),
             funs(as.numeric(ifelse(. %in% c("None", "_"), NA, .))))
```

    ## Warning: Expected 3 pieces. Missing pieces filled with `NA` in 1650941
    ## rows [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19,
    ## 20, ...].

    ## Warning: Expected 3 pieces. Missing pieces filled with `NA` in 943124 rows
    ## [105, 395, 454, 497, 549, 769, 8992, 9011, 9014, 9021, 9026, 9028, 9029,
    ## 9040, 9047, 9060, 9071, 9076, 9080, 9102, ...].

``` r
saveRDS(events, file = here::here("data/eventsClean_v3.RData"))

events <- readRDS(here::here("data/eventsClean_v3.RData"))

rm(events_preds)
gc()
```

    ##            used  (Mb) gc trigger   (Mb)  max used  (Mb)
    ## Ncells   895841  47.9    3952167  211.1   7719078 412.3
    ## Vcells 51785783 395.1  140253675 1070.1 116686212 890.3

``` r
arrest_date_imputation <- readRDS(file = here::here("data/arrest_date_imputation.RData"))

border_arrests <- fread(here::here("data/border_arrests.csv"))

events <- events %>% 
  left_join(arrest_date_imputation, by = "person_id", suffix = c(".only_label", "")) %>% 
  left_join(border_arrests, by = "person_id")

rm(list = c("arrest_date_imputation", "border_arrests"))
```

Plot the arrests by year for data with and without imputated year of arrest

``` r
events %>% 
  count(YEAR, source_date_arrest, sort =T) %>% 
  filter(YEAR >= 1921, YEAR < 1961) %>% 
  mutate(source_date_arrest = recode_factor(source_date_arrest, "prediction" = "Only arrest date imputations",
                                            "label" = "Only labeled data (no imputation)")) %>% 
  ggplot(aes(x = YEAR, y = n, col = source_date_arrest)) + geom_line() +
  scale_y_continuous(trans = "mylog10" , breaks = c(0,10,100,1000,10000, 100000), labels = comma)+
  #scale_x_continuous(breaks = c(1920, 1930, 1940, 1950)) + 
  labs(y = "Number of Arrests", x = "Year", 
       #caption = expression("Values on y-axis are plotted on the"~log[10](1 + y)~"scale."),
       col = element_blank()) + theme_minimal() +
  theme(axis.line = element_line(size = 1), 
       # panel.grid.major.x = element_blank(), 
        panel.grid.minor.y = element_blank(), 
        legend.position="bottom", 
        text = element_text(size=12))
```

``` r
ggsave(here::here("plots/arrests/date_imputation_line.pdf"))
```

    ## Saving 7 x 5 in image

Load the confusion matrix and save the summary table

``` r
conf_matrix <- readRDS(here::here("data/conf_matrix.RData"))

spec_sens_data <- as_tibble(conf_matrix$byClass, rownames = "ethnicity") %>% 
  mutate(ethnicity = str_replace(ethnicity, "Class: ", "")) %>% 
  dplyr::select(ethnicity:Specificity)


conf_matrix <- conf_matrix$table

opts_current$set(label = "conf_matrix_count")
as_tibble(conf_matrix) %>% 
  spread(key = Reference, value = n) %>% 
  kable("latex", booktabs = T, linesep = "", longtable = T,
        caption = "Confusion Matrix (based on 10-fold cross-validation) - Counts") %>% 
  kable_styling(font_size = 5, latex_options = c("repeat_header")) %>% 
  add_header_above(c(" " = 1, "Reference" = 18)) %>% 
  landscape() %>% 
    write_file(here::here("tables/conf_matrix_count.tex"))


conf_matrix <- apply(conf_matrix, 2, function(x) x/sum(x))

opts_current$set(label = "conf_matrix_prop")
as_tibble(conf_matrix, rownames = "Prediction") %>% 
  kable("latex", booktabs = T, linesep = "", digits = 3, longtable = T,
        caption = "Confusion Matrix (based on 10-fold cross-validation) - Proportions") %>% 
  kable_styling(font_size = 5, latex_options = c("repeat_header")) %>% 
  add_header_above(c(" " = 1, "Reference" = 18)) %>% 
  landscape() %>% 
    write_file(here::here("tables/conf_matrix_prop.tex"))

opts_current$set(label = "sens_spec")
spec_sens_data %>% 
  rename(Ethnicity = ethnicity) %>% 
  kable("latex", booktabs = T, linesep = "", digits = 3, longtable = T,
        caption = "Naive Bayes Performance Measures by Ethnicity") %>% 
  kable_styling(font_size = 8, latex_options = c("repeat_header")) %>% 
    write_file(here::here("tables/sens_spec.tex"))
```

Define a function that creates the main dataset. First the arrests are counted, then the imputations are performed.

``` r
get_arrests_by_year <- function(data){

min_by_year_preds <- data %>% 
  count(YEAR, prediction, ethnicity) %>% 
  filter(YEAR >= 1921, !is.na(ethnicity), !is.na(YEAR), YEAR < 1961) %>% 
  complete(YEAR, ethnicity, prediction, fill = list(n = 0))


min_by_year_all <- min_by_year_preds %>% 
  mutate(prediction = ifelse(prediction == 1, "prediction", "label")) 


pred_adj_full_by_year <- min_by_year_all %>% 
  filter(prediction == "prediction") %>% 
  dplyr::select(-prediction) %>% 
  nest(-YEAR) %>% 
  mutate(pred_adj_full = map(data, ~ solve(conf_matrix, .$n))) %>% 
  unnest(pred_adj_full, data) %>% 
  mutate(pred_adj_full = ifelse(pred_adj_full < 0, 0, pred_adj_full)) %>% 
  dplyr::select(-n)
  
  

min_by_year_all <- min_by_year_all %>% 
  spread(key = prediction, value = n) %>% 
  left_join(spec_sens_data, by = "ethnicity") %>% 
  left_join(pred_adj_full_by_year, by = c("YEAR", "ethnicity")) %>% 
  group_by(YEAR) %>% 
  add_tally(wt = prediction) %>% 
  ungroup() %>% 
  rename(total_pred_by_date = n) %>% 
  mutate(pred_adj = (prediction - total_pred_by_date * (1 - Specificity))/(Sensitivity + Specificity -1),
         pred_adj = ifelse(pred_adj < 0, 0, pred_adj),
         german = ifelse(ethnicity == "German", 1, 0)) %>% 
  add_tally(wt = prediction) %>% 
  rename(total_pred = n) %>% 
  add_tally(wt = pred_adj) %>% 
  rename(total_pred_adj = n) %>% 
  add_tally(wt = pred_adj_full) %>% 
  rename(total_pred_adj_full = n) %>% 
  mutate(pred_adj_scaled = round(pred_adj * (total_pred/total_pred_adj)),
         pred_adj_full_scaled = round(pred_adj_full * (total_pred/total_pred_adj_full)),
         n = label + pred_adj_full_scaled, 
         log_n = log(1 + n))

min_by_year_all
}
```

Create and save the different types of datasets (only rehabilitated indviduals, border areas, etc.).

``` r
all_dates <- events %>% 
  filter(!is.na(source_date_arrest)) %>% 
  get_arrests_by_year() %>% 
  dplyr::select(YEAR, ethnicity, label, prediction, pred_adj_scaled, pred_adj_full_scaled, n, log_n)

only_labeled_dates <- events %>% 
  filter(!is.na(source_date_arrest), source_date_arrest == "label") %>% 
  get_arrests_by_year() %>% 
  dplyr::select(YEAR, ethnicity, label, prediction, pred_adj_scaled, pred_adj_full_scaled, n, log_n)

rehabs <- events %>% 
  filter(!is.na(source_date_arrest), rehabilitation == 1) %>% 
  get_arrests_by_year() %>% 
  dplyr::select(YEAR, ethnicity, label, prediction, pred_adj_scaled, pred_adj_full_scaled, n, log_n)

non_border_provinces <- events %>% 
  filter(!is.na(source_date_arrest),  location == "outside 250 km border buffer") %>% 
  get_arrests_by_year() %>% 
  dplyr::select(YEAR, ethnicity, label, prediction, pred_adj_scaled, pred_adj_full_scaled, n, log_n) %>% 
  rename_at(vars(-YEAR, -ethnicity), function(x) paste0(x,"_non_border"))

border_provinces <- events %>% 
  filter(!is.na(source_date_arrest), location == "within 250 km border buffer") %>% 
  get_arrests_by_year() %>% 
  dplyr::select(YEAR, ethnicity, label, prediction, pred_adj_scaled, pred_adj_full_scaled, n, log_n) %>%
  rename_at(vars(-YEAR, -ethnicity), function(x) paste0(x,"_border"))



min_by_year_all <- only_labeled_dates %>% 
  left_join(all_dates, by = c("YEAR", "ethnicity"), suffix = c("", "_imp_date")) %>% 
  left_join(rehabs, by = c("YEAR", "ethnicity"), suffix = c("", "_rehab")) 

write_csv(min_by_year_all, here::here("data/min_by_year_preds.csv")) 


min_by_year_all_border <- non_border_provinces %>% 
  left_join(border_provinces, by = c("YEAR", "ethnicity")) 

write_csv(min_by_year_all_border, here::here("data/min_by_year_borders.csv")) 
```
