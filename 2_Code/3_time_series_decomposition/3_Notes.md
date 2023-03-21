# Time series decomposition
When we decompose a time series into components, we usually combine the trend and cycle into a single **trend-cycle** component (often just called **the trend** for simplicity). Thus we can think of a time series as comprising three components: a trend-cycle component, a seasonal component, and a remainder component (containing anything else in the time series). For some time series (e.g., those that are observed at least daily), there can be more than one seasonal component, corresponding to the different seasonal periods.

## 1.Transformation and adjustments
The purpose of these adjustments and transformations is to simplify the patterns in the historical data by removing known sources of variation, or by making the pattern more consistent across the whole data set.

### Calendar adjustments
Les mois n'ont pas la même durée !

### Population adjustments
Il est plus facile d'interpreter les variations par personne, plutot que le total
```r
global_economy |>
  filter(Country == "Australia") |>
  autoplot(GDP/Population) +
  labs(title= "GDP per capita", y = "$US")
```

### Inflation adjustmentk
On compare les valeurs monetaires a une annee echelon.
For consumer goods, a common price index is the **Consumer Price Index** (or CPI).
If $z_{t}$ denotes the price index and  $y_{t}$ denotes the original house price in year $t$, then $x_t = y_t/z_t \times z_{2000}$
  gives the adjusted house price at year 2000 dollar values.

### Mathematical transformations
If the data shows variation that increases or decreases with the level of the series, then a transformation can be useful.
Une famille de transformateurs utiles dans ce cas-là  **Box-Cox transformations** :
$$
w_t = \begin{cases}
log(y_t),  & \text{if $\lambda$ = 0} \\\
(sign(y_t) \lvert y_t \rvert^{\lambda} - 1), & \text{otherwise}
\end{cases}
$$
A good value of $\lambda$ is one which makes the size of the seasonal variation about the same across the whole series, as that makes the forecasting model simpler.

La feature `guerrero` choisit le lambda qui convient.
```r
lambda <- aus_production |>
  features(Gas, features = guerrero) |>
  pull(lambda_guerrero)
aus_production |>
  autoplot(box_cox(Gas, lambda)) +
  labs(y = "",
       title = latex2exp::TeX(paste0(
         "Transformed gas production with $\\lambda$ = ",
         round(lambda,2))))
```


## 2 Time series components
Modèle additif : $y_t = S_t + T_t + R_t$

Modèle multiplicatif : $y_t = S_t \times T_t \times R_t$

Le modèle additif est plus pertinent si l'amplitude des fluctuations saisonnières, ou la variation autour du trend-cycle ne varient pas avec le niveau de la série. S'il y'a une forme de proportionnalité, le modèle multiplicatif est plus pertinent.
```r
us_retail_employment <- us_employment |>
  filter(year(Month) >= 1990, Title == "Retail Trade") |>
  select(-Series_ID)
autoplot(us_retail_employment, Employed) +
  labs(y = "Persons (thousands)",
       title = "Total employment in US retail")
```

### Decomposition STL
Season + Trend + L
```r
dcmp <- us_retail_employment |>
  model(stl = STL(Employed))
components(dcmp)
```

On affiche tous les composants.
La barre grise permet de voir l'échelle de chaque composant.
```r
components(dcmp) |> autoplot()
```

Généralement, on est intéressé par la variable non-saisonnière. 
Par contre, pour détecter des "tournants", il est préférable de regarder le trend.
**Exemple** :
For example, monthly unemployment data are usually seasonally adjusted in order to highlight variation due to the underlying state of the economy rather than the seasonal variation. An increase in unemployment due to school leavers seeking work is seasonal variation, while an increase in unemployment due to an economic recession is non-seasonal.


## 3 Moving average
$$
\hat T_t = \frac{1}{m} \sum_{j=-k}^{k}  y_{t+j}
$$
où $m = 2k+1$
On estime le trend-cycle a l'aide d'une moyenne glissante.
La moyenne glissante permet d'éliminer une partie de l'aleatoire dans les donnees, car les observations proches les unes des autres ont probablement des valeurs proches.
```r
aus_exports <- global_economy |>
  filter(Country == "Australia") |>
  mutate(
    `5-MA` = slider::slide_dbl(Exports, mean,
                .before = 2, .after = 2, .complete = TRUE)
  )
aus_exports |>
  autoplot(Exports) +
  geom_line(aes(y = `5-MA`), colour = "#D55E00") +
  labs(y = "% of GDP",
       title = "Total Australian exports") +
  guides(colour = guide_legend(title = "series"))
```

### Moyenne glissante de moyenne glissante
Cette operation permet de rendre des MA d'ordre pair symétriques.
Par exemple, la $2 \times 4-MA$ permet d'obtenir une moyenne symétrique avec les coefficients centrés qui sont plus importants.
$$
\hat T_t = \frac{1}{8} y_{t-2} + \frac{1}{4} y_{t-1} + \frac{1}{4}y_{t} + \frac{1}{4}y_{t+1} + \frac{1}{8}y_{t+2}
$$

Si la donnée est répartie par quartiers $\\{Q_1 \dots Q_4\\}$ alors le 1er et dernier terme, pour un quartier donné, de la $2 \times 4-MA$ sont le meme quartier, a une année d'écart. On moyenne donc l'effet saisonnier.
$$
\hat T_{Q_3|A} = \frac{1}{8} y_{Q_1|A} + \frac{1}{4} y_{Q_2|A} + \frac{1}{4}y_{Q_3|A} + \frac{1}{4}y_{Q_4|A} + \frac{1}{8}y_{Q_1|A+1}
$$

La $2 \times m-MA$ est équivalente à une m-MA ou toutes les observations ont le meme poids, sauf la premiere et la derniere.
Si $m$, la periode saisonnière est paire on peut estimer le trend avec $2 \times m-MA$ 
Si $m$, la periode saisonnière est impaire on peut estimer le trend avec $m-MA$ 

**Exemples**
$2 \times 12-MA$ can be used to estimate the trend-cycle of monthly data with annual seasonality
$7-MA$ can be used to estimate the trend-cycle of daily data with a weekly seasonality

## 4 Decomposition classique
**Hypothèse :** we assume that the seasonal component is constant from year to year

### Additive
#### 1 - Calcul de $\hat T_t$
$2 \times m-MA$ ou $m-MA$ selon la parité de $m$.

#### 2 - Calcul de $y_t - \hat T_t$
#### 3 - Calcul de $\hat S_t$
To estimate the seasonal component for each season, average the detrended values for that season. These seasonal component values are then adjusted to ensure that they add to zero. The seasonal component is obtained by stringing together these monthly values, and then replicating the sequence for each year of data.
#### 3 - Calcul de $\hat R_t = y_t - \hat T_t - \hat S_t$
```r
us_retail_employment |>
  model(
    classical_decomposition(Employed, type = "additive")
  ) |>
  components() |>
  autoplot() +
  labs(title = "Classical additive decomposition of total
                  US retail employment")
```
### Multiplicative
Idem qu'en additif, avec des divisions à la place des soustractions.

### Commentaires
- pas de valeurs de trend pour les extrémités
- la trend peut oversmooth des variations rapides
- hypothèse : composante saisonnière **constante tous les ans** 
- peu robuste s'il y'a des outliers

## 5 Méthodes officielles
Les agences officielles utilisent des variantes de X-11 ou SEATS.
Ces méthodes fonctionnent bien avec des données par quartiers/mensuelles.

### Méthode X-11
- basée sur la décomposition classique
- on a du trend pour toutes les observations
- effet saisonnier peut varier légèrement au fil du temps
```r
x11_dcmp <- us_retail_employment |>
  model(x11 = X_13ARIMA_SEATS(Employed ~ x11())) |>
  components()
autoplot(x11_dcmp) +
  labs(title =
    "Decomposition of total US retail employment using X-11.")
```
Par défaut, le X-11 utilise un **modèle multiplicatif**

L'effet de la crise de 2007-2008 est bien capturé par les composants saisonniers et trend. Pour le modèle classique, la crise se répercutait dans la variable remainder : on y voyait une chute. 
On regarde la variation de l'effet saisonnier au fil du temps. Il est proche de 1, car on est en **modèle multiplicatif**
```r
x11_dcmp |>
  gg_subseries(seasonal)
```

### SEATS
SEATS stands for "Seasonal Extraction in ARIMA Time Series"
```r
seats_dcmp <- us_retail_employment |>
  model(seats = X_13ARIMA_SEATS(Employed ~ seats())) |>
  components()
autoplot(seats_dcmp) +
  labs(title =
    "Decomposition of total US retail employment using SEATS")
```

## 6 Decomposition STL
STL is an acronym for "Seasonal and Trend decomposition using Loess", while loess is a method for estimating nonlinear relationships. 

### Paramètres
The two main parameters to be chosen when using STL are the trend-cycle window trend(window = ?) and the seasonal window season(window = ?). These control how rapidly the trend-cycle and seasonal components can change. **Smaller values** allow for more **rapid changes**.
Both trend and seasonal windows should be odd numbers; **trend window** is the number of consecutive observations to be used when estimating the trend-cycle; **season window** is the number of consecutive years to be used in estimating each value in the seasonal component. 

**Avantages de STL**

- tout type de saisonnalité, pas juste quartier et mensuel (contrairement à X-11 et SEATS)
- composante saisonnière peut varier au fil du temps, à un taux controlable
- le smooth du trend est ajustable
- *peut* etre robuste aux outliers si spécifié, de sorte que des observations étranges affectent le remainder au lieu du trend ou du season

**Desavantages de STL**

- In particular, it does not handle trading day or calendar variation automatically (contrairement a X-11)
- conçu pour des modèles additifs

```r
us_retail_employment |>
  model(
    STL(Employed ~ trend(window = 7) +
                   season(window = "periodic"),
    robust = TRUE)) |>
  components() |>
  autoplot()
```