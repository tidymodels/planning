These are notes about how we will be incorporating case weights in to the itydmoels packages. 



I see two different use cases for case weights: 

1. Case weights are `n` for replicated covariate patterns. So a weight of 20 means that there were 20 rows with this same data pattern and we want to save memory by eliminating 19 redundant rows. 

1. Case weights signify how much importance each value should have. You might weight certain important data points higher for some reason. For example, in my last job, we would sometimes weight compounds inversely with their age (older data is less relevant). Fractional values make sense here. 

We tend to think of case weights only in terms of the model but, depending on which of these use cases you are in, it could/should impact other computations, such as: 

* Data Splitting (case 1 only): should all of the replicate configurations be in the training or test set? Should bootstrapping or other resampling methods account for the case weights? 

* Preprocessing (both cases): Arguably, the weights should matter here too. PCA, centering, and scaling are three basic example where the preprocessing should respect these weights but their underlying functions (`prcomp()`, `mean()`, `sd()`) have no capacity for case weights. 

* Performance determination (case 1 only): If a row is a placeholder for _X_ number of data points, it should have a higher weight in the metric calculations. Otherwise it is under-valued in the statistics. 



## `parsnip`

**How should the user-facing API change to use case weights?** The two main functions that might be affects are `fit()` and `fit_xy()`. 

`fit()` takes arguments `formula` and `data`. It is reasonable to expect the case weights to be in `data`. Similar to [this other question about formulas](https://github.com/tidymodels/multilevelmod/issues/5), we could add a special term that signals which column corresponds to the case weight.  For example: 

```r
model_fit <- spec %>% fit(y + x_1 + x_2 + case_weight(wts), data = dat)
```

`fit_xy()` doesn't take `data` but separate `x` and `y`. We can add a `case_weight` argument there. 

**Do model definitions change for case weights?** Yes since we need to check for each model/engine/mode combination that case weights are allowed for that model. This will also imact the `translate()` infrastructure so that the right column gets added to call to the underlying model fit funciton. 

## `workflows`/`hardhat`



## `yardstick`



## `rsample`



## `recipes`



## `tune`







