---
title: "Additional notes for Cox models"
author: "Devin Incerti"
---




# Data

First, let's simulate some data:


```r
library(prodlim)
set.seed(43500)
train_dat <- SimSurv(200)
set.seed(2721)
test_dat <- SimSurv(2)
test_pred <- test_dat[, 5:6]
.times <- c(1, 5, 10)
```

# Cox PH models with left truncation or interval censoring
Left truncated survival data is fairly common in practice. `survival::coxph()` can fit such models and survival functions can be predicted using `survival::survfit.coxph()` or with `pec::predictSurvProb()`; however, in the latter case, a warning is returned.


```r
library(survival)
library(pec)
cph_mod_lt <- coxph(Surv(time/2, time, status) ~ (X1 + X2)^2, data = train_dat,
                    x = TRUE)

# Survival predictions
summary(survfit(cph_mod_lt, newdata = test_pred), times = .times) 
```

```
Call: survfit(formula = cph_mod_lt, newdata = test_pred)

 time n.risk n.event survival1 survival2
    1     22       8     0.630  0.195911
    5     81      52     0.323  0.018511
   10     15      41     0.065  0.000065
```

```r
predictSurvProb(cph_mod_lt, newdata = test_pred, times = .times)
```

```
Warning in riskRegression::predictCox(object = object, newdata = newdata, : The current version of predictCox was not designed to handle left censoring 
The function may be used on own risks 
```

```
      [,1]   [,2]    [,3]
[1,] 0.630 0.3226 6.5e-02
[2,] 0.196 0.0185 6.5e-05
```

`survival::coxph()` does not support interval censoring:


```r
coxph(Surv(time/2, time, type = "interval2") ~ (X1 + X2)^2, data = train_dat)
```

```
Error in coxph(Surv(time/2, time, type = "interval2") ~ (X1 + X2)^2, data = train_dat): Cox model doesn't support "interval" survival data
```

# General "hack" for predicting survival from Cox models
Survival predictions can be made from a Cox model by running `survival::coxph()` with fixed parameters and then calling `survival::survfit.coxph()`. We illustrate using `mboost::glmboost()`:



```r
library(mboost)
f <- Surv(time, status) ~ X1 + X2
glmb_mod <- glmboost(f,data = train_dat, family = CoxPH())
```

`mboost` provides a `survFit()` function for predicting survival functions, but there is no explicit time argument:


```r
glmb_surv1 <- survFit(glmb_mod, test_pred) %>%
  {.[c("time","surv")]} %>%
  do.call("cbind", .) %>%
  as_tibble()
glmb_surv1
```

```
# A tibble: 107 x 3
    time   `1`   `2`
   <dbl> <dbl> <dbl>
 1 0.435 0.997 0.984
 2 0.537 0.993 0.967
 3 0.603 0.989 0.951
 4 0.635 0.986 0.935
 5 0.776 0.982 0.919
 6 0.822 0.979 0.903
 7 0.900 0.975 0.887
 8 0.926 0.972 0.872
 9 1.08  0.968 0.856
10 1.09  0.965 0.841
# … with 97 more rows
```

We can get around this with the `survival::survfit()` "hack":

```r
coxfit <- survival::coxph(f, data = train_dat,
                          init = coef(glmb_mod)[-1],
                          control = survival::coxph.control(iter.max = 0))
summary(survfit(coxfit, newdata = test_pred), times = glmb_surv1$time[1:3]) 
```

```
Call: survfit(formula = coxfit, newdata = test_pred)

  time n.risk n.event survival1 survival2
 0.435    200       1     0.997     0.984
 0.537    199       1     0.993     0.967
 0.603    197       1     0.989     0.951
```
