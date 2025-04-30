Regression Case Study: PSAAP II
================
(Your name here)
2020-

- [Grading Rubric](#grading-rubric)
  - [Individual](#individual)
  - [Submission](#submission)
- [Orientation: Exploring Simulation
  Results](#orientation-exploring-simulation-results)
  - [**q1** Perform your “initial checks” to get a sense of the
    data.](#q1-perform-your-initial-checks-to-get-a-sense-of-the-data)
  - [**q2** Visualize `T_norm` against `x`. Note that there are multiple
    simulations at different values of the Input variables: Each
    simulation result is identified by a different value of
    `idx`.](#q2-visualize-t_norm-against-x-note-that-there-are-multiple-simulations-at-different-values-of-the-input-variables-each-simulation-result-is-identified-by-a-different-value-of-idx)
  - [Modeling](#modeling)
    - [**q3** The following code chunk fits a few different models.
      Compute a measure of model accuracy for each model on
      `df_validate`, and compare their
      performance.](#q3-the-following-code-chunk-fits-a-few-different-models-compute-a-measure-of-model-accuracy-for-each-model-on-df_validate-and-compare-their-performance)
    - [**q4** Interpret this model](#q4-interpret-this-model)
  - [Contrasting CI and PI](#contrasting-ci-and-pi)
    - [**q5** The following code will construct a predicted-vs-actual
      plot with your model from *q4* and add prediction intervals. Study
      the results and answer the questions below under
      *observations*.](#q5-the-following-code-will-construct-a-predicted-vs-actual-plot-with-your-model-from-q4-and-add-prediction-intervals-study-the-results-and-answer-the-questions-below-under-observations)
- [Case Study: Predicting Performance
  Ranges](#case-study-predicting-performance-ranges)
  - [**q6** You are consulting with a team that is designing a prototype
    heat transfer device. They are asking you to help determine a
    *dependable range of values* for `T_norm` they can design around for
    this *single prototype*. The realized value of `T_norm` must not be
    too high as it may damage the downstream equipment, but it must also
    be high enough to extract an acceptable amount of
    heat.](#q6-you-are-consulting-with-a-team-that-is-designing-a-prototype-heat-transfer-device-they-are-asking-you-to-help-determine-a-dependable-range-of-values-for-t_norm-they-can-design-around-for-this-single-prototype-the-realized-value-of-t_norm-must-not-be-too-high-as-it-may-damage-the-downstream-equipment-but-it-must-also-be-high-enough-to-extract-an-acceptable-amount-of-heat)
- [References](#references)

*Purpose*: Confidence and prediction intervals are useful for studying
“pure sampling” of some distribution. However, we can combine CI and PI
with regression analysis to equip our modeling efforts with powerful
notions of uncertainty. In this challenge, you will use fluid simulation
data in a regression analysis with uncertainty quantification (CI and
PI) to support engineering design.

<!-- include-rubric -->

# Grading Rubric

<!-- -------------------------------------------------- -->

Unlike exercises, **challenges will be graded**. The following rubrics
define how you will be graded, both on an individual and team basis.

## Individual

<!-- ------------------------- -->

| Category | Needs Improvement | Satisfactory |
|----|----|----|
| Effort | Some task **q**’s left unattempted | All task **q**’s attempted |
| Observed | Did not document observations, or observations incorrect | Documented correct observations based on analysis |
| Supported | Some observations not clearly supported by analysis | All observations clearly supported by analysis (table, graph, etc.) |
| Assessed | Observations include claims not supported by the data, or reflect a level of certainty not warranted by the data | Observations are appropriately qualified by the quality & relevance of the data and (in)conclusiveness of the support |
| Specified | Uses the phrase “more data are necessary” without clarification | Any statement that “more data are necessary” specifies which *specific* data are needed to answer what *specific* question |
| Code Styled | Violations of the [style guide](https://style.tidyverse.org/) hinder readability | Code sufficiently close to the [style guide](https://style.tidyverse.org/) |

## Submission

<!-- ------------------------- -->

Make sure to commit both the challenge report (`report.md` file) and
supporting files (`report_files/` folder) when you are done! Then submit
a link to Canvas. **Your Challenge submission is not complete without
all files uploaded to GitHub.**

``` r
library(tidyverse)
```

    ## ── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
    ## ✔ dplyr     1.1.4     ✔ readr     2.1.5
    ## ✔ forcats   1.0.0     ✔ stringr   1.5.1
    ## ✔ ggplot2   3.5.1     ✔ tibble    3.2.1
    ## ✔ lubridate 1.9.4     ✔ tidyr     1.3.1
    ## ✔ purrr     1.0.2     
    ## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ## ✖ dplyr::filter() masks stats::filter()
    ## ✖ dplyr::lag()    masks stats::lag()
    ## ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors

``` r
library(modelr)
library(broom)
```

    ## 
    ## Attaching package: 'broom'
    ## 
    ## The following object is masked from 'package:modelr':
    ## 
    ##     bootstrap

``` r
## Helper function to compute uncertainty bounds
add_uncertainties <- function(data, model, prefix = "pred", ...) {
  df_fit <-
    stats::predict(model, data, ...) %>%
    as_tibble() %>%
    rename_with(~ str_c(prefix, "_", .))

  bind_cols(data, df_fit)
}
```

# Orientation: Exploring Simulation Results

*Background*: The data you will study in this exercise come from a
computational fluid dynamics (CFD) [simulation
campaign](https://www.sciencedirect.com/science/article/abs/pii/S0301932219308651?via%3Dihub)
that studied the interaction of turbulent flow and radiative heat
transfer to fluid-suspended particles\[1\]. These simulations were
carried out to help study a novel design of [solar
receiver](https://en.wikipedia.org/wiki/Concentrated_solar_power),
though they are more aimed at fundamental physics than detailed device
design. The following code chunk downloads and unpacks the data to your
local `./data/` folder.

``` r
## NOTE: No need to edit this chunk
## Download PSAAP II data and unzip
url_zip <- "https://ndownloader.figshare.com/files/24111269"
filename_zip <- "./data/psaap.zip"
filename_psaap <- "./data/psaap.csv"

curl::curl_download(url_zip, destfile = filename_zip)
unzip(filename_zip, exdir = "./data")
df_psaap <- read_csv(filename_psaap)
```

    ## Rows: 140 Columns: 22
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## dbl (22): x, idx, L, W, U_0, N_p, k_f, T_f, rho_f, mu_f, lam_f, C_fp, rho_p,...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

![PSAAP II irradiated core flow](./images/psaap-setup.png) Figure 1. An
example simulation, frozen at a specific point in time. An initial
simulation is run (HIT SECTION) to generate a turbulent flow with
particles, and that swirling flow is released into a rectangular domain
(RADIATED SECTION) with bulk downstream flow (left to right).
Concentrated solar radiation transmits through the optically transparent
fluid, but deposits heat into the particles. The particles then convect
heat into the fluid, which heats up the flow. The false-color image
shows the fluid temperature: Notice that there are “hot spots” where hot
particles have deposited heat into the fluid. The dataset `df_psaap`
gives measurements of `T_norm = (T - T0) / T0` averaged across planes at
various locations along the RADIATED SECTION.

### **q1** Perform your “initial checks” to get a sense of the data.

``` r
## TODO: Perform your initial checks

head(df_psaap)
```

    ## # A tibble: 6 × 22
    ##       x   idx     L      W   U_0     N_p    k_f   T_f rho_f    mu_f  lam_f  C_fp
    ##   <dbl> <dbl> <dbl>  <dbl> <dbl>   <dbl>  <dbl> <dbl> <dbl>   <dbl>  <dbl> <dbl>
    ## 1  0.25     1 0.190 0.0342  1.86  1.60e6 0.0832  300. 1.16  1.52e-5 0.0316 1062.
    ## 2  0.25     2 0.151 0.0464  2.23  2.22e6 0.111   243. 1.13  1.84e-5 0.0259 1114.
    ## 3  0.25     3 0.169 0.0398  2.04  1.71e6 0.0867  290. 1.10  2.18e-5 0.0349  952.
    ## 4  0.25     4 0.135 0.0325  2.45  2.08e6 0.121   358. 1.23  2.23e-5 0.0370  998.
    ## 5  0.25     5 0.201 0.0441  1.70  1.95e6 0.0904  252. 1.44  2.28e-5 0.0356  937.
    ## 6  0.25     6 0.160 0.0379  1.96  1.82e6 0.0798  280. 0.964 2.13e-5 0.0249 1224.
    ## # ℹ 10 more variables: rho_p <dbl>, d_p <dbl>, C_pv <dbl>, h <dbl>, I_0 <dbl>,
    ## #   eps_p <dbl>, avg_q <dbl>, avg_T <dbl>, rms_T <dbl>, T_norm <dbl>

``` r
distinct(df_psaap, W)
```

    ## # A tibble: 35 × 1
    ##         W
    ##     <dbl>
    ##  1 0.0342
    ##  2 0.0464
    ##  3 0.0398
    ##  4 0.0325
    ##  5 0.0441
    ##  6 0.0379
    ##  7 0.0360
    ##  8 0.0488
    ##  9 0.0419
    ## 10 0.0336
    ## # ℹ 25 more rows

**Observations**:

- There are a lot of different variables here.

  - 22 total

- All are numbers (dbl)

- There are only 4 documented positions for x –\> 0.25, 0.5, 0.75, 1

The important variables in this dataset are:

| Variable | Category | Meaning                           |
|----------|----------|-----------------------------------|
| `x`      | Spatial  | Channel location                  |
| `idx`    | Metadata | Simulation run                    |
| `L`      | Input    | Channel length                    |
| `W`      | Input    | Channel width                     |
| `U_0`    | Input    | Bulk velocity                     |
| `N_p`    | Input    | Number of particles               |
| `k_f`    | Input    | Turbulence level                  |
| `T_f`    | Input    | Fluid inlet temp                  |
| `rho_f`  | Input    | Fluid density                     |
| `mu_f`   | Input    | Fluid viscosity                   |
| `lam_f`  | Input    | Fluid conductivity                |
| `C_fp`   | Input    | Fluid isobaric heat capacity      |
| `rho_p`  | Input    | Particle density                  |
| `d_p`    | Input    | Particle diameter                 |
| `C_pv`   | Input    | Particle isochoric heat capacity  |
| `h`      | Input    | Convection coefficient            |
| `I_0`    | Input    | Radiation intensity               |
| `eps_p`  | Input    | Radiation absorption coefficient  |
| `avg_q`  | Output   | Plane-averaged heat flux          |
| `avg_T`  | Output   | Plane-averaged fluid temperature  |
| `rms_T`  | Output   | Plane-rms fluid temperature       |
| `T_norm` | Output   | Normalized fluid temperature rise |

The primary output of interest is `T_norm = (avg_T - T_f) / T_f`, the
normalized (dimensionless) temperature rise of the fluid, due to heat
transfer. These measurements are taken at locations `x` along a column
of fluid, for different experimental settings (e.g. different dimensions
`W, L`, different flow speeds `U_0`, etc.).

### **q2** Visualize `T_norm` against `x`. Note that there are multiple simulations at different values of the Input variables: Each simulation result is identified by a different value of `idx`.

``` r
## TODO: Visualize the data in df_psaap with T_norm against x;
##       design your visual to handle the multiple simulations,
##       each identified by different values of idx
df_psaap %>%
  ggplot() +
  geom_line(aes(x, T_norm, color = factor(idx)))
```

![](c11-psaap-assignment_files/figure-gfm/q2-task-1.png)<!-- -->

## Modeling

The following chunk will split the data into training and validation
sets.

``` r
## NOTE: No need to edit this chunk
# Addl' Note: These data are already randomized by idx; no need
# to additionally shuffle the data!
df_train <- df_psaap %>% filter(idx %in% 1:20)
df_validate <- df_psaap %>% filter(idx %in% 21:36)
```

One of the key decisions we must make in modeling is choosing predictors
(features) from our observations to include in the model. Ideally we
should have some intuition for why these predictors are reasonable to
include in the model; for instance, we saw above that location along the
flow `x` tends to affect the temperature rise `T_norm`. This is because
fluid downstream has been exposed to solar radiation for longer, and
thus is likely to be at a higher temperature.

Reasoning about our variables—at least at a *high level*—can help us to
avoid including *fallacious* predictors in our models. You’ll explore
this idea in the next task.

### **q3** The following code chunk fits a few different models. Compute a measure of model accuracy for each model on `df_validate`, and compare their performance.

``` r
## NOTE: No need to edit these models
fit_baseline <-
  df_train %>%
  lm(formula = T_norm ~ x)

fit_cheat <-
  df_train %>%
  lm(formula = T_norm ~ avg_T)

fit_nonphysical <-
  df_train %>%
  lm(formula = T_norm ~ idx)

## TODO: Compute a measure of accuracy for each fit above;
##       compare their relative performance
df_validate$pred_baseline <- predict(fit_baseline, newdata = df_validate)
df_validate$pred_cheat <- predict(fit_cheat, newdata = df_validate)
df_validate$pred_nonphysical <- predict(fit_nonphysical, newdata = df_validate)

rmse <- function(actual, predicted) {
  sqrt(mean((actual - predicted)^2))
}

rmse_baseline <- rmse(df_validate$T_norm, df_validate$pred_baseline)
rmse_cheat <- rmse(df_validate$T_norm, df_validate$pred_cheat)
rmse_nonphysical <- rmse(df_validate$T_norm, df_validate$pred_nonphysical)

cat(rmse_baseline, rmse_cheat, rmse_nonphysical)
```

    ## 0.2844778 0.2317709 0.3988128

**Observations**:

- Which model is *most accurate*? Which is *least accurate*?
  - The cheat one is most accurate and the non physical model is the
    least accurate
  - Since we are using rmse, the closer to 0 the value the more accurate
    the model.
- What *Category* of variable is `avg_T`? Why is it such an effective
  predictor?
  - it is an output of average temperature.
  - Its effective because were using the answer to predict the answer
- Would we have access to `avg_T` if we were trying to predict a *new*
  value of `T_norm`? Is `avg_T` a valid predictor?
  - No its not a valid predictor
- What *Category* of variable is `idx`? Does it have any physical
  meaning?
  - kind of simulation its running
  - its a metadata number so it doesnt actually hold a real value

### **q4** Interpret this model

Interpret the following model by answering the questions below.

*Note*. The `-` syntax in R formulas allows us to exclude columns from
fitting. So `T_norm ~ . - x` would fit on all columns *except* `x`.

``` r
## TODO: Inspect the regression coefficients for the following model
fit_q4 <-
  df_train %>%
  lm(formula = T_norm ~ . - idx - avg_q - avg_T - rms_T)
# lm(formula = T_norm ~ L + W + U_0 + N_p + k_f + T_f)
# lm(formula = T_norm ~ L - W - U_0 - N_p - k_f - T_f)

fit_q4 %>%
  tidy() %>%
  arrange(p.value)
```

    ## # A tibble: 18 × 5
    ##    term        estimate std.error statistic  p.value
    ##    <chr>          <dbl>     <dbl>     <dbl>    <dbl>
    ##  1 x            1.02e+0   5.61e-2  18.2     1.66e-26
    ##  2 W           -3.71e+1   6.39e+0  -5.81    2.33e- 7
    ##  3 L            3.96e+0   9.36e-1   4.23    7.81e- 5
    ##  4 I_0          1.60e-7   4.29e-8   3.73    4.13e- 4
    ##  5 U_0         -3.26e-1   1.14e-1  -2.87    5.63e- 3
    ##  6 d_p          1.24e+5   4.49e+4   2.75    7.76e- 3
    ##  7 C_fp        -6.59e-4   3.20e-4  -2.06    4.39e- 2
    ##  8 N_p          2.72e-7   1.38e-7   1.97    5.33e- 2
    ##  9 C_pv        -7.24e-4   3.72e-4  -1.94    5.64e- 2
    ## 10 rho_f       -5.62e-1   3.21e-1  -1.75    8.51e- 2
    ## 11 k_f          2.55e+0   2.13e+0   1.20    2.36e- 1
    ## 12 eps_p        1.11e+0   1.10e+0   1.01    3.18e- 1
    ## 13 mu_f        -8.25e+3   1.47e+4  -0.562   5.76e- 1
    ## 14 lam_f       -4.68e+0   1.11e+1  -0.420   6.76e- 1
    ## 15 T_f         -3.79e-4   1.17e-3  -0.323   7.48e- 1
    ## 16 rho_p        5.64e-6   1.84e-5   0.307   7.60e- 1
    ## 17 h            1.41e-6   4.87e-5   0.0289  9.77e- 1
    ## 18 (Intercept) -3.03e-3   1.66e+0  -0.00183 9.99e- 1

``` r
# df_psaap%>%
#   summarize(sd_x = sd(x, na.rm = TRUE), sd_Tf = sd(T_f, na.rm = TRUE))


df_validate$pred_q4 <- predict(fit_q4, newdata = df_validate)

rmse_q4 <- rmse(df_validate$T_norm, df_validate$pred_q4)

print(rmse_q4)
```

    ## [1] 0.1734866

**Observations**:

- Which columns are excluded in the model formula above? What categories
  do these belong to? Why are these important quantities to leave out of
  the model?
  - idx, avg_q, avg_T, rms_T
  - plane averaged heat flux (q)
  - average temp
  - plane rms fluid temperature
  - these are all outputs (or metadata in idx’s case) so its not
    relevant for the T_norm calculation
- Which inputs are *statistically significant*, according to the model?
  - if the p_value is less than 0.05

  - x

  - W

  - L

  - I_0

  - U_0

  - d_p

  - C_fp
- What is the regression coefficient for `x`? What about the regression
  coefficient for `T_f`?
  - the regression coeff for x is 1.02
  - The regression coeff for T_f is -3.79e-4
- What is the standard deviation of `x` in `df_psaap`? What about the
  standard deviation of `T_f`?
  - The standard deviation for x is 0.2805
  - The standard deviation for T_f is 38.94204
- How do these standard deviations relate to the regression coefficients
  for `x` and `T_f`?
  - the p_value represents how closely the input is associated with the
    output. The x is highly correlated with the output so its’ p_value
    is extremely small and its standard deviation is small for this
    reason. Since T_f is not well related with the output the standard
    deviation is quite large
- Note that literally *all* of the inputs above have *some* effect on
  the output `T_norm`; so they are all “significant” in that sense. What
  does this tell us about the limitations of statistical significance
  for interpreting regression coefficients?
  - The limitations of only considering statistical significant inputs
    is that the significance cutoff is fairly arbitrary. The p_value
    only demonstrates correlation and not causation. We know that all
    the inputs have *some* effect and perhaps they effect a different
    input.

## Contrasting CI and PI

Let’s revisit the ideas of confidence intervals (CI) and prediction
intervals (PI). Let’s fit a very simple model to these data, one which
only considers the channel location and ignores all other inputs. We’ll
also use the helper function `add_uncertainties()` (defined in the
`setup` chunk above) to add approximate CI and PI to the linear model.

``` r
## NOTE: No need to edit this chunk
fit_simple <-
  df_train %>%
  lm(data = ., formula = T_norm ~ x)

df_intervals <-
  df_train %>%
  add_uncertainties(fit_simple, interval = "confidence", prefix = "ci") %>%
  add_uncertainties(fit_simple, interval = "prediction", prefix = "pi")
```

The following figure visualizes the regression CI and PI against the
objects they are attempting to capture:

``` r
## NOTE: No need to edit this chunk
df_intervals %>%
  select(T_norm, x, matches("ci|pi")) %>%
  pivot_longer(
    names_to = c("method", ".value"),
    names_sep = "_",
    cols = matches("ci|pi")
  ) %>%
  ggplot(aes(x, fit)) +
  geom_errorbar(
    aes(ymin = lwr, ymax = upr, color = method),
    width = 0.05,
    size = 1
  ) +
  geom_smooth(
    data = df_psaap %>% mutate(method = "ci"),
    mapping = aes(x, T_norm),
    se = FALSE,
    linetype = 2,
    color = "black"
  ) +
  geom_point(
    data = df_validate %>% mutate(method = "pi"),
    mapping = aes(x, T_norm),
    size = 0.5
  ) +
  facet_grid(~method) +
  theme_minimal() +
  labs(
    x = "Channel Location (-)",
    y = "Normalized Temperature Rise (-)"
  )
```

    ## Warning: Using `size` aesthetic for lines was deprecated in ggplot2 3.4.0.
    ## ℹ Please use `linewidth` instead.
    ## This warning is displayed once every 8 hours.
    ## Call `lifecycle::last_lifecycle_warnings()` to see where this warning was
    ## generated.

    ## `geom_smooth()` using method = 'loess' and formula = 'y ~ x'

    ## Warning in simpleLoess(y, x, w, span, degree = degree, parametric = parametric,
    ## : pseudoinverse used at 0.24625

    ## Warning in simpleLoess(y, x, w, span, degree = degree, parametric = parametric,
    ## : neighborhood radius 0.50375

    ## Warning in simpleLoess(y, x, w, span, degree = degree, parametric = parametric,
    ## : reciprocal condition number 7.4302e-16

    ## Warning in simpleLoess(y, x, w, span, degree = degree, parametric = parametric,
    ## : There are other near singularities as well. 0.25376

![](c11-psaap-assignment_files/figure-gfm/data-simple-model-vis-1.png)<!-- -->

Under the `ci` facet we have the regression confidence intervals and the
mean trend (computed with all the data `df_psaap`). Under the `pi` facet
we have the regression prediction intervals and the `df_validation`
observations.

**Punchline**:

- Confidence intervals are meant to capture the *mean trend*
- Prediction intervals are meant to capture *new observations*

Both CI and PI are a quantification of the uncertainty in our model, but
the two intervals designed to answer different questions.

Since CI and PI are a quantification of uncertainty, they should tend to
*narrow* as our model becomes more confident in its predictions.
Building a more accurate model will often lead to a reduction in
uncertainty. We’ll see this phenomenon in action with the following
task:

### **q5** The following code will construct a predicted-vs-actual plot with your model from *q4* and add prediction intervals. Study the results and answer the questions below under *observations*.

``` r
## TODO: Run this code and interpret the results
## NOTE: No need to edit this chunk
## NOTE: This chunk will use your model from q4; it will predict on the
##       validation data, add prediction intervals for every prediction,
##       and visualize the results on a predicted-vs-actual plot. It will
##       also compare against the simple `fit_simple` defined above.
bind_rows(
  df_psaap %>%
    add_uncertainties(fit_simple, interval = "prediction", prefix = "pi") %>%
    select(T_norm, pi_lwr, pi_fit, pi_upr) %>%
    mutate(model = "x only"),
  df_psaap %>%
    add_uncertainties(fit_q4, interval = "prediction", prefix = "pi") %>%
    select(T_norm, pi_lwr, pi_fit, pi_upr) %>%
    mutate(model = "q4"),
) %>%
  ggplot(aes(T_norm, pi_fit)) +
  geom_abline(slope = 1, intercept = 0, color = "grey80", size = 2) +
  geom_errorbar(
    aes(ymin = pi_lwr, ymax = pi_upr),
    width = 0
  ) +
  geom_point() +
  facet_grid(~model, labeller = label_both) +
  theme_minimal() +
  labs(
    title = "Predicted vs Actual",
    x = "Actual T_norm",
    y = "Predicted T_norm"
  )
```

![](c11-psaap-assignment_files/figure-gfm/q5-task-1.png)<!-- -->

**Observations**:

- Which model tends to be more accurate? How can you tell from this
  predicted-vs-actual plot?
  - q4 is more accurate. I can tell as the points follow the reference
    line (for y = x) closest.
- Which model tends to be *more confident* in its predictions? Put
  differently, which model has *narrower prediction intervals*?
  - q4 is more confident as the error bars are smaller than the x_only
    model.
- How many predictors does the `fit_simple` model need in order to make
  a prediction? What about your model `fit_q4`?
  - fit simple only has 1 predictor input while q4 has all inputs. The
    variables removed were either outputs or model metadata.

Based on these results, you might be tempted to always throw every
reasonable variable into the model. For some cases, that might be the
best choice. However, some variables might be *outside our control*; for
example, variables involving human behavior cannot be fully under our
control. Other variables may be *too difficult to measure*; for example,
it is *in theory* possible to predict the strength of a component by
having detailed knowledge of its microstructure. However, it is
*patently infeasible* to do a detailed study of *every single component*
that gets used in an airplane.

In both cases—human behavior and variable mat=erial properties—we would
be better off treating those quantities as random variables. There are
at least two ways we could treat these factors: 1. Explicitly model some
inputs as random variables and construct a model that *propagates* that
uncertainty from inputs to outputs, or 2. Implicitly model the
uncontrolled the uncontrolled variables by not including them as
predictors in the model, and instead relying on the error term
$\epsilon$ to represent these unaccounted factors. You will pursue
strategy 2. in the following Case Study.

# Case Study: Predicting Performance Ranges

### **q6** You are consulting with a team that is designing a prototype heat transfer device. They are asking you to help determine a *dependable range of values* for `T_norm` they can design around for this *single prototype*. The realized value of `T_norm` must not be too high as it may damage the downstream equipment, but it must also be high enough to extract an acceptable amount of heat.

In order to maximize the conditions under which this device can operate
successfully, the design team has chosen to fix the variables listed in
the table below, and consider the other variables to fluctuate according
to the values observed in `df_psaap`.

| Variable | Value    |
|----------|----------|
| `x`      | 1.0      |
| `L`      | 0.2      |
| `W`      | 0.04     |
| `U_0`    | 1.0      |
| (Other)  | (Varies) |

Your task is to use a regression analysis to deliver to the design team
a *dependable range* of values for `T_norm`, given their proposed
design, and at a fairly high level `0.8`. Perform your analysis below
(use the helper function `add_uncertainties()` with the `level`
argument!), and answer the questions below.

*Hint*: This problem will require you to *build a model* by choosing the
appropriate variables to include in the analysis. Think about *which
variables the design team can control*, and *which variables they have
chosen to allow to vary*. You will also need to choose between computing
a CI or PI for the design prediction.

``` r
# NOTE: No need to change df_design; this is the target the client
#       is considering
df_design <- tibble(x = 1, L = 0.2, W = 0.04, U_0 = 1.0)
# NOTE: This is the level the "probability" level customer wants
pr_level <- 0.8

## TODO: Fit a model, assess the uncertainty in your prediction,
#        use the validation data to check your uncertainty estimates, and
#        make a recommendation on a *dependable range* of values for T_norm
#        at the point `df_design`
fit_q6 <- lm(T_norm ~ x + L + W + U_0, data = df_train)


df_validate$pred_q6 <- predict(fit_q6, newdata = df_validate)

rmse_q6 <- rmse(df_validate$T_norm, df_validate$pred_q6)

print(rmse_q6)
```

    ## [1] 0.2220979

``` r
df_intervals <-
  df_validate %>%
  add_uncertainties(fit_q6, interval = "prediction", prefix = "pi", level = pr_level)

df_intervals_d <-
  df_design %>%
  add_uncertainties(fit_q6, interval = "prediction", prefix = "pi", level = pr_level)

df_intervals_d %>%
  pull(pi_lwr, pi_upr)
```

    ## 2.2964256247594 
    ##         1.45685

``` r
df_check <- data.frame(
  total = nrow(df_validate),
  total_in_range = sum(df_validate$T_norm > df_intervals$pi_lwr & df_validate$T_norm < df_intervals$pi_upr),
  in_range = mean(df_validate$T_norm > df_intervals$pi_lwr & df_validate$T_norm < df_intervals$pi_upr)
)
df_check
```

    ##   total total_in_range  in_range
    ## 1    60             56 0.9333333

``` r
print(df_intervals_d)
```

    ## # A tibble: 1 × 7
    ##       x     L     W   U_0 pi_fit pi_lwr pi_upr
    ##   <dbl> <dbl> <dbl> <dbl>  <dbl>  <dbl>  <dbl>
    ## 1     1   0.2  0.04     1   1.88   1.46   2.30

**Recommendation**:

- How much do you trust your model? Why?
  - I generally trust this model as it returns an rmse of 0.22 being a
    relatively good fit.
- What kind of interval—confidence or prediction—would you use for this
  task, and why?
  - Prediction since we want to capture new observations and not the
    mean trend.
- What fraction of validation cases lie within the intervals you
  predict? (NB. Make sure to calculate your intervals *based on the
  validation data*; don’t just use one single interval!) How does this
  compare with `pr_level`?
  - the fraction is 56/60 results in 93.3% and the level of probability
    the customer wanted was 80%
- What interval for `T_norm` would you recommend the design team to plan
  around?
  - pi_lwr is 1.45685 and pi_upr is2.29426. I would recommend to stay in
    those bounds.
- Are there any other recommendations you would provide?
  - To look at the breakdown of highly correlated variables and pick the
    highest correlated ones that are best suited for their design.
    Perhaps radiation intensity is the fourth most correlated input but
    it isn’t important to their design.

*Bonus*: One way you could take this analysis further is to recommend
which other variables the design team should tightly control. You could
do this by fixing values in `df_design` and adding them to the model. An
exercise you could carry out would be to systematically test the
variables to see which ones the design team should more tightly control.

# References

- \[1\] Jofre, del Rosario, and Iaccarino “Data-driven dimensional
  analysis of heat transfer in irradiated particle-laden turbulent
  flow” (2020) *International Journal of Multiphase Flow*,
  <https://doi.org/10.1016/j.ijmultiphaseflow.2019.103198>
