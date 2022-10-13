

# Package changes for censored regression models

These are notes from a session where we discussed the required changes to tidymodels. Some open issues are:

* Where does the user specify the time points for certain performance metrics (e.g. ROC methods)? There could be an option to the `tune` functions (or their respective control functions) for this. Alternatively, `.time` could be encoded into a metric set. In either case, the `.time` vector will be passed to the `parsnip` predict calls (when appropriate) so that the right predictions are created. We will _not_ have a `.time` argument to `yardstick` metrics since the times will be in the results of the prediction functions. [Note, the argument to `predict()` is called `time`, the column in the prediction object is called `.time`. In the rest of these notes, we use `.time` as the name for the vector of time at which we predict survival probability or hazard.]

* `tune` has some functions that require the user to pick a metric. For example, `tune_bayes()` will optimize on the first metric in the metric set. Functions like `select_best()` or `autoplot()` have specific arguments for 1+ metrics. It is unclear how the user should pick a metric when some of them are computed at specific times. 

* There was some discussion about packages like `workflows` (and maybe `recipes`) making the time/event columns into a `Surv` object. This would simplify the changes to `tune` and other packages but increase the complexity to `workflows`. Our current thinking is to require the user to make a `Surv` object in a column in the data frame, prior to any analysis. This follows the recommendation to do any modifications of the response prior to recipes etc.

Details on changes, per package, follow. 

## parsnip

* [x] Allow a new mode of  `"censored regression"` ([done](https://github.com/tidymodels/parsnip/pull/396))
* [x] Allow a new prediction types for linear predictors, time, survival probabilities, and hazard probabilities.  ([done](https://github.com/tidymodels/parsnip/pull/396))
* [x] Restrict `fit_xy()` from working with censored regression models. ([done](https://github.com/tidymodels/parsnip/pull/445))
* [x] Document that the linear predictor prediction type will always be increasing with time. ([done](https://github.com/tidymodels/parsnip/pull/396))
* [ ] Add linear predictor prediction types for existing parametric models (e.g. `glm` and so on)
* [x] Add internal functions to enable censored regression models to make all types of predictions (e.g. time or probability based). 
* [x] Add more models (on-going in the `censored` package). 
* [ ] Enable `augment()` for the `"censored regression"` mode.
* [ ] Relax the restriction on `fit_xy()` to allow fit via matrix interface if the outcome is a `Surv` object.

## rsample

* [x] Prevent the user from passing `Surv` objects to the `strata` argument (or just extract the times when they do). ([done](https://github.com/tidymodels/rsample/pull/231))

## yardstick

* [ ] Add new metric types for those that use censored regression probabilities (that require a time specification) and event time metrics (which are not dynamic). Main question for dynamic measures: where do we but `.time`? In an added column to the yardstick output object? (Keep in mind that we'd have a similar problem for quantile regression, so maybe don't call it `.time`.)
* [ ] Add new metrics: start with non-dynamic measure such as concordance, possibly Brier score as the first dynamic one (then ROC)
* [ ] Determine how to create a metric set when the same metric is used with multiple times. 

## workflows

* [ ] Enable the workflow selector methods to accept the event type column as input.
* [ ] Determine if workflows should make the `Surv` object itself or leave the time and event columns as-is.
* [ ] Determine if workflows should default to `fit()` instead of `fit_xy()` for censored regression models (so we don't have to trigger it via supplying a `formula` in `add_model()`) -- if we don't relax the restriction on `fix_xy()`.

## tune

* [ ] Add an approach for `autoplot()` to show probability-based metrics over time.
* [ ] Consider cases where `.time` contains a large number of time points. 
* [ ] Determine how to select a metric in functions like `select_best()` when the same metric is used over multiple time periods. 
* [ ] Update the logic to match prediction types to metric types. 

## recipes

* [ ] Document a standard role type for censoring indicator columns. 
* [ ] Create `step_surv()` to create `Surv` objects.

## Documentation

* [ ] Collect all the places that need updating, across packages, tidymodels.org, and TMwR.
