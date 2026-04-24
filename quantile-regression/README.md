# Quantile Regression

This discusses how we can include quantile regression in tidymodels. 

The discussion below is organized by package

## parsnip

Users will probably have to specify the quantiles at the specification time (see section on new models). 

### Engines and Modes

The main question is: Should we make new engines available via `set_engine()` or create a new mode of `"quantile regression"`? 

Pros of `set_engine()`: 

* Most (but not all) quantile regression predictions will be for numeric modes. 
* There is not much difference between quantile models that require a new mode (unlike censored regression). For example, our yardstick functions to compute loss for quantile regression can ingest the list column of predictions (similar to dynamic survival models). 
* Where do we specify the list of quantiles? It would be suboptimal to add a main option to each function. The users could set it in `set_engine(),`, but that would require more specialized parsing of the return value of that function. 

Cons of `set_engine()`: 

* There could be confusion around the type of model and prediction being made. Suppose that `rpart` could make quantile predictions (it cannot). Would use an engine of “part” result in the regular CART model or one optimized via quantile loss? 
* Since the list of quintiles would be specified upfront, it would be advantageous to do this with the new mode (`set_mode(“quantgreg,” quantiles = (1:9) / 10)`). 

To test using a new mode, there is a [parsnip branch](https://github.com/tidymodels/parsnip/tree/quantile-mode) that enables that so you can use: 

```
pak::pak(c("tidymodels/parsnip@quantile-mode"), ask = FALSE)
```

### New Models

Some packages already fit these models, notably quantreg and quantregForest. 

We can also make our own (I might do this with brulee). If so, the model functions should try to emulate the main arguments that we have. So for a neural network, use `hidden_units`, `penalty`, etc. 

Many models need a specific training run for each quantile value (since the loss depends on it). For this reason, we would have users specify the required quantiles when they specify the model. 

We should make a general class for these models in an engine-specific package. 

#### Predictions

We currently note tin `?predict.model_fit`: 

> For `type = "quantile"`, the tibble has a `.pred` column, which is a list-column. Each list element contains a tibble with columns `.pred` and `.quantile` (and perhaps other columns).

This should be changed so the list column is called `.pred_quantile` if we want `.pred` to contain the 50% quantile results. 

The format of the list column would be 

```
tibble(.quantile = numeric(), .pred_quantile = numeric())
```

## yardstick

We need one or more model performance metrics for quantile loss that work across all quantiles. They will need a new class too (say `“quantile_metric”`). 

## tune

We will need an `estimate_quantiles()` (analogous to functions such as [`estimate_class_prob()`](https://github.com/tidymodels/tune/blob/main/R/grid_performance.R#L108C1-L108C20)). 

Some specific callouts: 

- https://github.com/tidymodels/tune/blob/main/R/grid_performance.R#L4
- https://github.com/tidymodels/tune/blob/main/R/grid_performance.R#L98


