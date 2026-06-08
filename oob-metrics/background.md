
## @hardin47 suggests:

In the [code you link to](https://github.com/juliasilge/juliasilge.com/issues/21#issuecomment-960253058) (two comments up) you get "folds" from bootstrapping and in your `tune_grid()` you use those bootstrap folds.

why not `tune_grid()` using OOB?  is there anything in **tidymodels** that will let you tune parameters using OOB?  can i hack some of the **caret** functionality to do some OOB work?

it seems like double the (computational) work to do extra bootstrapping instead of letting the free OOB values provide model information.

thank you!!!

## @juliasilge [responds](https://github.com/juliasilge/juliasilge.com/issues/21#issuecomment-961234835):

@hardin47 We don't super fluently getting those OOB samples out because we believe it is better practice to tune using a nested scheme, but if you want to see if it works out OK in your particular setting, you might want to check out [this article for handling](https://www.tidymodels.org/learn/work/nested-resampling/) some of the objects/approaches involved.


## @hardin47 [argues](https://github.com/tidymodels/planning/issues/25#issue-1614087478) for needing OOB functionality

In situations when running random forests (or other bagged models), OOB model information (predictions, error rates, etc.) should be available.  

1. First of all, I'm not convinced that OOB is a bad option.  In [this recent paper](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0201904) they say:

> In line with results reported in the literature [[5](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0201904#pone.0201904.ref005)], the use of stratified subsampling with sampling fractions that are proportional to response class sizes of the training data yielded almost unbiased error rates in most settings with metric predictors. It therefore presents an easy way of reducing the bias in the OOB error. It does not increase the cost of constructing the RF, since unstratified sampling (bootstrap of subsampling) is simply replaced by stratified subsampling.

Indicating that OOB errors **are** doing a good job of estimating error rates (with the added benefit that they require no additional model fitting) as long as stratified sampling is done instead of subsampling.

2. Even if nested resampling is superior (and I'll buy that there is an argument to be made), I find that cross validation and OOB are stepping stones to understanding nested resampling.   Do you argue that nested resampling is better than CV?  If so, why have CV in the package?  Again, OOB happens for free, and sometimes nested resampling isn't even *that* much better.  I think that more people will use nested resampling if they understand OOB, and the path to understanding OOB happens when it is included in the **tidymodels** package.

Thanks for all that you do!!  The **tidymodels** package is amazing, and I really appreciate all the hard work that has gone into creating it.


## @topepo [said the following](https://github.com/tidymodels/planning/issues/25#issuecomment-1458731856) when reflecting on the idea:

This is a good idea and I think that we should try to solve this systematically (and not just for ranger).

Other models have OOB errors but they come back in a different format (e.g. a OOB confusion table, etc). We might not be able to do something comprehensive across all models.

I think I have a solution but I won't be able to get to it right away. I've put a moratorium on new packages/features until we have made a lot of progress on case weights.

The idea would be to produce a tibble of specific characteristics of models. For example:

The number of terminal nodes in a tree.
The number of predictors actually used by the model.
etc. We would have an option to bundle these statistics into the results of the tune functions.

I could add a set of OOB statistics for ranger in the process of doing this.

A side note: you would probably want to avoid any external resampling if you can get OOB errors. In that case, you can use the (poorly named by me) apparent() function to make a resampling object. This specifies that the modeling and holdout sets are the same. This would avoid making multiple versions of the data to estimate performance.


