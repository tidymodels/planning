`tidymodels` has excellent out of bag (OOB) functionality for bagged models, but it does not easily provide the relevant OOB metrics.

Consider a Random Forest model.
    * the model can be fit with OOB predictions.
    * ideally there would be an option for OOB model tuning.
    * ideally there would be resulting OOB metrics for the final bagged model.


Nested models are great, but for pedagogical reasons, they are usually taught after CV and OOB.  

OOB is efficient, and having both parameter tuning as well as error metrics output would be very helpful.


### Functions to adjust

Here are just a few functions which might need tweaking.  In general, anything that uses cross validation could be adjusted to include an OOB option instead.

* `vfold_cv()`  I'm not sure what the equivalent would be.  The computer needs to do all of the work for each combination of parameters, so there shouldn't need to be a separate function (for creating "folds"), but somehow you need to communicate how the data are being broken up inside `tune_grid()`.
* `tune_grid()` calculates metrics across the "folds"  (i.e., OOB structure) for each of the hyperparameter options.
* `collect_metrics()` needs to have an option for OOB $R^2$ and OOB MSE (and others) for continuous regression models and OOB `roc_auc` OOB `accuracy` etc for binary classification models.


