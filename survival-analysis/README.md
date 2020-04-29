
# Survival Methods in tidymodels

There is significant interest in creating infrastructure for modeling
censored data using the tidymodels packages. This document is a summary
and discussion of what needs to be done and a survey of many of the
modeling approaches existing in R.

We’d like to get feedback, edits, and/or contributions. Please add
issues or a pull request in case for those. API suggestions are very
welcome.

# Missing components and open questions

What needs to be developed for tidymodels? `parsnip` already contains
some simple wrappers for parametric survival models but, to really be
useful, there is a fair amount of infrastructure needed.

This can be broken down into a few parts:

1.  model fit wrapper for `parsnip`

2.  new predictions *types*

3.  performance metrics for censored data.

The first item is fairly straight-forward. Since `parsnip` wraps model
functions, this should be the easy part.

There is some work for the second item. Many applications of censored
data are less focused on predicting the actual event time; the focus is
often on predicting the probability of the event (e.g. death) at
specific times. For this, a new prediction type is needed. Instead of
using `type = "prob"`, a different values could be used (e.g. `type =
"hazard"` or `type = "survival"`). However, these predictions are
*dynamic* since they are made at specific time points. For this reason,
`parsnip` and other packages will require a specific argument for the
time of interest (I nominate a common argument name of `.time`).

Since this is a big difference, it may be helpful to define a new
`parsnip` model mode, perhaps called “risk prediction” or “censored
regression” so that we can delineate the problem. For example, there is
an `rpart` engine for `decision_tree()` that can be used for
classification or regression. Without another mode, there would be
conflicts or the need to make alternate function names (e.g.
`survival_decision_tree()`, which is less attractive).

The third item, performance metrics, will also require some minor
modifications for dynamic predictions. A standard argument will be
required for survival metrics that tell the function what column has the
time of interest.

What about `Surv()`? This function is a mainstay of survival analysis in
S and R. It encapsulates the event time(s) and the censoring indicator.
Most functions standardize on this format.

The main issue is that, within data frames, it might be difficult to
work with. For example:

``` r
library(survival)
library(tidymodels)

data(lung) # usually use `Surv(time, status)` for these data

surv_obj <- Surv(lung$time, lung$status)
is.vector(surv_obj)
```

    [1] FALSE

``` r
is.matrix(surv_obj)
```

    [1] TRUE

``` r
lung$surv <- surv_obj
head(lung)
```

``` 
  inst time status age sex ph.ecog ph.karno pat.karno meal.cal wt.loss  surv
1    3  306      2  74   1       1       90       100     1175      NA   306
2    3  455      2  68   1       0       90        90     1225      15   455
3    3 1010      1  56   1       0       90        90       NA      15 1010+
4    5  210      2  57   1       1       90        60     1150      11   210
5    1  883      2  60   1       0      100        90       NA       0   883
6   12 1022      1  74   1       1       50        80      513       0 1022+
```

``` r
lung_tbl <- as_tibble(lung)
lung_tbl$surv <- surv_obj
lung_tbl %>% slice(1:6)
```

    # A tibble: 6 x 11
       inst  time status   age   sex ph.ecog ph.karno pat.karno meal.cal wt.loss surv[,"time"]
      <dbl> <dbl>  <dbl> <dbl> <dbl>   <dbl>    <dbl>     <dbl>    <dbl>   <dbl>         <dbl>
    1     3   306      2    74     1       1       90       100     1175      NA           306
    2     3   455      2    68     1       0       90        90     1225      15           455
    3     3  1010      1    56     1       0       90        90       NA      15          1010
    4     5   210      2    57     1       1       90        60     1150      11           210
    5     1   883      2    60     1       0      100        90       NA       0           883
    6    12  1022      1    74     1       1       50        80      513       0          1022
    # … with 1 more variable: [,"status"] <dbl>

Typically, `Surv` objects are not added as columns but are used when the
model formula is specified. For tidymodels, there may be some added
complexity since special, model-specific formulas might be needed to use
workflows. This is very similar to special formulas for generalized
additive models and mixed models.

It would probably be better to leave the columns alone and use `Surv()`
in the model formula. However, for recipes, the censoring column might
get treated as a predictor or outcome so a special role might be helpful
to make sure that the data are properly handled during model fitting.

# Where to start

## Metrics

The cleanest place to start this work is to create `yardstick` metric
functions since these are model-agnostic. The concordance statistic uses
a static, event time prediction and would be very straight-forward to
wrap the function from the survival package.

For dynamic metrics, such as the prediction error curve or survival ROC
metrics, the problem is a little more complex since an interface entry
point is needed for the time points of interest.

`metric_set()` will need to have a new `class` attribute. For example,
depending on the type of prediction needed for a metric, this attribute
is either `class_metric`, `prob_metric`, or `numeric_metric`. We’ll need
new classes for static and dynamic metrics.

## Model fitting

Once an official name for the new mode is determined, engines can be
added for models like `surv_reg()`, `decision_tree()`, `rand_forest()`,
and others. Some details are below for possible models. To begin, we
should be relatively selective for picking new models/engines.

## Prediction functions

These is a fair amount of complexity in the prediction modules. The type
and format of return results from `predict()` methods can be very
different and un-tidy. This alone raises the complexity of the problem.

For some models, there will have to be different sub-modules for
probability predictions that depend on the statistical distribution of
the data. For example, `survreg` can use a few different distributions
and translating the estimated parameters to these probabilities can be
complicated. Some of this, and other details, is already handled by the
`pec` package (as will be seen below). Also `mlr3` has implemented these
too so, at the least, good test cases will be available.

As previously mentioned, `predict(x, type = "survival")` will have a
mandatory `.time` argument for dynamic predictions.

Also, the convention in tidymodels is, when predicting, to produce a
tibble with as many rows as `new_data` and standardized column names.
For static predictions, `.pred_time` is reasonable and `.pred_survival`
and `.pred_hazard` might be used for the two respective probabilities.

If dynamic predictions are made, the result will be nested tibbles. For
example, if there were two samples being predicted at three time points,
the structure should look like:

``` r
pred_vals
```

    # A tibble: 2 x 2
      .pred_time .pred           
           <dbl> <list>          
    1       206. <tibble [3 × 2]>
    2       167. <tibble [3 × 2]>

``` r
pred_vals$.pred[[1]]
```

    # A tibble: 3 x 2
      .time .pred_hazard
      <dbl>        <dbl>
    1   100      0.00419
    2   200      0.00651
    3   500      0.0116 

``` r
tidyr::unnest(pred_vals, cols = c(.pred))
```

    # A tibble: 6 x 3
      .pred_time .time .pred_hazard
           <dbl> <dbl>        <dbl>
    1       206.   100      0.00419
    2       206.   200      0.00651
    3       206.   500      0.0116 
    4       167.   100      0.00588
    5       167.   200      0.00914
    6       167.   500      0.0164 

# Possible models

First, let’s simulate some data:

``` r
library(prodlim)
set.seed(43500)
train_dat <- SimSurv(200)
set.seed(2721)
test_dat <- SimSurv(2)
test_dat
```

``` 
  eventtime censtime   time event X1    X2 status
1    10.840    13.53 10.840     1  1 -0.51      1
2     0.963     2.28  0.963     1  1  1.18      1
```

``` r
test_pred <- test_dat[, 5:6]

# We will predict at these time points
.times <- c(1, 5, 10)
```

## Parametric Models

### `survival:::`[`survreg`](https://rdrr.io/cran/survival/man/survreg.html)

The model fitting function:

``` r
library(survival)
sr_mod <- survreg(Surv(time, status) ~ (X1 + X2)^2, data = train_dat)
```

The `predict()` function can generate predictions for the event time
and/or percentiles of the survival function via

``` r
predict(sr_mod, test_pred)
```

``` 
   1    2 
8.04 3.19 
```

``` r
predict(sr_mod, test_pred, type = 'quantile', p = c(.1, .5, .9))
```

``` 
     [,1] [,2]  [,3]
[1,] 2.54 6.66 12.31
[2,] 1.01 2.65  4.89
```

Note that the latter is in terms of the time units and are not
probabilities of survival by a specified time. Since these are
parametric models, probablity values can be derived
indirectly.

### `flexsurv::`[`flexsurvreg`](https://rdrr.io/cran/flexsurv/man/flexsurvreg.html)

The main parametric fitting function is

``` r
library(flexsurv)
flxs_mod <-
  flexsurvreg(Surv(time, status) ~ (X1 + X2) ^ 2,
              data = train_dat,
              dist = "weibull")
```

Prediction is done using the `summary()` method and the results come
back as a data frame with `tidy = TRUE`:

``` r
# Event time prediction
summary(flxs_mod, test_pred, type = "mean", tidy = TRUE)
```

``` 
   est  lcl  ucl X1    X2
1 7.12 6.01 8.25  1 -0.51
2 2.83 2.25 3.57  1  1.18
```

``` r
# Probabilities
summary(flxs_mod, test_pred, type = "survival", t = .times, tidy = TRUE)
```

``` 
  time      est      lcl    ucl X1    X2
1    1 0.983100 9.69e-01 0.9917  1 -0.51
2    5 0.673176 5.88e-01 0.7535  1 -0.51
3   10 0.215802 1.16e-01 0.3195  1 -0.51
4    1 0.901743 8.33e-01 0.9501  1  1.18
5    5 0.090590 2.41e-02 0.2115  1  1.18
6   10 0.000091 7.30e-08 0.0036  1  1.18
```

``` r
# Same for type = "hazard"
```

The `flexsurvspline()` function has the same
interface.

## `casebase::`[`fitSmoothHazard`](https://rdrr.io/cran/casebase/man/fitSmoothHazard.html)

The main fitting function is `fitSmoothHazard`:

``` r
library(casebase)
cb_mod <-
  fitSmoothHazard(status ~ time + (X1 + X2) ^ 2,
                  data = train_dat)
```

Note that `time` is modeled explicitly and with flexibility:

``` r
library(splines)
cb_sp_mod <-
  fitSmoothHazard(status ~ bs(time) + (X1 + X2) ^ 2,
                  data = train_dat)
```

And we can also do penalized estimation:

``` r
cb_glmnet_mod <-
  fitSmoothHazard(status ~ time + (X1 + X2) ^ 2,
                  data = train_dat, family = "glmnet")
```

Predictions for the cumulative incidence or survival probabilities are
obtained using `absoluteRisk`:

``` r
absoluteRisk(cb_mod, newdata = test_pred, time = .times, type = "CI")
```

``` 
 time             
    0 0.0000 0.000
    1 0.0352 0.218
    5 0.2761 0.891
   10 0.7797 1.000
```

``` r
absoluteRisk(cb_mod, newdata = test_pred, time = .times, type = "survival")
```

``` 
 time               
    0 1.000 1.00e+00
    1 0.965 7.82e-01
    5 0.724 1.09e-01
   10 0.220 3.08e-05
```

(Note that the argument `type` is only available in the development
version.)

## Cox PH Models

### `survival:::`[`coxph`](https://rdrr.io/cran/survival/man/coxph.html)

``` r
cph_mod <- coxph(Surv(time, status) ~ (X1 + X2)^2, data = train_dat, x = TRUE)
```

Both restricted mean survival time (RMST) and survival function
probabilities at specific times can be predicted with
`summary.survit()`:

``` r
cph_surv <- summary(survfit(cph_mod, newdata = test_pred), times = .times) 

# Event time prediction (RMST)
cph_surv$table
```

``` 
  records n.max n.start events *rmean *se(rmean) median 0.95LCL 0.95UCL
1     200   200     200    107   7.05     0.2759   7.29    6.21    8.43
2     200   200     200    107   2.75     0.0483   2.25    1.85    3.81
```

``` r
# Survival probabilities
cph_surv 
```

    Call: survfit(formula = cph_mod, newdata = test_pred)
    
     time n.risk n.event survival1 survival2
        1    191       8     0.974  8.46e-01
        5     96      52     0.718  1.18e-01
       10     15      41     0.133  2.29e-06

Survial probabilities can also be predicted using the `pec` package;

``` r
predictSurvProb(cph_mod, newdata = test_pred, times = .times)
```

``` 
      [,1]  [,2]     [,3]
[1,] 0.974 0.718 1.33e-01
[2,] 0.846 0.118 2.29e-06
```

See the document “cox\_notes\_extra” for more notes about this model.

### `glmnet`

The model fitting input for the predictors is a matrix and the response
data should be a `Surv()` object. We would have to make a new column for
the interaction effect.

``` r
library(glmnet)
glmn_fit <- glmnet(x = as.matrix(train_dat[, c("X1", "X2")]), 
                   y = Surv(train_dat$time, train_dat$event),
                   family = "cox")
```

`glmnet` produces predictions across a variety of penalty values. The
relative-risk can be obtained using `type =
"response"`:

``` r
predict(glmn_fit, as.matrix(test_pred), type = "response")
```

``` 
     s0    s1    s2   s3    s4   s5    s6   s7    s8    s9   s10   s11  s12  s13  s14  s15  s16
[1,]  1 0.963 0.929 0.90 0.873 0.85 0.829 0.81 0.837 0.892 0.945 0.997 1.05 1.09 1.14 1.18 1.22
[2,]  1 1.092 1.184 1.28 1.367 1.46 1.542 1.63 1.824 2.120 2.436 2.768 3.11 3.47 3.84 4.21 4.59
      s17  s18  s19  s20  s21  s22  s23  s24  s25  s26  s27  s28  s29 s30  s31  s32  s33  s34  s35
[1,] 1.26 1.30 1.34 1.37 1.40 1.43 1.46 1.48 1.50 1.53 1.55 1.57 1.58 1.6 1.61 1.63 1.64 1.65 1.66
[2,] 4.96 5.34 5.71 6.07 6.42 6.76 7.09 7.41 7.71 8.00 8.27 8.53 8.77 9.0 9.22 9.42 9.61 9.78 9.94
       s36   s37   s38  s39  s40   s41   s42   s43
[1,]  1.67  1.68  1.69  1.7  1.7  1.71  1.72  1.72
[2,] 10.09 10.23 10.36 10.5 10.6 10.70 10.79 10.88
```

## `mboost` models

The `mboost` package has a suite of functions for fitting boosted models
whose base learners are either trees, GLM’s, or GAM’s. Each can take an
argument of `family = CoxPH()` and each can make predictions on the
linear predictor.

For example, for a GLM model:

``` r
library(mboost)
glmb_mod <-
  glmboost(Surv(time, status) ~ X1 + X2,
             data = train_dat,
             family = CoxPH())
predict(glmb_mod, test_pred)
```

``` 
    [,1]
1 0.0399
2 1.6059
```

## Tree-Based Models

### `rpart:::`[`rpart`](https://rdrr.io/cran/rpart/man/rpart.html)

``` r
library(rpart)
rp_mod <- rpart(Surv(time, status) ~ X1 + X2, data = train_dat)
```

The basic invocation of `predict` generates the predicted event time:

``` r
predict(rp_mod, test_pred)
```

``` 
   1    2 
1.15 3.12 
```

The `pec` package can be used to fit an augmented `rpart` model and,
from this, we can get the survivor
probabilities:

``` r
pec_rpart_mod <- pecRpart(Surv(time, status) ~ X1 + X2, data = train_dat)
predictSurvProb(pec_rpart_mod, newdata = test_pred, times = .times)
```

``` 
      [,1]  [,2]   [,3]
[1,] 0.964 0.739 0.0612
[2,] 0.938 0.309     NA
```

### `ipred:::`[`bagging`](https://rdrr.io/cran/ipred/man/bagging.html)

``` r
library(ipred)
bag_mod <- bagging(Surv(time, status) ~ X1 + X2, data = train_dat)
```

When generating predictions, `survfit` objects is returned for each data
point being predicted. To get survival functions including median
survival:

``` r
bag_preds <- predict(bag_mod, test_pred)
bag_preds
```

    [[1]]
    Call: survfit(formula = Surv(agglsample[[j]], aggcens[[j]]) ~ 1)
    
          n  events  median 0.95LCL 0.95UCL 
    1112.00  679.00    7.58    7.29    7.91 
    
    [[2]]
    Call: survfit(formula = Surv(agglsample[[j]], aggcens[[j]]) ~ 1)
    
          n  events  median 0.95LCL 0.95UCL 
     676.00  552.00    2.43    2.43    3.32 

``` r
# Use summary.survfit() to get survival function (pec would also work)
# For the first sample
summary(bag_preds[[1]], times = .times)
```

    Call: survfit(formula = Surv(agglsample[[j]], aggcens[[j]]) ~ 1)
    
     time n.risk n.event survival std.err lower 95% CI upper 95% CI
        1   1083      27   0.9757 0.00462       0.9667       0.9848
        5    570     232   0.7369 0.01428       0.7094       0.7654
       10     16     404   0.0515 0.01177       0.0329       0.0806

``` r
# Median survival
library(purrr)
map_dbl(bag_preds, ~ quantile(.x, probs = .5)$quantile)
```

    [1] 7.58 2.43

### `party:::`[`ctree`](https://rdrr.io/cran/party/man/ctree.html)

``` r
library(party)
ctree_mod <- ctree(Surv(time, status) ~ X1 + X2, data = train_dat)
```

The basic invocation of `predict` generates the predicted event time.

``` r
predict(ctree_mod, test_pred)
```

    [1] 7.58 3.32

`pec` has a wrapper for this model. Since `party` uses S4 methods, this
just puts the model inside of a wrapper S3
class:

``` r
pec_ctree_mod <- pecCtree(Surv(time, status) ~ X1 + X2, data = train_dat)
all.equal(ctree_mod, pec_ctree_mod$ctree)
```

    [1] TRUE

``` r
predictSurvProb(pec_ctree_mod, newdata = test_pred, times = .times)
```

``` 
      [,1]  [,2]   [,3]
[1,] 0.964 0.739 0.0612
[2,] 0.947 0.302 0.3022
```

### `party:::`[`cforest`](https://rdrr.io/cran/party/man/cforest.html)

``` r
library(party)

set.seed(342)
cforest_mod <-
  cforest(Surv(time, status) ~ X1 + X2,
          data = train_dat,
          controls = cforest_unbiased(ntree = 100, mtry = 1))
```

The basic invocation of `predict` generates the predicted event time:

``` r
predict(cforest_mod, newdata = test_pred)
```

``` 
   1    2 
7.28 4.65 
```

Another S3 wrapper is available:

``` r
set.seed(342)
pec_cforest_mod <-
  pecCforest(Surv(time, status) ~ X1 + X2,
             data = train_dat,
             controls = cforest_unbiased(ntree = 100, mtry = 1))

all.equal(cforest_mod, pec_cforest_mod$forest)
```

    [1] TRUE

``` r
predictSurvProb(pec_cforest_mod, newdata = test_pred, times = .times)
```

``` 
      [,1]  [,2]   [,3]
[1,] 0.963 0.673 0.0803
[2,] 0.947 0.451 0.0472
```

### `randomForestSRC:::`[`rfsrc`](https://rdrr.io/cran/randomForestSRC/man/rfsrc.html)

``` r
library(randomForestSRC)
rfsrce_mod <- rfsrc(Surv(time, status) ~ X1 + X2, data = train_dat, ntree = 1000)
```

The `predict` function generates an object with classes `"rfsrc"`,
`"predict"`, and `"surv"`. It is unclear what is being predicted:

``` r
predict(rfsrce_mod, test_pred)$predicted
```

    [1]  56.6 143.8

The `survival` slot appears to be survival probabilities:

``` r
rfsrce_pred <- predict(rfsrce_mod, test_pred)
round(rfsrce_pred$survival, 2)
```

``` 
     [,1] [,2] [,3] [,4] [,5] [,6] [,7] [,8] [,9] [,10] [,11] [,12] [,13] [,14] [,15] [,16] [,17]
[1,] 1.00 0.98 0.98 0.98 0.98 0.97 0.97 0.97 0.95  0.93  0.93  0.93  0.93  0.93  0.93  0.92  0.92
[2,] 0.97 0.97 0.97 0.97 0.96 0.96 0.96 0.93 0.93  0.93  0.92  0.90  0.86  0.84  0.84  0.84  0.83
     [,18] [,19] [,20] [,21] [,22] [,23] [,24] [,25] [,26] [,27] [,28] [,29] [,30] [,31] [,32]
[1,]  0.92  0.92  0.92  0.92  0.92  0.92  0.92  0.92  0.92  0.92  0.92  0.90   0.9  0.90  0.88
[2,]  0.80  0.77  0.75  0.75  0.74  0.72  0.71  0.69  0.68  0.65  0.62  0.62   0.6  0.57  0.57
     [,33] [,34] [,35] [,36] [,37] [,38] [,39] [,40] [,41] [,42] [,43] [,44] [,45] [,46] [,47]
[1,]  0.88  0.88  0.85  0.85  0.85  0.85  0.85  0.85  0.85  0.84  0.84  0.81  0.81  0.81  0.78
[2,]  0.56  0.52  0.52  0.51  0.50  0.48  0.48  0.48  0.48  0.48  0.47  0.47  0.45  0.43  0.43
     [,48] [,49] [,50] [,51] [,52] [,53] [,54] [,55] [,56] [,57] [,58] [,59] [,60] [,61] [,62]
[1,]  0.78  0.78  0.78  0.74  0.74  0.74  0.74  0.74  0.74  0.74  0.74  0.73  0.73  0.73  0.73
[2,]  0.42  0.40  0.38  0.38  0.38  0.36  0.32  0.31  0.29  0.29  0.29  0.29  0.28  0.28  0.26
     [,63] [,64] [,65] [,66] [,67] [,68] [,69] [,70] [,71] [,72] [,73] [,74] [,75] [,76] [,77]
[1,]  0.71  0.71  0.69  0.69  0.69  0.68  0.68  0.68  0.67  0.63  0.63  0.63  0.63  0.60  0.60
[2,]  0.26  0.23  0.23  0.21  0.17  0.16  0.16  0.14  0.14  0.14  0.14  0.14  0.14  0.14  0.08
     [,78] [,79] [,80] [,81] [,82] [,83] [,84] [,85] [,86] [,87] [,88] [,89] [,90] [,91] [,92]
[1,]  0.60  0.57  0.56  0.53  0.50  0.50  0.50  0.48  0.43  0.42  0.39  0.39  0.34  0.33  0.30
[2,]  0.06  0.06  0.05  0.05  0.05  0.03  0.02  0.02  0.02  0.02  0.02  0.02  0.02  0.02  0.02
     [,93] [,94] [,95] [,96] [,97] [,98] [,99] [,100] [,101] [,102] [,103] [,104] [,105] [,106]
[1,]  0.30  0.25  0.25  0.23  0.23  0.17  0.12   0.12   0.05   0.05   0.05   0.04   0.03   0.03
[2,]  0.02  0.02  0.01  0.01  0.01  0.01  0.01   0.01   0.01   0.01   0.01   0.01   0.01   0.01
     [,107]
[1,]   0.03
[2,]   0.01
```

The columns appear to correspond to the “time of
interest”:

``` r
rfsrce_pred$time.interest
```

``` 
  [1]  0.435  0.537  0.603  0.635  0.776  0.822  0.900  0.926  1.080  1.091  1.202  1.307  1.344
 [14]  1.419  1.481  1.482  1.750  1.756  1.768  1.852  1.923  1.955  2.012  2.014  2.028  2.087
 [27]  2.098  2.231  2.254  2.313  2.356  2.359  2.406  2.427  2.743  2.837  2.872  2.984  2.987
 [40]  3.153  3.260  3.284  3.325  3.385  3.510  3.810  4.111  4.336  4.416  4.434  4.467  4.521
 [53]  4.601  4.654  4.740  4.806  4.857  4.881  4.889  4.938  5.061  5.090  5.099  5.118  5.221
 [66]  5.408  5.601  5.758  5.793  5.931  5.973  6.056  6.097  6.211  6.244  6.342  6.378  6.546
 [79]  6.722  6.871  7.277  7.291  7.401  7.492  7.580  7.914  7.924  8.024  8.193  8.350  8.428
 [92]  8.451  8.484  8.542  8.794  9.077  9.399  9.435  9.775  9.778  9.789 10.526 12.087 12.745
[105] 15.407 17.477 18.077
```

although these values do not bracket the observed times:

``` r
summary(train_dat$time)
```

``` 
   Min. 1st Qu.  Median    Mean 3rd Qu.    Max. 
   0.44    2.83    4.89    5.55    7.38   20.19 
```

# Model Prediction Scorecard

In most cases, the open circle means that this type of prediction is
facilitated *indirectly*. For example, using “tricks” related to
`survfit()`, methods in the `pec` package, or other means.

<table class="table" style="margin-left: auto; margin-right: auto;">

<thead>

<tr>

<th style="text-align:left;">

Package

</th>

<th style="text-align:left;">

Function

</th>

<th style="text-align:left;">

pred event time

</th>

<th style="text-align:left;">

survival probs

</th>

<th style="text-align:left;">

linear pred

</th>

</tr>

</thead>

<tbody>

<tr>

<td style="text-align:left;">

`survival`

</td>

<td style="text-align:left;">

`survreg`

</td>

<td style="text-align:left;">

✔

</td>

<td style="text-align:left;">

✖

</td>

<td style="text-align:left;">

✔

</td>

</tr>

<tr>

<td style="text-align:left;">

`flexsurv`

</td>

<td style="text-align:left;">

`flexsurvreg`

</td>

<td style="text-align:left;">

✔

</td>

<td style="text-align:left;">

✔

</td>

<td style="text-align:left;">

✔

</td>

</tr>

<tr>

<td style="text-align:left;">

`casebase`

</td>

<td style="text-align:left;">

`fitSmoothHazard`

</td>

<td style="text-align:left;">

✖

</td>

<td style="text-align:left;">

✔

</td>

<td style="text-align:left;">

✔

</td>

</tr>

<tr>

<td style="text-align:left;">

`survival`

</td>

<td style="text-align:left;">

`coxph`

</td>

<td style="text-align:left;">

✔

</td>

<td style="text-align:left;">

✔

</td>

<td style="text-align:left;">

✔

</td>

</tr>

<tr>

<td style="text-align:left;">

`glmnet`

</td>

<td style="text-align:left;">

`glmnet`

</td>

<td style="text-align:left;">

◯

</td>

<td style="text-align:left;">

◯

</td>

<td style="text-align:left;">

✔

</td>

</tr>

<tr>

<td style="text-align:left;">

`mboost`

</td>

<td style="text-align:left;">

`*boost`

</td>

<td style="text-align:left;">

◯

</td>

<td style="text-align:left;">

◯

</td>

<td style="text-align:left;">

✔

</td>

</tr>

<tr>

<td style="text-align:left;">

`rpart`

</td>

<td style="text-align:left;">

`rpart`

</td>

<td style="text-align:left;">

✔

</td>

<td style="text-align:left;">

◯

</td>

<td style="text-align:left;">

✖

</td>

</tr>

<tr>

<td style="text-align:left;">

`ipred`

</td>

<td style="text-align:left;">

`bagging`

</td>

<td style="text-align:left;">

✔

</td>

<td style="text-align:left;">

◯

</td>

<td style="text-align:left;">

✖

</td>

</tr>

<tr>

<td style="text-align:left;">

`party`

</td>

<td style="text-align:left;">

`ctree`

</td>

<td style="text-align:left;">

✔

</td>

<td style="text-align:left;">

◯

</td>

<td style="text-align:left;">

✖

</td>

</tr>

<tr>

<td style="text-align:left;">

`party`

</td>

<td style="text-align:left;">

`cforest`

</td>

<td style="text-align:left;">

✔

</td>

<td style="text-align:left;">

◯

</td>

<td style="text-align:left;">

✖

</td>

</tr>

<tr>

<td style="text-align:left;">

`randomForestSRC`

</td>

<td style="text-align:left;">

`rfsrc`

</td>

<td style="text-align:left;">

✔

</td>

<td style="text-align:left;">

◯

</td>

<td style="text-align:left;">

✖

</td>

</tr>

</tbody>

</table>
