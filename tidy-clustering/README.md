# Overview

`tidymodels` currently focuses on supervised learning methods, with unsupervised methods in [recipes](https://recipes.tidymodels.org/) and related-packages like [embed](https://embed.tidymodels.org/) mainly framed as preprocessing steps for supervised learning.  This document explores the possibility of including unsupervised methods in the same framework.

## Basic Definition

I'll use the term "clustering" to refer generally to any method that does the following:

1. Define a **measure of similarity** between objects.

2. Discover **groups of objects** that are similar.

Many of the thoughts in this section are taken from [this paper](http://proceedings.mlr.press/v27/luxburg12a/luxburg12a.pdf)

## Uses of Clustering

1. **Semi-Supervised Learning:** If we have class labels on *some* of the objects, we can apply unsupervised clustering, then let the clusters be defined by their class enrichment of labelled objects.

    +  A word of caution for this approach:  Just because a clustering structure
    doesn't align with known labels doesn't mean it is "wrong".  It could be 
    capturing a different (true) aspect of the data than the one we have labels for.

2. **EDA:** Sometimes clustering is applied as a first exploratory step, to get a sense of the structure of the data.  This is somewhat nebulous, and usually involves eyeballing a visualization.

    +  In my experience this is usually a precursor to classification.  We'd do
    an unsupervised clustering and see how well the objects "group up".  This 
    gives us an idea of whether the measurements we're using for classification
    are a good choice, before we fit a formal model.

3. **Pre-processing:** Clustering can be used to discover relationships in data that are undesirable, so that we can *residualize* or *decorrelate* the objects before applying an analysis.

    + A great example of this is in genetics, where we have measurements of gene
    expression for several subjects.  Typically, gene expression is most
    strongly correlated by race.  If we cluster the subjects on gene expression,
    we can then identify unwanted dependence to remove from the data.
    
4. **Clusters as analysis:** Sometimes, assignment of cluster membership *is* the end goal of the study. For example:

    +  In the Enron corruption case in 2001, researchers created a network based
    on who emailed who within the company.  They then looked at which clusters
    contained known consiprators, and investigated the other individuals in those
    groups.
    
    +  In the early days of breast cancer genetic studies, researchers clustered
    known patients on genetic expression, which led to the discovery of different
    tumor types (e.g. Basal, Her-2, Luminal).  These have later been clinically
    validated and better defined.


## Ways to validate a cluster

The major departure from supervised learning is this:  With a supervised method, we have a very clear way to measure success, namely, how well does it predict?

With clustering, there is no "right answer" to compare results againsts.

There are several ways people typically validate a clustering result:

1. **within-group versus without-group similarity:**  The goal is to find groups of similar objects.  Thus, we can check how close objects in the same cluster are as compared to how close objects in different clusters are.

     +  A problem with this is that there's not objective baseline about what is
     a "good" ratio. 
     +  (Gao, Witten, Bien have a cool new paper suggesting a statistical test
     for cluster concentration.)
     
     
2.  **Stability:**  If we regard the objects being clustered as a random subset of a population, we can ask whether the same cluster structure would have emerged in a different random subset.  We can measure this with bootstrapped subsampling.

    +  A cluster structure being stable doesn't necessarily mean it is meaningful.
    See e.g. [this tweet](https://twitter.com/AndrewZalesky/status/1363741266761027585)
    
    
3.  **Enrichment in known labels:** (See *semi-supervised learning* above)

4.  **Statistical significance/generative models:** If our clustering method  places model assumptions on how the data was generated, we can formulate a notion of statistically signficant clusters.  

    +  A good example of this is "model-based clustering", which typically assumes
    the data is generated from a mixture of multivariate Gaussians.  We can then
    estimate the probability that a particular object came from a particular 
    distribution in the mixture.
    
    
## Other details

#### Partitions versus Extractions

In many clustering algorithms, each object is placed into **exactly one** cluster. This is called a **partition**.

However, some algorithms allow for overlapping clusters, i.e. objects belonging to  multiple clusters - or for some objects to be "background" and have no cluster  membership at all.

(This is sometimes called **community detection** or **cluster extraction**)

#### What are the variables, what are the samples?

Consider the example of data consisting of many gene expression measurements for several individual subjects.

From a statistical perspective, we'd typically regard the subjects as samples from a population, and the gene expression levels as variables being studied.

When we cluster this data, sometimes we look for clusters of **subjects**, i.e., people whose gene signature is similar.  Sometimes we look for clusters of **genes**, i.e., genes that activate or deactivate together.

Non-statistical algorithms (e.g. kmeans) are agnostic to what set of objects we  cluster on.  In statistical algorithms, your model assumptions need to match your notion of samples/variables.

#### Beyond the similarity matrix

In the vast majority of cases, all you need for a clustering algorithm is a  *similarity matrix*; that is, entry $(i,j)$ of the matrix is the pairwise similarity  measure between object $i$ and object $j$.

Of course, the choice of similarity measure matters enormously.  But the point is you only need pairwise similarities to run e.g. kmeans.

Some approaches, though, need more information.  Statistical approaches in particular often estimate nuisance variables (like variance) from multiple measurements before collapsing the data into pairwise similarities.


# Inclusion in tidymodels

Here's an overview of how I see unsupervised learning fitting into a tidymodels framework.

## Recipes and Workflow

These steps can remain the same; there's no reason unsupervised methods need different preprocessing options/structure.

## `parsnip`-like models

Any and all clustering methods could be implemented as model spec functions, and the setup would look similar, e.g.


```r
km_mod <- k_means(centers = 5) %>%
    set_mode("partition") %>%
    set_engine("stats")
```


In my opinion, it makes sense to create a "fraternal twin" package to `parsnip`  that is essentially the same structure.  The reason I think this should be a  separate package is that there are two major philosophical shifts for supervised vs. unsupervised learning.

I'm going to refer to the twin package as `celery` for now because that name makes me laugh.

#### Modes

In `parsnip`, the possible modes are *classification*, *censored regression*,  or *regression*, and they correspond to the nature of the response variable.

In `celery`, there is no response variable.  I believe the modes should be  *partition* or *extraction*.  (see above for details)  The way that a model fit should output results is fundamentally different for those two tasks.

I don't really see a parallel between the classification/regression divide and the partition/extraction divide.  It doesn't feel right to just lump them together into four "modes" options.

#### Choosing a dissimilarity metric

Every model comes with choices about parameters and assumptions and such.  To some degree, this would work the same in `parsnip` and `celery`:


```r
knn_mod <- nearest_neighbors(neighbors = 5)

km_mod <- k_means(centers = 5)
```

However, for clustering, the decision of how to define "dissimilarity" feels like a more major decision.  That is, supervised methods require **one major choice** (what model) and then many minor choices (the parameters/penalties) for that model. Unsupervised methods require **two major choices**: the algorithm and the similarity measure.

I would like to see the dissimilarity measure be given "front-row billing", to

a. make it easy to swap between choices and compare outputs
b. make it obvious what we are looking for in our clusters
c. force the user to choose very deliberately rather than relying on defaults

For example,


```r
k_means(centers = 5) %>%
    set_dist("euclidean")

k_means(centers = 5) %>%
    set_dist("manhattan")
```

Much like with *modes*, this would require the `celery` package to pre-specify which disssimilarities were available for particular clustering algorithms.  For example, some methods require a true distance measure, while others are less strict.

These would have to be more flexible than modes, though, since clustering methods have **so** many possible dissimilarity measures.

## fitting to data

There is no real reason the `fit` step needs to look different, once a model is specified:


```r
km_mod <- k_means(centers = 5) %>%
    set_dist("euclidean") %>%
    set_mode("partition") %>%
    set_engine("stats")


km_mod %>%
    fit(vars(bill_length_mm:body_mass_g), data = penguins)
```

It's been suggested that "fit" might not be the right verb for unsupervised methods.  Technicallly, many supervised methods are not truly fitting a model either (see: K nearest neighbors), so I'm not worried about this.

## `rsample` and cross-validating

Cross-validation as such doesn't make sense for clustering, because there is no notion of prediction on the test set.

However, a similar framework could make sense for subsampling.  Instead of $v$ cross-validations, we'd have $v$ subsamples, and we'd look at certain success metrics for each subsample to get a sense of variability in the metric.


## `yardstick`-like metrics

Unsupervised methods have very different (and more ambiguous) metrics for "success" than supervised methods.  (see above for lots and lots of detail)

The underlying structure of the package doesn't need to change like it would for `parsnip`/`celery`.  We just need more functions implementing metrics that make sense for clustering.

# Meta:  What does this buy us?

Here are the advantages to a tidymodels implementation of clustering:

1. **Consistency and clarity:**  It's nice that all my modeling code looks similar even for different analyses with different models.  I'd like that to extend to unsupervised analyses.

2. **Easy framework for beginners:**  One problem tidymodels solves is that  beginners tend to shove their data into a model until they get an output without  error.  Tidymodels separates the important decisions in the data processing and the model specification and the validation.  This advantage would also exist in the unsupervised case.

3. **Comparison of methods:** Sometimes it's hard to decide between clustering algorithms.  There's less of a clear measure of success in unsupervised learning - but it would still be a huge help if it were easier to "just try a bunch of things and see what works".

4. **Wrappers to automated repetition:**  In the same way that tidymodels condenses the v-fold cross-validation process into a single function, it could condense subsampling into a single function.  Also, I haven't said much about tuning, but it'd be great to be able to do something like:


```r
k_means(clusters = tune())
```



