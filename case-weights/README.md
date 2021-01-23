These are notes about how we will be incorporating case weights in to the tidymodels packages. 

I see two different use cases for case weights: 

1. Case weights are `n` for replicated covariate patterns. So a weight of 20 means that there were 20 rows with this same data pattern and we want to save memory by eliminating 19 redundant rows. 

1. Case weights signify how much importance each value should have. You might weight certain important data points higher for some reason. For example, in my last job, we would sometimes weight compounds inversely with their age (older data is less relevant). Fractional values make sense here. 

Possible lables for these two situations would be `pattern_count` for the first and `case_weight` for the second. 

We tend to think of case weights only in terms of the model but, depending on which of these use cases you are in, it could/should impact other computations, such as: 

* Data Splitting (case 1 only): should all of the replicate configurations be in the training or test set? Should bootstrapping or other resampling methods account for the case weights? 

* Preprocessing (both cases): Arguably, the weights should matter here too. PCA, centering, and scaling are three basic example where the preprocessing should respect these weights but their underlying functions (`prcomp()`, `mean()`, `sd()`) have no capacity for case weights. 

* Performance determination (both cases): If a row is a placeholder for _X_ number of data points, it should have a higher weight in the metric calculations. Otherwise it is under-valued in the statistics. 

## `parsnip` 

Change level:  🔥🔥🔥🔥 (out of 5)

**How should the user-facing API change to use case weights?** The two main functions that might be affects are `fit()` and `fit_xy()`. 

`fit()` takes arguments `formula` and `data`. It is reasonable to expect the case weights to be in `data`. Similar to [this other question about formulas](https://github.com/tidymodels/multilevelmod/issues/5), we could add a special term that signals which column corresponds to the case weight.  For example: 

```r
model_fit <- spec %>% fit(y + x_1 + x_2 + case_weight(wts), data = dat)
```

We might consider using different functions to differentiate the two use cases described above. 

`fit_xy()` doesn't take `data` but separate `x` and `y`. We can add a `case_weight` argument there. 

**Do model definitions change for case weights?** Yes since we need to check for each model/engine/mode combination that case weights are allowed for that model. This will also impact the `translate()` infrastructure so that the right column gets added to call to the underlying model fit function. 

## `recipes`

Change level:  🔥🔥🔥🔥

The user can change a role to be `"case_weight"` and we can document this as the canonical role. Again, we might want different roles for the two different types of weights described at the top. 

**Each step function** will require modifications _if_ their calculations can use case weights (some implementations in package dependencies may not accept them). For example, `prcomp()` does not accept case weights so `step_pca()` would not either. 

## `workflows`/`hardhat`

Change level:  🔥🔥

`hardhat` probably does [not need a lot of changes](https://github.com/tidymodels/hardhat/issues/15#issuecomment-464345264) but `workflows` probably will. It would need to use the same changes to the formula interface defined above (for `parsnip`)  and `add_variables()` should have a slot for case weights (and censoring indicators). Where these data columns are stored in the workflow should probably change too. 

## `yardstick`

Change level:  🔥🔥🔥🔥🔥

There are _a lot_ of changes here. For the user-facing API, we'll need a `case_weight` argument to specify the appropriate column name. 

Under the hood, all of the metrics would need to accommodate case weights. In most cases, the won't be too difficult (but tedious) but in others, how case weights affect the calculations may not be obvious. 

## `rsample`

Change level:  🔥🔥

The changes to `rsample` are a little ambiguous. For the two classes of case weight scenarios listed above, we would probably do different things. _This is probably the only situation where the two scenarios matter._ 

When case weights reflect the number of rows with that pattern, it is unclear how `rsample` should handle this. We could keep the rows together so that they all either go into the training or test set _or_ split them up. I can envision problems with biasing the performance metrics either way. 

## `tune`

Change level:  🔥🔥

`tune` will mostly change in how it handles the new roles and passing the data around. It's internal APIs will be affected by the changes listed above but the user-facing APIs are most likely unaffected.  



## List of specific changes

### Recipe steps

* `step_adasyn()`: Underlying implementation prevents changes

* `step_arrange()`: No changes

* `step_bin2factor()`: No changes

* `step_BoxCox()`: Changes required

* `step_bs()`: No changes

* `step_bsmote()`: Underlying implementation prevents changes

* `step_center()`: Straightforward changes

* `step_classdist()`: Straightforward changes

* `step_corr()`: Changes required. We can use `cov.wt()` instead of `cor()` (with fewer options) then convert to a correlation matrix via `solve(sqrt(diag(cov))) * cov * solve(sqrt(diag(cov)))`

* `step_count()`: No changes

* `step_cut()`: No changes

* `step_date()`: No changes

* `step_depth()`: Underlying implementation prevents changes

* `step_discretize()`: Changes required

* `step_discretize_cart()`: Changes required

* `step_discretize_xgb()`: Changes required

* `step_downsample()`: Changes required

* `step_dummy()`: No changes

* `step_embed()`: Underlying implementation prevents changes

* `step_factor2string()`: No changes

* `step_feature_hash()`: No changes

* `step_filter()`: No changes

* `step_geodist()`: No changes

* `step_holiday()`: No changes

* `step_hyperbolic()`: No changes

* `step_ica()`: Underlying implementation prevents changes

* `step_impute_bag()`: Changes required

* `step_impute_knn()`: Underlying implementation prevents changes

* `step_impute_linear()`: Changes required

* `step_impute_lower()`: No changes

* `step_impute_mean()`: Straightforward changes

* `step_impute_median()`: Straightforward changes

* `step_impute_mode()`: Straightforward changes

* `step_impute_roll()`:

* `step_integer()`: No changes

* `step_interact()`: No changes

* `step_intercept()`: No changes

* `step_inverse()`: No changes

* `step_invlogit()`: No changes

* `step_isomap()`: Underlying implementation prevents change

* `step_kpca()`: Underlying implementation prevents changes

* `step_kpca_poly()`: Underlying implementation prevents changes

* `step_kpca_rbf()`: Underlying implementation prevents changes

* `step_lag()`: No changes

* `step_lencode_bayes()`: Straightforward changes

* `step_lencode_glm()`: Straightforward changes

* `step_lencode_mixed()`: Straightforward changes

* `step_lincomb()`: No changes

* `step_log()`: No changes

* `step_logit()`: No changes

* `step_mutate()`: No changes

* `step_mutate_at()`: No changes

* `step_naomit()`: No changes

* `step_nearmiss()`: Underlying implementation prevents changes

* `step_nnmf()`: Underlying implementation prevents changes

* `step_normalize()`: Straightforward changes

* `step_novel()`: No changes

* `step_ns()`: No changes

* `step_num2factor()`: No changes

* `step_nzv()`: No changes

* `step_ordinalscore()`: No changes

* `step_other()`: Straightforward changes

* `step_pca()`: Underlying implementation prevents changes

* `step_pls()`: Underlying implementation prevents changes

* `step_poly()`: No changes

* `step_profile()`: No changes

* `step_range()`: Straightforward changes

* `step_ratio()`: No changes

* `step_regex()`: No changes

* `step_relevel()`: No changes

* `step_relu()`: No changes

* `step_rename()`: No changes

* `step_rename_at()`: No changes

* `step_rm()`: No changes

* `step_rose()`: Underlying implementation prevents changes

* `step_sample()`: Changes required

* `step_scale()`: Straightforward changes

* `step_shuffle()`: No changes

* `step_slice()`: No changes

* `step_smote()`: Underlying implementation prevents changes

* `step_spatialsign()`: Straightforward changes

* `step_sqrt()`: No changes

* `step_string2factor()`: No changes

* `step_tomek()`: Underlying implementation prevents changes

* `step_umap()`: Underlying implementation prevents changes

* `step_unknown()`: No changes

* `step_unorder()`: No changes

* `step_upsample()`: Changes required

* `step_window()`:

* `step_woe()`: Underlying implementation prevents changes

* `step_YeoJohnson()`: Changes required

* `step_zv()`: No changes

 

