Implementation proposal for case weights
================

I’ve ordered the packages listed here in terms of cross-package effect.
The first few packages have interfaces that are the “easiest” to design
(since they are user-facing), and then we discuss the lower
infrastructure packages. The case weights implementation for the
infrastructure packages has been designed purely to support the various
combinations of higher level user-facing packages.

One other thing to keep in mind is that different steps of the process
might require different types of case weights. Additionally, some parts
might require case weights while others don’t. This proposal tries to be
flexible to most use cases.

# yardstick

-   Relevant metrics gain a `case_weights` argument that supports
    tidyselect to specify which column should be used as the weighting
    column.

-   How does this work with `metric_set()`?

    -   Ideally just `metrics(truth, estimate, case_weights = blah)`
        like any other metric, but we will have to consider what happens
        when some metrics support case weights and others don’t.

-   Case weight support might be rolled out over multiple yardstick
    releases, but is something that could be done separately from other
    tidymodels packages.

# rsample

rsample is probably the most straightforward of the bunch, since it is
separate from everything else.

-   Where applicable, an rsample function will gain a `case_weights`
    argument that works using tidyselect to select a single column that
    will be used as the case weights.

    -   Possibly only frequency weights are relevant for rsample (which
        means that only integers are allowed)

-   (Note: Hopefully not needed) It is possible that we will need a
    second `case_weights_type` argument that specifies the type of case
    weight, like `"proportional"` or `"frequency"`, since frequency
    weights might need their own separate implementation in rsample.

-   The current thinking is that we will just store the name of the
    column in the rsample object, and then when `analysis()` and
    `assessment()` is called, the resamples will be sliced out from the
    original data frame and then the weights column will be recomputed
    to better reflect the updated number of rows. This is still up in
    the air.

    -   Possibly need `in_weights` and `out_weights` (mimic `in_id` and
        `out_id`)

# parsnip

parsnip is slowly becoming more of a “lower level” package, in that you
specify the model spec but rarely use `parsnip::fit()` directly, instead
going through a workflow or using tune. Because of that, I think that
`parsnip::fit()` and `fit_xy()` should get a new `case_weights` argument
that simply accepts a vector of weights. It might make sense to add a 
new flag to the `set_encoding()` functions that define new models (example [here](https://github.com/tidymodels/parsnip/blob/master/R/linear_reg_data.R#L22:L32))
to check for the ability for a model to use case weights. 

-   This is the obvious choice for
    `fit_xy(x, y, …, case_weights = NULL)` , where `x` and `y` are
    already separated into predictors/outcomes, and it doesn’t make
    sense to give `case_weights` a tidyselect interface that selects
    from `x`.

-   For `fit(formula, data, ..., case_weights = NULL)` , one could argue
    that `case_weights` should be tidyselect and select from `data`
    before applying the `formula`. However, I think it is better to be
    consistent with `fit_xy()`, and it will be easier to pass on to
    `parsnip::fit()` from higher level packages like workflows/tune if a
    simple vector is being passed through.

    -   Consider `y ~ x + case_weights(z)`

It will be up to parsnip to check if a specific model allows weights or
not, and possibly error at `fit()` time if weights were specified for a
model that doesn’t support them.

# recipes

recipes is a bit tricky, because none of our other preprocessing methods
(formula or variables) have any interactions with weighting schemes. But
for recipes, you might imagine someone using `step_center()` which might
need to computed a *weighted* mean at `prep()` time, which it can then
apply at `bake()` time. The weights would come from one of the columns
already in the data.

That said, not all of the steps will support weights, and it will be
hard to roll out the ones that do all at once.

Because of this, I propose that weights be specified on a *step-by-step
basis*. If a particular step allows weights, it should gain a
`case_weights` argument that allows tidyselect syntax to select a single
column that should be used for case weights in that method.

A possible alternative is to allow a user to specify a column with a
`"case_weight"` role which would be globally applied to all steps that
allow case weights. There are a few downsides to this:

-   We probably won’t be able to add case weight support everywhere at
    once. So new releases of recipes might change existing code where
    steps previously didn’t work with case weights, but now
    “automatically” do.

-   What about the potential case where different weighting schemes need
    to be applied to different steps?

-   How would you opt out if you needed to?

-   It may be difficult to have visible documentation of which steps
    support case weights. On the other hand, an argument is very
    explicit and visible.

It remains to be seen if there are steps that will require weights at
*both* `prep()` and `bake()` time. The `step_center()` example would not
require the weights column at bake-time, since the weighted mean would
just be stored. But some other steps might need it (possibly knn
imputation). The optional presence of the weights column at `bake()`
time is something that will affect how `workflows::predict()` and
`hardhat::forge()` work, and guides the workflows and hardhat proposals
below.

It will be useful to add a `"case_weight"` role to your case weights, as
this affects selectors like `all_predictors()`, and will categorize your
case weights column as an “extra” role in hardhat, which (with the
proposal below) will mean that it will not be a required column for
`hardhat::forge()` (and through that, `workflows::predict()`).

# workflows

This proposal asserts that workflows should only be concerned with
*model case weights*, i.e. the ones that will get passed on to parsnip
`fit()` or `fit_xy()`. Pre-processing specific case weights will be
handled through recipes.

With that in mind, I propose that `add_model()` gain two new optional
arguments:

-   `case_weights`: an optional tidyselect specified column to use as
    case weights, which would get extracted from `data` at `fit()` time
    and passed on to parsnip’s fitting functions.

-   `case_weights_remove`: a boolean flag, defaulting to `TRUE`, that
    determines whether or not to *remove* the `case_weights` column in
    `data` after it has been extracted, but before any pre-processing
    has been applied.

    -   For formula and variables pre-processors, you always want this
        to be `TRUE`.

    -   For recipe pre-processors where no case weights are being used,
        you want this to be `TRUE`.

    -   For recipe pre-processors where you have *exactly the same* case
        weights that will be passed on to you model, set this to `FALSE`
        to ensure that the specified `case_weights` column is retained
        in `data` for usage in the recipe.

# hardhat

The main problem that hardhat needs to overcome is knowing whether or
not to require case weight columns at `forge()` time, as this affects
how restrictive `workflows::predict()` is.

In general, case weights should not be required for prediction. However,
in some situations a recipe may require the `case_weights` column to be
able to `bake()` new data (i.e. to make a prediction). We feel that this
is relatively rare (one example of this is if `step_knn_impute()`
hypothetically accepted case weights, which would also be required at
bake-time).

This problem only affects recipes, but is a little bit more general than
just case weights. In general, it is possible for any non-standard
recipes role (i.e. not `"outcome"` or `"predictor"`) to be required only
at `prep()` time, and not at `bake()` time. In fact, this is probably
the most common scenario.

Because of this, the proposed change here is for `forge()` to only
require the `"outcome"` and `"predictor"` role columns by default, and
ignore the non-standard roles. Even if those columns existed, by default
they would not be passed on to the internal `bake()` call and would not
be returned in the `forge()$extras$roles` slot. To override this,
`hardhat::default_recipe_blueprint()` would gain a new argument,
`extra_bake_roles`, that would accept a character vector of non-standard
roles that are actually required at bake time.

# tune

Most of the changes from above would automatically flow through to tune.
However, tune will still need to be able to extract the `case_weights`
column from the assessment data if the user requested case-weighted
performance calculations. This requires accessing the name of the case
weight column specified in the workflow (which may be through some
helper that workflows exports), which can then be used to extract the
relevant column from the assessment data.

tune internally calls `forge_from_workflow()`, but this is only for
computing the predictions themselves, and the case weights column will
not be extracted from the result of this.

tune also needs a user specified way to opt out of using model case
weights for performance calculations, which will likely be through a
control argument.
