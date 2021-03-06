Difference-in-diffrences robustness checks
================
Martin Kosík
April 4, 2019

Import the data

``` r
ethnicity_controls <- read_excel(here::here("data/ethnicity_info.xlsx")) %>%
  mutate(ethnicity_id = as.numeric(1:n()),
         urb_rate_pct = urb_rate * 100)
  
min_by_year <-  read_csv(here::here("data/min_by_year_preds.csv")) %>% 
  mutate(log_n = log(1 + label ),
         log_n_pred_full = log(1 + label + pred_adj_full_scaled),
         log_n_pred = log(1 + label + prediction),
         log_n_imp_date = log(1 + label_imp_date),
         log_n_pred_full_imp_date = log(1 + label_imp_date + pred_adj_full_scaled_imp_date),
         n_pred_full_imp_date =  label_imp_date + pred_adj_full_scaled_imp_date,
         log_n_pred_full_imp_date_rehab = log(1 + label_rehab + pred_adj_full_scaled_rehab),
         log_n_pred_pars_imp_date = log(1 + label_imp_date + pred_adj_scaled_imp_date),
         log_n_pred_imp_date = log(1 + label_imp_date + prediction_imp_date)) %>% 
  left_join(ethnicity_controls, by = "ethnicity") %>% 
  mutate(german = ifelse(ethnicity == "German", 1, 0),
         post_german = german * ifelse(YEAR >= 1933, 1, 0),
         ethnicity = as.factor(ethnicity),
         YEAR_sq = YEAR^2, 
         pre_treatment = ifelse(YEAR >= 1922 & YEAR < 1933, 1, 0),
         hostility = ifelse(YEAR >= 1933 & YEAR <= 1939, 1, 0), 
         pact = ifelse(YEAR > 1939 & YEAR < 1941, 1, 0), 
         war = ifelse(YEAR >= 1941 & YEAR < 1945, 1, 0),
         post_war = ifelse(YEAR >= 1945, 1, 0)) 
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

Geopolitical controls - definition

``` r
min_by_year <- min_by_year %>% 
  mutate(geopol_finnish_war = as.numeric(YEAR >= 1939 & YEAR < 1945 & (ethnicity == "Finnish")),
         geopol_finnish_postwar = as.numeric(YEAR >=  1945 & (ethnicity == "Finnish")),
         geopol_baltic_anex = as.numeric(YEAR == 1940 & (ethnicity %in% c("Estonian", "Latvian", "Lithuanian"))),
         geopol_baltic_nazi = as.numeric(YEAR >= 1941 & YEAR <= 1943 &
                                           (ethnicity %in% c("Estonian", "Latvian", "Lithuanian"))),
         geopol_baltic_postwar = as.numeric(YEAR >= 1944 & 
                                           (ethnicity %in% c("Estonian", "Latvian", "Lithuanian"))),
         geopol_polish_war = as.numeric(YEAR == 1939 & (ethnicity == "Polish")),
         geopol_polish_soviet = as.numeric(YEAR == 1940 & (ethnicity  == "Polish")),
         geopol_polish_nazi = as.numeric(YEAR >= 1941 & YEAR < 1945 & (ethnicity == "Polish")),
         geopol_polish_postwar = as.numeric(YEAR >=  1945 & (ethnicity == "Polish")),
         geopol_japnese_war = as.numeric(YEAR >= 1938 & YEAR <= 1939 &  (ethnicity == "Japanese")),
         geopol_japnese_neutrality = as.numeric(YEAR >= 1938 & YEAR <= 1944 &  (ethnicity == "Japanese")),
         geopol_japnese_ww2 = as.numeric(YEAR == 1945 &  (ethnicity == "Japanese")),
         geopol_japnese_postwar = as.numeric(YEAR > 1945 &  (ethnicity == "Japanese")),
         geopol_hungarian_war = as.numeric(YEAR >= 1941 & YEAR < 1945 & (ethnicity == "Hungarian")),
         geopol_hungarian_postwar = as.numeric(YEAR >=  1945 & (ethnicity == "Hungarian")))
```

Year dummies - definition

``` r
years <- 1922:1960

year_dummies <- map_dfc(years, ~  if (dplyr::last(years) == .){
                                                    as.numeric(min_by_year$YEAR >= .)} else {
                                                    as.numeric(min_by_year$YEAR == .)}) %>% 
  rename_all(funs( c(paste0("year_" ,years))))

min_by_year <- bind_cols(min_by_year, year_dummies)
```

We create all necessary formulas

``` r
geopol_vars <- str_subset(names(min_by_year), "geopol")

fmla_pred_full_imp_date_lin_trends_geopol <- as.formula(paste("log_n_pred_full_imp_date ~ ", 
                         paste0("german:","year_", years, collapse = " + "),  "+ as.factor(YEAR) +
                         ethnicity + ethnicity: YEAR +", paste0(geopol_vars, collapse = " + ")))

fmla_pred_full_imp_date_geopol <- as.formula(paste("log_n_pred_full_imp_date ~ ", 
                         paste0("german:","year_", years, collapse = " + "),  "+ as.factor(YEAR) +
                         ethnicity + ethnicity: YEAR+ ethnicity: YEAR_sq + ",  paste0(geopol_vars, collapse = " + ")))


fmla_pred_full_imp_date_no_trends_geopol <- as.formula(paste("log_n_pred_full_imp_date ~ ", 
                         paste0("german:","year_", years, collapse = " + "),  "+ as.factor(YEAR) +
                         ethnicity+ ",  paste0(geopol_vars, collapse = " + ")))
```

``` r
fit_robust_model <- function(formula = fmla_window_pre_treat_pred_full_imp_date, 
                             data = min_by_year, level = 0.95, vcov = "CR2", estimatr = 1){
  if (estimatr != 1){
  model <- lm(formula,  data = data)
  model %>% 
  conf_int(vcov = vcov,  level = level,
          cluster = data$ethnicity,  test = "Satterthwaite") %>% 
  as_tibble(rownames = "term") %>% 
  filter(str_detect(term, "german:")) %>% 
 # mutate(nice_labs = str_replace(term,"german:","") %>% 
 #          fct_relevel("pre_treatment", "hostility", "pact", "war", "post_war")) %>% 
  rename(conf.low = CI_L, conf.high = CI_U, estimate = beta)
  }
  else {
    model_robust <- lm_robust(formula = formula, data = data, clusters = ethnicity, se_type = vcov,
                                ci = TRUE, alpha = 1 - level)
    model_robust %>% 
      estimatr::tidy(model_robust) %>% 
      filter(str_detect(term, "german:")) %>% 
      dplyr::select(term, estimate, SE = std.error,  conf.low, conf.high) %>% 
      as_tibble()
  }
  }
```

``` r
trends_formulas_full_years_geopol <- c(fmla_pred_full_imp_date_geopol, fmla_pred_full_imp_date_lin_trends_geopol,
                                       fmla_pred_full_imp_date_no_trends_geopol)


map(trends_formulas_full_years_geopol, ~ fit_robust_model(formula = ., vcov = "CR2", estimatr = 0)) %>% 
  bind_rows(.id = "id") %>% 
  mutate(nice_labs = str_replace(term,"german:year_",""),
         nice_labs = as.numeric(nice_labs)
   #      ,nice_labs = ifelse(as.numeric(nice_labs) == max(as.numeric(nice_labs)), str_c(nice_labs, "+"), nice_labs)
         ) %>% 
  mutate(trends = recode_factor(id, "1" = "Quadratic", "2" = "Linear", "3" = "None")) %>% 
  ggplot( aes(x = nice_labs, y = estimate, ymin = conf.low, ymax = conf.high, group = 1, col = trends))+ 
  geom_pointrange(position = position_dodge2(width = 0.45))+  geom_hline(yintercept= 0) + theme_minimal() + 
  geom_vline(xintercept= 1932.5, col = "red", linetype = "dashed", size = 1)+
  theme(axis.line = element_line(size = 1), 
        panel.grid.major.x = element_blank(), panel.grid.minor.x = element_blank(), 
        text = element_text(size=14),
        legend.position = "bottom", 
        axis.text.x = element_text(angle = 60, hjust = 1)) +
  labs(x = "Year", 
        #caption = "error bars show 95% confidence intervals \n 
         #                      SE are based on the cluster-robust estimator by Pustejovsky and Tipton (2018)", 
       y = "Coefficient", col = "Ethnicity-specific time trends")+ 
  scale_x_continuous(breaks=seq(1922,1960,1))
```

``` r
ggsave(here::here("plots/effects/robustness_checks/trends_comp_pred_full_imp_date_cr2.pdf"))
```

    ## Saving 7 x 5 in image

``` r
map(trends_formulas_full_years_geopol, ~ fit_robust_model(formula = ., vcov = "CR2", estimatr = 0)) %>% 
  bind_rows(.id = "id") %>% 
  mutate(nice_labs = str_replace(term,"german:year_",""),
         nice_labs = as.numeric(nice_labs)
   #      ,nice_labs = ifelse(as.numeric(nice_labs) == max(as.numeric(nice_labs)), str_c(nice_labs, "+"), nice_labs)
         ) %>% 
  mutate(trends = recode_factor(id, "1" = "Quadratic", "2" = "Linear", "3" = "None")) %>% 
  ggplot( aes(x = nice_labs, y = estimate, ymin = conf.low, ymax = conf.high, group = 1, col = trends))+ 
  geom_pointrange(position = position_dodge2(width = 0.45))+  geom_hline(yintercept= 0) + theme_minimal() + 
  geom_vline(xintercept= 1932.5, col = "red", linetype = "dashed", size = 1)+
  theme(axis.line = element_line(size = 1), 
        panel.grid.major.x = element_blank(), panel.grid.minor.x = element_blank(), 
        text = element_text(size=14),
        legend.position = "bottom", 
        axis.text.x = element_text(angle = 0, hjust = 0.5)) +
  labs(x = "Year", 
        #caption = "error bars show 95% confidence intervals \n 
         #                      SE are based on the cluster-robust estimator by Pustejovsky and Tipton (2018)", 
       y = "Coefficient", col = "Ethnicity-specific time trends")+ 
  scale_x_continuous(breaks=seq(1922,1960,3))
```

``` r
ggsave(here::here("plots/for_presentation/trends_comp_pred_full_imp_date_cr2.pdf"), scale = 0.8)
```

    ## Saving 5.6 x 4 in image

``` r
geopol_vars <- str_subset(names(min_by_year), "geopol")

fmla_pred_imp_date_no_trends_geopol <- as.formula(paste("log_n_pred_imp_date ~ ", 
                         paste0("german:","year_", years, collapse = " + "),  "+ as.factor(YEAR) +
                         ethnicity+ ",  paste0(geopol_vars, collapse = " + ")))

fmla_pred_full_imp_date_no_trends_geopol <- as.formula(paste("log_n_pred_full_imp_date ~ ", 
                         paste0("german:","year_", years, collapse = " + "),  "+ as.factor(YEAR) +
                         ethnicity+ ",  paste0(geopol_vars, collapse = " + ")))

fmla_pars_pred_imp_date_no_trends_geopol <- as.formula(paste("log_n_pred_pars_imp_date ~ ", 
                         paste0("german:","year_", years, collapse = " + "),  "+ as.factor(YEAR) +
                         ethnicity+ ",  paste0(geopol_vars, collapse = " + ")))
```

``` r
pred_adj_formulas_full_years_geopol <- c(fmla_pred_full_imp_date_no_trends_geopol,
                                       fmla_pars_pred_imp_date_no_trends_geopol,
                                       fmla_pred_imp_date_no_trends_geopol)

## Position dodge
map(pred_adj_formulas_full_years_geopol, ~ fit_robust_model(formula = ., vcov = "CR2", estimatr = 0)) %>% 
  bind_rows(.id = "id") %>% 
  mutate(nice_labs = str_replace(term,"german:year_",""),
         nice_labs = as.numeric(nice_labs)
         #,nice_labs = ifelse(as.numeric(nice_labs) == max(as.numeric(nice_labs)), str_c(nice_labs, "+"), nice_labs)
         ) %>% 
  mutate(trends = recode_factor(id, "1" = "Full matrix", "2" = "Parsimonious", "3" = "None")) %>% 
  ggplot( aes(x = nice_labs, y = estimate, ymin = conf.low, ymax = conf.high, group = 1, col = trends))+ 
  geom_pointrange(position = position_dodge2(width = 0.45))+  geom_hline(yintercept= 0) + theme_minimal() + 
  theme(axis.line = element_line(size = 1), 
        panel.grid.major.x = element_blank(), panel.grid.minor.x = element_blank(), 
        text = element_text(size=14),
        legend.position = "bottom", 
        axis.text.x = element_text(angle = 60, hjust = 1)) +
  scale_x_continuous(breaks=seq(1922,1960,1))+ 
  geom_vline(xintercept= 1932.5, col = "red", linetype = "dashed", size = 1)+
  labs(x = "Year", 
        #caption = "error bars show 95% confidence intervals \n 
         #                      SE are based on the cluster-robust estimator by Pustejovsky and Tipton (2018)", 
       y = "Coefficient", col = "Ethnicity imputaiton adjustment")
```

``` r
ggsave(here::here("plots/effects/robustness_checks/pred_adj_comp_pred_full_imp_date_cr2.pdf"))
```

    ## Saving 7 x 5 in image

``` r
map(pred_adj_formulas_full_years_geopol, ~ fit_robust_model(formula = ., vcov = "CR2", estimatr = 0)) %>% 
  bind_rows(.id = "id") %>% 
  mutate(nice_labs = str_replace(term,"german:year_",""),
         nice_labs = as.numeric(nice_labs)
         #,nice_labs = ifelse(as.numeric(nice_labs) == max(as.numeric(nice_labs)), str_c(nice_labs, "+"), nice_labs)
         ) %>% 
  mutate(trends = recode_factor(id, "1" = "Full matrix", "2" = "Parsimonious", "3" = "None")) %>% 
  ggplot( aes(x = nice_labs, y = estimate, ymin = conf.low, ymax = conf.high, group = 1, col = trends))+ 
  geom_pointrange(position = position_dodge2(width = 0.45))+  geom_hline(yintercept= 0) + theme_minimal() + 
  theme(axis.line = element_line(size = 1), 
        panel.grid.major.x = element_blank(), panel.grid.minor.x = element_blank(), 
        text = element_text(size=14),
        legend.position = "bottom", 
        axis.text.x = element_text(angle = 0, hjust = 0.5)) +
  scale_x_continuous(breaks=seq(1922,1960,3))+ 
  geom_vline(xintercept= 1932.5, col = "red", linetype = "dashed", size = 1)+
  labs(x = "Year", 
        #caption = "error bars show 95% confidence intervals \n 
         #                      SE are based on the cluster-robust estimator by Pustejovsky and Tipton (2018)", 
       y = "Coefficient", col = "Ethnicity imputaiton adjustment")
```

``` r
ggsave(here::here("plots/for_presentation/pred_adj_comp_pred_full_imp_date_cr2.pdf"), scale = 0.8)
```

    ## Saving 5.6 x 4 in image
