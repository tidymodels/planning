---
title: "Tidyup X: Adding post-processing operations to workflows"
execute:
  keep-md: true
---



# Tidyup X: Adding post-processing operations to workflows

**Champion**: Max

**Co-Champion**: Simon

**Status**: Draft

## Abstract

tidymodels workflow objects include pre-processing and modeling steps. Pre-processors are data-oriented operations to prepare the data for the modeling function. We will be including post-processors that can do various things to model outputs (e.g. predictions). This tidyup focuses on the APIs and data structures inside of a workflow object. 


## Motivation

After modeling is complete there are occasions where model outputs, mostly predictions, might requirement modification. For example, calibration of model predictions is an effective technique for improving models. Also, some users may wish to optimize the probability threshold to call a sample an event (in a binary classification problem). 

## Solution

### User-facing APIs

The plan is to have a small set of à la carte functions for specific operations. For example, let's look at a binary classification model with calibration and threshold optimization: 


::: {.cell}

```{.r .cell-code}
wflow_2 <- 
  workflow() %>% 
  add_formula(Class ~ .) %>% 
  add_model(logistic_reg())

wflow_individ <- 
  wflow_2 %>% 
  add_prob_calibration(cal_object) %>% 
  add_prob_threshold(threshold = 0.7)
```
:::


The new `add_*` functions would have `update_*()` and `remove_*()` analogs or overall functions that conduct those operations for all of the different post-processors. 

Note that the order of operations matter. In the above example, the system should execute the calibration before threshold the probabilities since the calibration will change the data in a way that invalidates the new threshold. 

However, the order that the operations are added to the worklflow should not matter. There are some rules that can be enacted (e.g. calibration before thresholding) but  the proposed solution is to have a prioritization scheme that can be, within reason, altered by the user. Each `add_*` function will have a `.priority` argument that has reasonable defaults (with one exception, see below).  

### Current list of post-processors

* Calibration of probability estimates to increase their probabilistic fidelity to the observed data

```r
# 'object' is a pre-made calibration object
# Data Inputs: class probabilities
# Data Outputs: class probabilities and recomputed class predictions
add_prob_calibration(object, .priority = 1.0)
```

* For binary classification outcomes, there may be the need to optimize the probability threshold that produces hard class predictions.

```r
# Potentially tunable
# Data Inputs: class probabilities
# Data Outputs: class predictions
add_prob_threshold(threshold = numeric(), .priority = 2.0)
```

* An addition of an equivocal zone where highly uncertain probability estimates where no hard class prediction is reported. 

```r
# Potentially tunable
# Data Inputs: class probabilities
# Data Outputs: class predictions
add_cls_eq_zone(value = numeric(), threshold = numeric(), .priority = 3.0)
```

* Calibration of regression predictions to avoid regression to the mean or other systematic issues with models.

```r
# 'object' is a pre-made calibration object
# Data Inputs: regression predictions
# Data Outputs: regression predictions
add_reg_calibration(object, .priority = 1.0)
```

* A general "mutate" operations where users can do on-the-fly operations using self-contained functions. For example,

  - If a user log-transformed their data prior to model, then can exponentiate them here. 
  - For regression models, the predictions can be thresholded to be within a specific range. 

```r
# User will have to always set the priority
# Data Inputs: all predictions
# Data Outputs: all predictions (only these columns are retained)
add_post_mutate(..., .priority)
```

This is likely to be a fairly complete menu of possible options. For example, at a later date, nearest-neighbor adjustment of regression predictions (see [this blog post](https://rviews.rstudio.com/2020/05/21/modern-rule-based-models/)) would be an interesting option.

## Implementation

There are implementation issues within the workflows and tune packages to discuss. There is also the idea that a small side-package will keep the underlying functions and data structures (for now at least). We'll tentatively call this the `dwai`  package (pronounced "duh-why" and short for Don't Worry About It).

### dwai

This package will contain

 - A container for the list of possible post-processors specified by the user.
 - A validation system to resolve conflicts in type or priority. 
 - An interface to apply the operations to the predicted values. 
 - The requisite package dependencies (primarily the probably package)
 
The dwai code may eventually make its way into workflows or probably. 

### Workflow structure

Workflows already contain a placeholder for post-processing: 




::: {.cell}

```{.r .cell-code}
library(tidymodels)

wflow_1 <- 
  workflow() %>% 
  add_model(linear_reg()) %>% 
  add_formula(mpg ~ .)
names(wflow_1)
```

::: {.cell-output .cell-output-stdout}
```
[1] "pre"     "fit"     "post"    "trained"
```
:::
:::


There are also existing helper functions that may be relevant: 

 - `order_stage_{pre,fit,post}` is used to resolve data priorities. For example, in pre-processing, case weights are [evaluated before the pre-processor method](https://github.com/tidymodels/workflows/blob/main/R/action.R#L36:L42). 
 
 - `new_stage_post()`, and `new_action_post()` are existing constructors. 

We will require a `.fit_post(workflow, data)` that will execute _only_ the post-processing operations; the `workflows` object will already have trained the `pre` and `fit` stages. 

### tune

The tune package can currently handle pre-processors and models; either stage can be tuned over multiple parameters. tune handles this in a conditional way. It separates out the pre-processing and model tuning parameters and first loops over the pre-processing parameters (if any). Within that loop, another loop tunes over the model parameters (if any). 

For post-processing, we will have to add another nested loop that evaluates post-processing tuning parameters in a way that doesn't require recomputing any model or pre-processing items. 

This has the potential to be computationally expensive and adds more complexity to the tune package.  

Our current list of post-processors only includes two tuning parameters: threshold for equivocal zones and probability thresholding. These are simple (and fast) operations and should not significant add to the computational burden.

Future operations might be more expensive. For example, see the section below on "Calibration tuning". 

## Backwards compatibility

There should be no issues since we are adding functionality that is independent of the current workflow capabilities.


## Aside: Calibration tuning

