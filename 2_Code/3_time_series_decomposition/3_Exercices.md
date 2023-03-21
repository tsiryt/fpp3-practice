# 1
```r
global_economy |>
	filter(Country %in% c('Paraguay', "Madagascar")) |> 
	mutate(gdp_per_capita = GDP / Population) |>
	select(Year, Country, gdp_per_capita) |>
	autoplot(gdp_per_capita, facets = TRUE) + 
	geom_line()
	labs(title= "GDP per capita", y = "$US")
```

# 2
- United States GDP from global_economy.
```r
global_economy |>
	mutate(gdp_per_capita = GDP / Population / CPI) |>
	filter(Country == 'United States') |>
	select(Year, Country, gdp_per_capita) |>
	autoplot(gdp_per_capita) + 
	geom_line(aes(col=Country)) +
	labs(title= "GDP per capita", y = "$US")
```
- Slaughter of Victorian "Bulls, bullocks and steer" in aus_livestock.
```r
aus_livestock |>
	filter(Animal == "Bulls, bullocks and steers") |>
	select(Month , State, Count) |>
	autoplot(Count) + geom_line(aes(col=State)) + labs(title= "GDP per capita", y = "Slaughters") + facet_grid(vars(State), scales="free_y")
```
- Victorian Electricity Demand from vic_elec.
```r
vic_elec |>
	autoplot() + geom_line() + labs(title= "Victorian Electricity & Temp")
vic_elec_demand <- vic_elec |>
	autoplot(Demand) + geom_line() + labs(title= "Victorian Electricity & Temp")
vic_elec_temp <- vic_elec |>
	autoplot(Temperature) + geom_line() + labs(title= "Victorian Electricity & Temp")
vic_elec_demand / vic_elec_temp
```
- Gas production from aus_production.
 ```r
aus_production |>
	autoplot(Gas) + geom_line() + labs(title= "Gas Production Aus")
```
On a besoin d'une Box-Cox transformation
```r
lambda <- aus_production |>
  			features(Gas, feature = guerrero) |>
  			pull(lambda_guerrero)
aus_production |>
  autoplot(box_cox(Gas, lambda)) +
  labs(y = "",
       title = latex2exp::TeX(paste0(
         "Transformed gas production with $\\lambda$ = ",
         round(lambda,2))))
```

# 3 Why is a Box-Cox transformation unhelpful for the canadian_gas data?
```r
canadian_gas |> 
  autoplot() + geom_line() + labs(title= "Canadian Gas") -> plt_canada_gas
plt_canada_gas
```

L'amplitude des variations saisonnieres reste la meme a partir d'un certain moment
```r
lambda <- canadian_gas |>
  			features(Volume, feature = guerrero) |>
  			pull(lambda_guerrero)
canadian_gas |>
  autoplot(box_cox(Volume, lambda)) +
  labs(y = "",
       title = latex2exp::TeX(paste0(
         "Transformed gas production with $\\lambda$ = ",
         round(lambda,2)))) -> plt_bxcx_canada_gas
plt_canada_gas + plt_bxcx_canada_gas
```

# 5 For the following series, find an appropriate Box-Cox transformation in order to stabilise the variance. Tobacco from aus_production, Economy class passengers between Melbourne and Sydney from ansett, and Pedestrian counts at Southern Cross Station from pedestrian.
```r
ansett %>%
  filter(Class == "Economy", Airports == "MEL-SYD") %>%
  tsibble() %>%
  autoplot(Passengers) + geom_line() + 
  labs(title = "Passengers MEL-SYD")

lambda_ansett <- ansett %>%
                   filter(Class == "Economy", Airports == "MEL-SYD") %>%
                   features(Passengers, features = guerrero) %>%
                   pull(lambda_guerrero)

ansett %>%
  filter(Class == "Economy", Airports == "MEL-SYD") %>%
  autoplot(box_cox(Passengers, lambda_ansett)) + 
  geom_line() + 
  labs(title = "Passengers MEL-SYD")
```

# 7 Consider the last five years of the Gas data from aus_production
```r
gas <- tail(aus_production, 5*4) |> select(Gas)
```

## a Plot the time series. Can you identify seasonal fluctuations and/or a trend-cycle?
trend croissante et composante saisonniere

## b Use classical_decomposition with type=multiplicative to calculate the trend-cycle and seasonal indices.
```r
gas %>%
  model(classical_decomposition(Gas, type = "multiplicative")) %>%
  components() %>%
  autoplot()
```
## c Do the results support the graphical interpretation from part a?
Il y'a une forme de periodicite dans le composant random

## d Compute and plot the seasonally adjusted data.
```r
gas %>%
  model(classical_decomposition(Gas, type = "additive")) %>%
  components() %>%
  select(Quarter, season_adjust) %>%
  autoplot()
```
## e Change one observation to be an outlier, and recompute the seasonally adjusted data. What is the effect of the outlier?
```r
p2 <- gas %>%
  mutate(
    Gas = case_when(
      Quarter == make_yearquarter(year = 2006, quarter = 1L) ~ Gas * 300,
      .default = Gas 
    ) 
  ) %>%
  model(classical_decomposition(Gas, type = "additive")) %>%
  components() %>%
  select(Quarter, season_adjust) %>%
  autoplot() + geom_line() + labs(title= "Donnee ajustee", y = "Gas")

p1 <- gas %>%
  mutate(
    Gas = case_when(
      Quarter == make_yearquarter(year = 2006, quarter = 1L) ~ Gas * 300,
      .default = Gas 
    ) 
  ) %>%
  select(Quarter, Gas) %>%
  autoplot() + geom_line() + labs(title= "Donnee brute", y = "Gas")

p1 + p2
```
Le outlier deforme la donnee ajustee, en amortissant le choc.
## f Does it make any difference if the outlier is near the end rather than in the middle of the time series?
```r
gas %>%
  mutate(
    Gas = case_when(
      Quarter == make_yearquarter(year = 2010, quarter = 2L) ~ Gas * 300,
      .default = Gas 
    ) 
  ) %>%
  model(classical_decomposition(Gas, type = "additive")) %>%
  components() %>%
  select(Quarter, season_adjust) %>%
  autoplot()
```
Ca ne change rien si l'outlier est proche de la fin.
S'il s'agissait d'une valeur non touchee par le trend, l'effet n'est pas amorti.

# 10 This exercise uses the canadian_gas data (monthly Canadian gas production in billions of cubic metres, January 1960 - February 2005)

## a Plot the data using autoplot(), gg_subseries() and gg_season() to look at the effect of the changing seasonality over time
```r
canadian_gas %>%
  autoplot() + labs(title = "Canadian gas", y = "Volume")

canadian_gas %>%
  gg_subseries() + labs(title = "Canadian gas", y = "Volume")

canadian_gas %>%
  gg_season() + labs(title = "Canadian gas", y = "Volume")

```
## b Do an STL decomposition of the data. You will need to choose a seasonal window to allow for the changing shape of the seasonal component.
```r
canadian_gas %>%
  model(
      STL(Volume ~ trend(window = 13) + season(window = 7))
  ) %>%
  components() %>%
  autoplot()
```
Je choisis un trend de window 13 par défaut.
Je choisis un season de window 7, en observant le nombre d'années où l'effet saisonnier a un ordre de grandeur "stable", en me basant sur gg_season()

## c How does the seasonal shape change over time? [Hint: Try plotting the seasonal component using gg_season().]
```r
canadian_gas %>%
  model(
      STL(Volume ~ trend(window = 13) + season(window = 7))
  ) %>%
  components() %>%
  gg_season(season_year) + labs(title = "Canadian gas", y = "Effet saisonnier")
```
L'effet saisonnier s'amplifie au cours du temps.
## d Can you produce a plausible seasonally adjusted series?
Je tente avec la transformation de Box-Cox, en effet elle sert a faire que la variation saisonniere soit la meme dans la serie.

En observant les résultats d'après, je constate que le Box-Cox ne parvient pas à modifier simultanément les années de début et les années de fin.
```r
lambda_can_gas <- canadian_gas %>%
                    features(Volume, features = guerrero) %>%
                    pull(lambda_guerrero)

canadian_gas %>%
  mutate(box_cox = box_cox(Volume, lambda_can_gas)) %>%
  autoplot(box_cox) + geom_line() + labs(title = "Box-Cox Canadian gas")
```

J'applique un box-cox et inv_box_cox
```r
canadian_gas %>%
  mutate(Volume = box_cox(Volume, lambda_can_gas)) %>%
  model(
      STL(Volume ~ trend(window = 13) + season(window = 7))
  ) %>%
  components() %>%
  select(Month, season_adjust) %>%
  mutate(Volume = inv_box_cox(season_adjust, lambda_can_gas)) %>%
  autoplot() + labs(title = "Canadian gas", y = "Volume ajuste de l'effet saisonnier post Box-Cox")
```

Je compare les résultats.
```r
pat_can_1 <- canadian_gas %>%
               autoplot() + labs(title = "Canadian gas", y = "Volume")

pat_can_2 <- canadian_gas %>%
  model(
      STL(Volume ~ trend(window = 13) + season(window = 7))
  ) %>%
  components() %>%
  select(Month, season_adjust) %>%
  autoplot() + labs(title = "Canadian gas", y = "Volume ajuste de l'effet saisonnier")

pat_can_3 <- canadian_gas %>%
  mutate(Volume = box_cox(Volume, lambda_can_gas)) %>%
  model(
      STL(Volume ~ trend(window = 13) + season(window = 7))
  ) %>%
  components() %>%
  select(Month, season_adjust) %>%
  mutate(season_adjust = inv_box_cox(season_adjust, lambda_can_gas)) %>%
  autoplot() + labs(title = "Canadian gas", y = "Volume ajuste de l'effet saisonnier post Box-Cox")

pat_can_1 + pat_can_2 + pat_can_3
```
## e Compare the results with those obtained using SEATS and X-11. How are they different?
### Différents modèles
```r
p_x11 <- canadian_gas %>%
  model(x11 = X_13ARIMA_SEATS(Volume ~ x11())) %>%
  components() %>%
  autoplot() + labs(title = "Canadian gas X-11", y = "Volume ajuste de l'effet saisonnier")

p_seats <- canadian_gas %>%
  model(x11 = X_13ARIMA_SEATS(Volume ~ seats())) %>%
  components() %>%
  autoplot() + labs(title = "Canadian gas SEATS", y = "Volume ajuste de l'effet saisonnier")

p_stl <- canadian_gas %>%
  model(
      STL(Volume ~ trend(window = 13) + season(window = 7))
  ) %>%
  components() %>%
  autoplot() + labs(title = "Canadian gas STL", y = "Volume ajuste de l'effet saisonnier")
```
Pour X-11 et SEATS, l'effet saisonnier est plus fort dans les premières années.
Pour STL, l'effet saisonnier est plus fort dans les années du milieu. Cela semble être influencé par les variations fortes de l'effet saisonnier dans cette période.

### Composante saisonniere
```r
p_x11 <- canadian_gas %>%
  model(x11 = X_13ARIMA_SEATS(Volume ~ x11())) %>%
  components() %>%
  select(Month, seasonal) %>%
  gg_season() + labs(title = "Canadian gas X-11", y = "effet saisonnier")

p_seats <- canadian_gas %>%
  model(x11 = X_13ARIMA_SEATS(Volume ~ seats())) %>%
  components() %>%
  select(Month, seasonal) %>%
  gg_season() + labs(title = "Canadian gas SEATS", y = "effet saisonnier")

p_stl <- canadian_gas %>%
  model(
      STL(Volume ~ trend(window = 13) + season(window = 7))
  ) %>%
  components() %>%
  select(Month, season_year) %>%
  gg_season() + labs(title = "Canadian gas STL", y = "effet saisonnier")

p_stl_box_cox <- canadian_gas %>%
  mutate(Volume = box_cox(Volume, lambda_can_gas)) %>%
  model(
      STL(Volume ~ trend(window = 13) + season(window = 7))
  ) %>%
  components() %>%
  select(Month, season_year) %>%
  mutate(season_year = inv_box_cox(season_year, lambda_can_gas)) %>%
  gg_season() + labs(title = "Canadian gas STL Box-Cox", y = "effet saisonnier")

(p_seats + p_x11) / (p_stl + p_stl_box_cox)
```

### Variable hors effet saisonnier
```r
p_x11 <- canadian_gas %>%
  model(x11 = X_13ARIMA_SEATS(Volume ~ x11())) %>%
  components() %>%
  select(Month, season_adjust) %>%
  autoplot() + labs(title = "Canadian gas X-11", y = "Volume ajuste de l'effet saisonnier")

p_seats <- canadian_gas %>%
  model(x11 = X_13ARIMA_SEATS(Volume ~ seats())) %>%
  components() %>%
  select(Month, season_adjust) %>%
  autoplot() + labs(title = "Canadian gas SEATS", y = "Volume ajuste de l'effet saisonnier")

p_stl <- canadian_gas %>%
  model(
      STL(Volume ~ trend(window = 13) + season(window = 7))
  ) %>%
  components() %>%
  select(Month, season_adjust) %>%
  autoplot() + labs(title = "Canadian gas STL", y = "Volume ajuste de l'effet saisonnier")

p_stl_box_cox <- canadian_gas %>%
  mutate(Volume = box_cox(Volume, lambda_can_gas)) %>%
  model(
      STL(Volume ~ trend(window = 13) + season(window = 7))
  ) %>%
  components() %>%
  select(Month, season_adjust) %>%
  mutate(season_adjust = inv_box_cox(season_adjust, lambda_can_gas)) %>%
  autoplot() + labs(title = "Canadian gas STL Box-Cox", y = "Volume ajuste de l'effet saisonnier")

(p_seats + p_x11) / (p_stl + p_stl_box_cox)
```
