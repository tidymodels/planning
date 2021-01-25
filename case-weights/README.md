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

** No changes

* `step_arrange()`

* `step_bin2factor()`

* `step_bs()`

* `step_count()`

* `step_cut()`

* `step_date()`

* `step_dummy()`

* `step_factor2string()`

* `step_feature_hash()` 

* `step_filter()`

* `step_geodist()`

* `step_holiday()`

* `step_hyperbolic()`

* `step_impute_lower()`

* `step_integer()`

* `step_interact()`

* `step_intercept()`

* `step_inverse()`

* `step_invlogit()`

* `step_lag()`

* `step_lincomb()`

* `step_log()`

* `step_logit()`

* `step_mutate()`

* `step_mutate_at()`

* `step_naomit()`

* `step_novel()`

* `step_ns()`

* `step_num2factor()`

* `step_nzv()`

* `step_ordinalscore()`

* `step_poly()`

* `step_profile()`

* `step_ratio()`

* `step_regex()`

* `step_relevel()`

* `step_relu()`

* `step_rename()`

* `step_rename_at()`

* `step_rm()`

* `step_shuffle()`

* `step_slice()`

* `step_sqrt()`

* `step_string2factor()`

* `step_unknown()`

* `step_unorder()`

* `step_zv()`

** Underlying implementation prevents changes

* `step_adasyn()`

* `step_bsmote()`

* `step_depth()`

* `step_embed()`

* `step_ica()`

* `step_impute_knn()`

* `step_isomap()`

* `step_kpca()`

* `step_kpca_poly()`

* `step_kpca_rbf()`

* `step_nearmiss()`

* `step_nnmf()`

* `step_pca()`

* `step_pls()`

* `step_rose()`

* `step_smote()`

* `step_tomek()`

* `step_umap()`

* `step_woe()`


** Straightforward changes

This means including an extra argument to the underlying call or similar api changes

* `step_center()`

* `step_classdist()`

* `step_corr()`. We can use `cov.wt()` instead of `cor()` (with fewer options) then convert to a correlation matrix via `solve(sqrt(diag(cov))) * cov * solve(sqrt(diag(cov)))`

* `step_discretize_cart()`

* `step_discretize_xgb()`

* `step_impute_bag()`

* `step_impute_linear()`

* `step_impute_mean()`

* `step_lencode_bayes()`

* `step_lencode_glm()`

* `step_lencode_mixed()`

* `step_normalize()`

* `step_scale()`

** Significant changes

This means defining new calculations or expanding the individual vectors using their weights. 

* `step_BoxCox()`

* `step_discretize()`

* `step_downsample()`

* `step_impute_median()`

* `step_impute_mode()`

* `step_impute_roll()`:

* `step_other()`

* `step_range()`

* `step_sample()`

* `step_spatialsign()`

* `step_upsample()`

* `step_window()`:

* `step_YeoJohnson()`

 
## `parsnip` models

Accepts case weights: 

 * `baguette::bag_mars` (earth)

 * `baguette::bag_tree` (C5.0)

 * `baguette::bag_tree` (rpart)

 * `parsnip::boost_tree` (C5.0)

 * `parsnip::boost_tree` (xgboost)

 * `rules::C5_rules` (C5.0)

 * `rules::cubist_rules` (Cubist)

 * `parsnip::decision_tree` (C5.0)

 * `parsnip::decision_tree` (rpart) 

 * `discrim::discrim_flexible` (earth)

 * `parsnip::linear_reg` (glmnet)

 * `parsnip::linear_reg` (lm)

 * `parsnip::linear_reg` (stan)

 * `parsnip::logistic_reg` (glm)

 * `parsnip::logistic_reg` (glmnet)

 * `parsnip::logistic_reg` (stan)

 * `parsnip::mars` (earth)

 * `parsnip::multinom_reg` (glmnet)

 * `parsnip::multinom_reg` (nnet)

 * `parsnip::mlp` (nnet)

 * `plsmod::pls` (mixOmics)

 * `poissonreg::poisson_reg` (glm)

 * `poissonreg::poisson_reg` (glmnet)

 * `poissonreg::poisson_reg` (hurdle)

 * `poissonreg::poisson_reg` (zeroinfl)

 * `poissonreg::poisson_reg` (stan)

 * `parsnip::rand_forest` (ranger)

 * `parsnip::surv_reg` (flexsurv)

 * `parsnip::surv_reg` (survival)


Does not allow case weights

* `modeltime::arima_boost` (arima_xgboost)

 * `modeltime::arima_boost` (auto_arima_xgboost)

 * `modeltime::arima_reg` (arima)

 * `modeltime::arima_reg` (auto_arima)

 * `discrim::discrim_linear` (MASS)

 * `discrim::discrim_linear` (mda)

 * `discrim::discrim_regularized` (klaR)

 * `modeltime::exp_smoothing` (ets)
 
 * `discrim::naive_Bayes` (klaR)

 * `discrim::naive_Bayes` (naivebayes)

 * `parsnip::nearest_neighbor` (kknn)

 * `parsnip::linear_reg` (keras)

 * `parsnip::logistic_reg` (keras)

 * `parsnip::mlp` (keras)

 * `parsnip::multinom_reg` (keras)

 * `modeltime::nnetar_reg` (nnetar)
 
 * `modeltime::prophet_boost` (prophet_xgboost)

 * `modeltime::prophet_reg` (prophet)

 * `parsnip::rand_forest` (randomForest)

 * `rules::rule_fit` (xrf)

 * `modeltime::seasonal_reg` (stlm_arima)

 * `modeltime::seasonal_reg` (stlm_ets)

 * `modeltime::seasonal_reg` (tbats)

 * `parsnip::svm_poly` (kernlab)

 * `parsnip::svm_rbf` (kernlab)

 * `parsnip::svm_rbf` (liquidSVM)






