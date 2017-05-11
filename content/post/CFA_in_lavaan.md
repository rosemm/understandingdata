---
title: "CFA in lavaan"
author: "Rose Hartman"
date: "2017-03-22"
categories: [ "Tutorials"]
tags: [ "annotated_output", "CFA", "R", "lavaan", "SEM" ]
Draft: false
output: 
  md_document: 
    preserve_yaml: true
---

One of the most widely-used models is the confirmatory factor analysis
(CFA). It specifies how a set of observed variables are related to some
underlying latent factor or factors. In this post, I step through how to
run a CFA in R using the lavaan package, how to interpret your output,
and how to write up the results.

<!--more-->
Quick links to content in this tutorial:

[Running a CFA in lavaan](#model)

[Annotated output](#output)

[How to write up the results](#writeup)

[Recommended references](#refs)

### This tutorial assumes...

-   That you are comfortable with basic stats and regression concepts
    such as
    [residuals](https://www.khanacademy.org/math/ap-statistics/bivariate-data-ap/least-squares-regression/a/introduction-to-residuals),
    [correlation](http://rpsychologist.com/d3/correlation/),
    [variance](https://www.khanacademy.org/math/statistics-probability/displaying-describing-data/sample-standard-deviation/v/sample-variance),
    and [model
    coefficients](http://stats.idre.ucla.edu/spss/output/regression-analysis/).
-   That you already have a basic understanding of [what a CFA model
    is](http://davidakenny.net/cm/mfactor.htm) (and [how it differs
    from, for example, an EFA or
    PCA](http://jonathantemplin.com/files/multivariate/mv11icpsr/mv11icpsr_lecture12.pdf)).
    You should also know [important concepts for running basic SEM
    models](http://davidakenny.net/cm/basics.htm), such as [model
    identification](http://davidakenny.net/cm/identify.htm), and [SEM
    degrees of
    freedom](https://clavelresearch.wordpress.com/2014/05/03/disentangling-degrees-of-freedom-for-sem/).
-   That you're not [brand new to
    R](http://blogs.uoregon.edu/rclub/2014/09/29/welcome-to-the-wonderful-world-of-r/).
    If you are, the descriptions may still be useful to you, but you may
    run into problems replicating the analysis on your own computer or
    editing the code to suit your needs.

<h3 id="model">
Running a CFA in R
</h3>
There's lots of great software available for running CFA models, but R
is my favorite because it's completely free and open-source. This
tutorial uses [lavaan](http://lavaan.ugent.be), an excellent R package
for structural equation modeling. If you're familiar with mplus, the
lavaan syntax and output will probably look somewhat familiar to you ---
it's designed to be used in a similar way.

#### Set up

If you don't already have `lavaan` installed, you'll need to do that
first:

    install.packages("lavaan")

As of when this post was published, when you load `lavaan`, you'll get
an warning that it is in beta still; that just means it's still in
development. Check back at the [lavaan website](http://lavaan.ugent.be)
for updates periodically.

    library(lavaan)

    ## Warning: package 'lavaan' was built under R version 3.3.2

    ## This is lavaan 0.5-23.1097

    ## lavaan is BETA software! Please report any bugs.

We'll use the Holzinger and Swineford (1939) data set for this example,
which comes built-in when you install `lavaan`. To learn more about the
data set, you can pull up its help documentation in R.

    ?HolzingerSwineford1939

Here are the first six rows of the data:

    head(HolzingerSwineford1939)

    ##   id sex ageyr agemo  school grade       x1   x2    x3       x4   x5
    ## 1  1   1    13     1 Pasteur     7 3.333333 7.75 0.375 2.333333 5.75
    ## 2  2   2    13     7 Pasteur     7 5.333333 5.25 2.125 1.666667 3.00
    ## 3  3   2    13     1 Pasteur     7 4.500000 5.25 1.875 1.000000 1.75
    ## 4  4   1    13     2 Pasteur     7 5.333333 7.75 3.000 2.666667 4.50
    ## 5  5   2    12     2 Pasteur     7 4.833333 4.75 0.875 2.666667 4.00
    ## 6  6   2    14     1 Pasteur     7 5.333333 5.00 2.250 1.000000 3.00
    ##          x6       x7   x8       x9
    ## 1 1.2857143 3.391304 5.75 6.361111
    ## 2 1.2857143 3.782609 6.25 7.916667
    ## 3 0.4285714 3.260870 3.90 4.416667
    ## 4 2.4285714 3.000000 5.30 4.861111
    ## 5 2.5714286 3.695652 6.30 5.916667
    ## 6 0.8571429 4.347826 6.65 7.500000

#### Check assumptions

Note that because CFAs (and all SEM models) are based on the covariances
among variances, they are susceptible to the effects of violations to
the assumption of normality (especially skew and outliers), which can
strongly affect covariances. Before running your model, you should
examine your variables to check that there are no serious deviations
from normality. The MVN package provides a handy function for plotting
this (as well as lots of other useful tests and plots --- [check out the
MVN vingette for
examples](https://cran.r-project.org/web/packages/MVN/vignettes/MVN.pdf)).

    library(MVN)
    # pull out just the variables we're using (x1-x9)
    x_vars <- HolzingerSwineford1939[,paste("x", 1:9, sep="")]

    uniPlot(x_vars, type = "histogram")

![](https://s3.amazonaws.com/www.understandingdata.net/CFA_histograms.png)

Some of these variables are note quite normal (e.g. `x6` definitely has
some positive skew), but for the most part these look acceptable. If you
see drastic deviations from normality (such as skew from a ceiling or
floor effect), you'll need to either transform those variables before
continuing or drop them from your model. Including highly non-normal
variables in your CFA model can wreck havoc on the estimation.

#### Specifying a CFA model

For this example, we'll focus on the nine variables called `x1` through
`x9`.

To build a CFA model in `lavaan`, you'll save a string with the model
details. Each line is one latent factor, with its indicators following
the `=~` (read this symbol as "is measured by").

    HS.model <- ' visual  =~ x1 + x2 + x3
                  textual =~ x4 + x5 + x6
                  speed   =~ x7 + x8 + x9 '

In the code above, there are three latent factors referring to students'
mental ability: visual, textual, and speed. The latent factors
themselves are never directly measured (that's what it means for them to
be latent), but we're assuming the nine variables we did observe are
indicators of those latent factors: The visual latent factor is measured
by x1, x2 and x3. The textual latent factor is measured by x4, x5, and
x6. The speed patent factor is measured by x7, x8, and x9.

To estimate the model in `lavaan`, the easiest method is to use the
`cfa` function. It comes with sensible defaults for estimating CFA
models, including the assumption that you'll want to estimate
covariances among all of your latent factors (so we don't actually have
to write those covariances into the model above). You can run a basic
CFA here by just using `cfa(HS.model, data=HolzingerSwineford1939)`.
There are actually a couple options I recommend changing from the
defaults, though, so we'll go through those before running the model.

#### Options for estimating the CFA model

There are lots of options for controlling the way the model is
interpreted, estimated, and presented, many of which are not highlighted
in the documentation for either the `cfa` or `sem` functions. To see the
full list, read the help documentation for `lavOptions`:

    ?lavOptions

The default estimator for CFA models with continuous indicators is
maximum likelihood (ML), which is probably what you want. The default
treatment of missing data is listwise deletion, though, which is
probably not what you want. As long the estimator is ML, you can set the
missingness option to full information maximum likelihood (FIML) with
`missing="fiml"`. FIML will generally result in estimates [similar to
what you would get with multiple
imputation](https://www.iriseekhout.com/missing-data/missing-data-methods/full-information-maximum-likelihood/),
but with the added advantage that it's all done in one step instead of
needing to do imputation, analysis, and pooling of estimates in three
steps.

Latent factors aren't measured, so they don't naturally have any scale.
In order to come up with a unique solution, though, the estimator needs
to have some scale for them. One solution is to set each latent factor's
scale to the scale of its first indicator --- this is lavaan's default
behavior. Another option is to constrain the latent factors to have a
mean of 0 and a variance of 1 (i.e. to standardize them). Although both
approaches will give you equivalent results, I prefer the second option
because it forces the latent covariances to be correlations, which is
handy for interpretation. It also means you don't have to give up the
test of the loading of the first indicator for each factor. You can
control this behavior by setting `std.lv=TRUE` when you call `cfa()`.

    fit <- cfa(HS.model, data=HolzingerSwineford1939, 
               std.lv=TRUE,  
               missing="fiml")

Confirm the estimator that was used (should be ML), make sure the model
converged normally, and check basics like the number of observations
(should equal the number of rows in the data):

    fit

    ## lavaan (0.5-23.1097) converged normally after  45 iterations
    ## 
    ##   Number of observations                           301
    ## 
    ##   Number of missing patterns                         1
    ## 
    ##   Estimator                                         ML
    ##   Minimum Function Test Statistic               85.306
    ##   Degrees of freedom                                24
    ##   P-value (Chi-square)                           0.000

Note that `lavaann` model objects are [S4
objects](http://adv-r.had.co.nz/S4.html), which may be a little
different in structure from other R models you're used to working with.
Check out `str(fit)` to see.

<h3 id="output">
</h3>
### Annotated CFA output from lavaan

First, I'll just load the `knitr` package, so I can turn some of the
output into nicer looking tables.

    library(knitr)
    options(knitr.kable.NA = '') # this will hide missing values in the kable table

You can get most of the information you'll want about your model from
one summary command:

    summary(fit, fit.measures=TRUE, standardized=TRUE)

This produces a lot of output, so we'll look at it piece by piece, and
then use `parameterEstimates(fit)` to pull out parts of the `summary()`
output individually.

#### Fit Indices: Does your model fit your data?

You'll see quite a few fit indices, and you certainly don't need to
report all of them. There are plenty of [good resources that go into
much more detail about each of
these](http://davidakenny.net/cm/fit.htm), so I'll just point out a few
of the most useful and widely-used measures.

    > summary(fit, fit.measures=TRUE, standardized=TRUE)
    ...

    User model versus baseline model:

      Comparative Fit Index (CFI)                    0.931
      Tucker-Lewis Index (TLI)                       0.896

**CFI (Comparative fit index):** Measures whether the model fits the
data better than a more restricted baseline model. Higher is better,
with okay fit &gt; .9.

**TLI (Tucker-Lewis index):** Similar to CFI, but it penalizes overly
complex models (making it more conservative than CFI). Measures whether
the model fits the data better than a more restricted baseline model.
Higher is better, with okay fit &gt; .9.

    > summary(fit, fit.measures=TRUE, standardized=TRUE)
    ...

    Loglikelihood and Information Criteria:

      Loglikelihood user model (H0)              -3737.745
      Loglikelihood unrestricted model (H1)      -3695.092

      Number of free parameters                         21
      Akaike (AIC)                                7517.490
      Bayesian (BIC)                              7595.339
      Sample-size adjusted Bayesian (BIC)         7528.739

**AIC (Akaike’s information criterion):** Attempts to select models that
are the most parsimonious/efficient representations of the observed
data. Lower is better.

**BIC (Schwarz’s Bayesian information criterion):** Similar to AIC but a
little more conservative, also attempts to select models that are the
most parsimonious/efficient representations of the observed data. Lower
is better.

    > summary(fit, fit.measures=TRUE, standardized=TRUE)
    ...

    Root Mean Square Error of Approximation:

      RMSEA                                          0.092
      90 Percent Confidence Interval          0.071  0.114
      P-value RMSEA <= 0.05                          0.001

**RMSEA (Root mean square error of approximation):** The "error of
approximation" refers to residuals. Instead of comparing to a baseline
model, it measures how closely the model reproduces data patterns (i.e.
the covariances among indicators). Lower is better. It comes with a
90%CI in `lavaan` and other major SEM software, so that's often reported
along with it.

The p-value printed with it tests the hypothesis that RMSEA is less than
or equal to .05 (a cutoff sometimes used for "close" fit); here, our
RMSEA is greater than .05 (it's .092, with a 90%CI from .07 to .11), so
the p-value is unsurprisingly significant, telling us that RMSEA is NOT
less than or equal to .05. This p-value is sometimes called "the p of
Close Fit" or "PCLOSE" in other software. If it is *greater* than *α*
(usually set at .05), then it is typical to report that the model has
"close fit" according to the RMSEA.

#### Super important caveat

If your model fits well, that does NOT necessarily mean it is a "good"
model, or that it reflects truth or reality well. In CFA even more so
than other kinds of statistical modeling, the *theory* behind your model
is crucial to deciding whether or not a model is any good. If you start
playing around with SEM, you'll quickly realize that for a given set of
variables, there are often many different models that fit well, and they
may even seriously contradict each other and/or suggest nonsensical
relationships. **Good model fit does not make a good model. The model
needs to be solid theoretically before you estimate it.** If you're not
sure about the theory supporting your model, then CFA is not the right
tool for this stage of your research.

#### Parameter Estimates

Let's look at the complete list of all of the parameters in our model.

    parameterEstimates(fit, standardized=TRUE)

    ##        lhs op     rhs   est    se      z pvalue ci.lower ci.upper std.all
    ## 1   visual =~      x1 0.900 0.083 10.808      0    0.736    1.063   0.772
    ## 2   visual =~      x2 0.498 0.081  6.164      0    0.340    0.656   0.424
    ## 3   visual =~      x3 0.656 0.078  8.458      0    0.504    0.808   0.581
    ## 4  textual =~      x4 0.990 0.057 17.458      0    0.879    1.101   0.852
    ## 5  textual =~      x5 1.102 0.063 17.601      0    0.979    1.224   0.855
    ## 6  textual =~      x6 0.917 0.054 17.051      0    0.811    1.022   0.838
    ## 7    speed =~      x7 0.619 0.074  8.337      0    0.474    0.765   0.570
    ## 8    speed =~      x8 0.731 0.075  9.682      0    0.583    0.879   0.723
    ## 9    speed =~      x9 0.670 0.078  8.642      0    0.518    0.822   0.665
    ## 10      x1 ~~      x1 0.549 0.119  4.612      0    0.316    0.782   0.404
    ## 11      x2 ~~      x2 1.134 0.104 10.875      0    0.929    1.338   0.821
    ## 12      x3 ~~      x3 0.844 0.095  8.881      0    0.658    1.031   0.662
    ## 13      x4 ~~      x4 0.371 0.048  7.739      0    0.277    0.465   0.275
    ## 14      x5 ~~      x5 0.446 0.058  7.703      0    0.333    0.560   0.269
    ## 15      x6 ~~      x6 0.356 0.043  8.200      0    0.271    0.441   0.298
    ## 16      x7 ~~      x7 0.799 0.088  9.130      0    0.628    0.971   0.676
    ## 17      x8 ~~      x8 0.488 0.092  5.321      0    0.308    0.667   0.477
    ## 18      x9 ~~      x9 0.566 0.091  6.250      0    0.389    0.744   0.558
    ## 19  visual ~~  visual 1.000 0.000     NA     NA    1.000    1.000   1.000
    ## 20 textual ~~ textual 1.000 0.000     NA     NA    1.000    1.000   1.000
    ## 21   speed ~~   speed 1.000 0.000     NA     NA    1.000    1.000   1.000
    ## 22  visual ~~ textual 0.459 0.063  7.225      0    0.334    0.583   0.459
    ## 23  visual ~~   speed 0.471 0.086  5.457      0    0.302    0.640   0.471
    ## 24 textual ~~   speed 0.283 0.071  3.959      0    0.143    0.423   0.283
    ## 25      x1 ~1         4.936 0.067 73.473      0    4.804    5.067   4.235
    ## 26      x2 ~1         6.088 0.068 89.855      0    5.955    6.221   5.179
    ## 27      x3 ~1         2.250 0.065 34.579      0    2.123    2.378   1.993
    ## 28      x4 ~1         3.061 0.067 45.694      0    2.930    3.192   2.634
    ## 29      x5 ~1         4.341 0.074 58.452      0    4.195    4.486   3.369
    ## 30      x6 ~1         2.186 0.063 34.667      0    2.062    2.309   1.998
    ## 31      x7 ~1         4.186 0.063 66.766      0    4.063    4.309   3.848
    ## 32      x8 ~1         5.527 0.058 94.854      0    5.413    5.641   5.467
    ## 33      x9 ~1         5.374 0.058 92.546      0    5.260    5.488   5.334
    ## 34  visual ~1         0.000 0.000     NA     NA    0.000    0.000   0.000
    ## 35 textual ~1         0.000 0.000     NA     NA    0.000    0.000   0.000
    ## 36   speed ~1         0.000 0.000     NA     NA    0.000    0.000   0.000

Take a look at the `op` column. These are the same symbols you use to
specify relationships when writing your model code. The first set of
parameters you'll see are all `=~` operators ("is measured by"), giving
you the loading for each indicator on its factor. For example, the
factor loading for `x1` on the visual factor is .90.

Factor loadings can be interpreted like a regression coefficient. For
example, the first parameter says that for each unit increase in the
latent visual ability (since we standardized latent factors, this means
for each 1SD increase), the model predicts a .90-unit increase in x1.
Because we included `standardized=TRUE` in the command, standardized
parameter estimates as well; the column called "std.all" is the one
people typically think of as "standardized regression coefficients"
(often reported as *β* in regression software). The full output actually
includes two additional columns of standardized coefficients, but I
omitted them here to save room and because they're rarely reported.

The next kind of operator in the table is `~~`. This means covariance,
or, when it's between a variable and itself, variance. The variances of
observed variables (`x1` through `x9` in our case) are the error
variances. All of our indicators' error variances are significantly
greater than 0, suggesting that the latent factors don't perfectly
predict the observed variable scores. That's typical.

When you get down to the latent variable variances (e.g.
`visual ~~ visual`), you'll see they're all exactly 1 and there are no
standard errors or significance tests provided. That's because we
standardized the latent variables when we ran the model, constraining
them so the variance would equal exactly 1.

The next operator is `~1`, which is the intercept for each variable.
This works like an intercept in regular regression models --- it is the
expected value for that variable when all of its predictors are at 0. In
this case, each indicator has only one predictor (its latent factor),
and the latent factors are all standardized so that their means are at
0; so it works out that this is just literally telling us the mean of
each variable here. In a more complex model, the intercepts won't always
equal the variable means. And just as with the variances, you'll see
that the means of the latent variables are all exactly the same, with no
standard errors. That's because the latent variables were standardized
when we ran the model, so they all have a mean of 0 and variance of 1 by
definition.

Note that the parameter estimates table is an R data frame, so you can
use it in other R code easily. For example, you can pull out all of the
factor loadings and put them in their own table with some `dplyr`
functions and `kable` from the `knitr` package.

    library(dplyr) 
    library(tidyr)
    parameterEstimates(fit, standardized=TRUE) %>% 
      filter(op == "=~") %>% 
      select('Latent Factor'=lhs, Indicator=rhs, B=est, SE=se, Z=z, 'p-value'=pvalue, Beta=std.all) %>% 
      kable(digits = 3, format="pandoc", caption="Factor Loadings")

<table>
<caption>Factor Loadings</caption>
<thead>
<tr class="header">
<th align="left">Latent Factor</th>
<th align="left">Indicator</th>
<th align="right">B</th>
<th align="right">SE</th>
<th align="right">Z</th>
<th align="right">p-value</th>
<th align="right">Beta</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">visual</td>
<td align="left">x1</td>
<td align="right">0.900</td>
<td align="right">0.083</td>
<td align="right">10.808</td>
<td align="right">0</td>
<td align="right">0.772</td>
</tr>
<tr class="even">
<td align="left">visual</td>
<td align="left">x2</td>
<td align="right">0.498</td>
<td align="right">0.081</td>
<td align="right">6.164</td>
<td align="right">0</td>
<td align="right">0.424</td>
</tr>
<tr class="odd">
<td align="left">visual</td>
<td align="left">x3</td>
<td align="right">0.656</td>
<td align="right">0.078</td>
<td align="right">8.458</td>
<td align="right">0</td>
<td align="right">0.581</td>
</tr>
<tr class="even">
<td align="left">textual</td>
<td align="left">x4</td>
<td align="right">0.990</td>
<td align="right">0.057</td>
<td align="right">17.458</td>
<td align="right">0</td>
<td align="right">0.852</td>
</tr>
<tr class="odd">
<td align="left">textual</td>
<td align="left">x5</td>
<td align="right">1.102</td>
<td align="right">0.063</td>
<td align="right">17.601</td>
<td align="right">0</td>
<td align="right">0.855</td>
</tr>
<tr class="even">
<td align="left">textual</td>
<td align="left">x6</td>
<td align="right">0.917</td>
<td align="right">0.054</td>
<td align="right">17.051</td>
<td align="right">0</td>
<td align="right">0.838</td>
</tr>
<tr class="odd">
<td align="left">speed</td>
<td align="left">x7</td>
<td align="right">0.619</td>
<td align="right">0.074</td>
<td align="right">8.337</td>
<td align="right">0</td>
<td align="right">0.570</td>
</tr>
<tr class="even">
<td align="left">speed</td>
<td align="left">x8</td>
<td align="right">0.731</td>
<td align="right">0.075</td>
<td align="right">9.682</td>
<td align="right">0</td>
<td align="right">0.723</td>
</tr>
<tr class="odd">
<td align="left">speed</td>
<td align="left">x9</td>
<td align="right">0.670</td>
<td align="right">0.078</td>
<td align="right">8.642</td>
<td align="right">0</td>
<td align="right">0.665</td>
</tr>
</tbody>
</table>

Neat, huh? :)

#### Residuals and Modification Indices

The goal of the CFA is to explain relationships among the observed
variables by specifying a latent structure connecting them.

For example, in our model, we're saying that `x1`, `x2`, and `x3` are
all correlated because they're different ways to measure the same basic
underlying ability, visual ability. And although `x1` and `x4` measure
different abilities (visual and textual ability, respectively), we would
still expect them to have some correlation because individuals latent
visual and textual ability are correlated.

Because our model implies expected relationships among the observed
variables, one way to examine its performance is to look at the
difference between the correlation matrix the model expects and the
actual, observed correlation matrix you get from your raw data. These
(or the equivalent based on covariance matrices) are the residuals of an
SEM model. Any large residual correlations between variables suggests
that there's something about the relationship between those two
indicators that the model is not adequately capturing.

    residuals(fit, type = "cor")$cor

To make the correlation matrix a little easier to read, I'll wrap it in
a `kable()` command from the `knitr` package to print a nice table:

    cor_table <- residuals(fit, type = "cor")$cor

    cor_table[upper.tri(cor_table)] <- NA # erase the upper triangle
    diag(cor_table) <- NA # erase the diagonal 0's

    kable(cor_table, digits=2) # makes a nice table and rounds everyhing to 2 digits

<table>
<thead>
<tr class="header">
<th></th>
<th align="right">x1</th>
<th align="right">x2</th>
<th align="right">x3</th>
<th align="right">x4</th>
<th align="right">x5</th>
<th align="right">x6</th>
<th align="right">x7</th>
<th align="right">x8</th>
<th align="right">x9</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>x1</td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
</tr>
<tr class="even">
<td>x2</td>
<td align="right">-0.03</td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
</tr>
<tr class="odd">
<td>x3</td>
<td align="right">-0.01</td>
<td align="right">0.09</td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
</tr>
<tr class="even">
<td>x4</td>
<td align="right">0.07</td>
<td align="right">-0.01</td>
<td align="right">-0.07</td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
</tr>
<tr class="odd">
<td>x5</td>
<td align="right">-0.01</td>
<td align="right">-0.03</td>
<td align="right">-0.15</td>
<td align="right">0.01</td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
</tr>
<tr class="even">
<td>x6</td>
<td align="right">0.06</td>
<td align="right">0.03</td>
<td align="right">-0.03</td>
<td align="right">-0.01</td>
<td align="right">0.00</td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
</tr>
<tr class="odd">
<td>x7</td>
<td align="right">-0.14</td>
<td align="right">-0.19</td>
<td align="right">-0.08</td>
<td align="right">0.04</td>
<td align="right">-0.04</td>
<td align="right">-0.01</td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
</tr>
<tr class="even">
<td>x8</td>
<td align="right">-0.04</td>
<td align="right">-0.05</td>
<td align="right">-0.01</td>
<td align="right">-0.07</td>
<td align="right">-0.04</td>
<td align="right">-0.02</td>
<td align="right">0.07</td>
<td align="right"></td>
<td align="right"></td>
</tr>
<tr class="odd">
<td>x9</td>
<td align="right">0.15</td>
<td align="right">0.07</td>
<td align="right">0.15</td>
<td align="right">0.05</td>
<td align="right">0.07</td>
<td align="right">0.06</td>
<td align="right">-0.04</td>
<td align="right">-0.03</td>
<td align="right"></td>
</tr>
</tbody>
</table>

Keep an eye out for residual correlations larger than about .1. These
residuals mostly look really good, with a few possible exceptions: `x1`
with `x7` and `x9`; `x2` with `x7`; `x3` with `x5` and `x9`. The RMSEA
discussed above is based on these residual correlations, so the
deviations we're seeing here are what's driving the RMSEA value we saw
above.

Note that if you have categorical indicator variables, you'll want to
look at expected vs. observed counts in each level instead of residual
correlations. You can get that with `lavTables(fit)`.

Modification indices tell you how model fit would change if you added
new parameters to the model. Since your CFA model should not be
exploratory (i.e. you should know what parameters you want to include in
the model before you begin), modification indices can be dangerous. If
you make the changes they suggest, you run a serious risk of
over-fitting your data and reducing the generalizability of your
results.

Instead, I recommend using modification indices mostly as another
description of the places where your model is not fitting well, like
examining the residuals. In the code below, I've sorted the modification
indices by `mi` which is an estimate of how much the model fit would
improve if each parameter were added. You can see from the output below
that the top modification indices are all for variables `x7`, `x8`, and
`x9`, suggesting that those variables are involved in some covariances
that aren't well captured by the current model structure. In particular,
the top modification index is for a factor loading from `visual` to
`x9`; this wouldn't actually make sense theoretically, but it's useful
to know that there is some extra covariance between `x9` and the
variables that measure visual ability. The next modification index is
proposing a covariance between `x7` and `x8`. They already share a
latent factor, so this is reflecting an additional relationship above
their loadings on the `speed` factor --- this could happen if `x7` and
`x8` are more tightly correlated with each other than either is with
`x9`. Taken together, this all suggests to me that `x9` is not quite
adhering to the expected pattern from the model. Since the overall model
fit is good and the residual correlations aren't very extreme, I'm not
too concerned (but it's still nice to know).

    modificationIndices(fit, sort.=TRUE, minimum.value=3)

    ##        lhs op rhs     mi    epc sepc.lv sepc.all sepc.nox
    ## 42  visual =~  x9 36.411  0.519   0.519    0.515    0.515
    ## 88      x7 ~~  x8 34.145  0.536   0.536    0.488    0.488
    ## 40  visual =~  x7 18.631 -0.380  -0.380   -0.349   -0.349
    ## 90      x8 ~~  x9 14.946 -0.423  -0.423   -0.415   -0.415
    ## 45 textual =~  x3  9.151 -0.269  -0.269   -0.238   -0.238
    ## 67      x2 ~~  x7  8.918 -0.183  -0.183   -0.143   -0.143
    ## 43 textual =~  x1  8.903  0.347   0.347    0.297    0.297
    ## 63      x2 ~~  x3  8.532  0.218   0.218    0.164    0.164
    ## 71      x3 ~~  x5  7.858 -0.130  -0.130   -0.089   -0.089
    ## 38  visual =~  x5  7.441 -0.189  -0.189   -0.147   -0.147
    ## 62      x1 ~~  x9  7.335  0.138   0.138    0.117    0.117
    ## 77      x4 ~~  x6  6.221 -0.235  -0.235   -0.185   -0.185
    ## 78      x4 ~~  x7  5.920  0.098   0.098    0.078    0.078
    ## 60      x1 ~~  x7  5.420 -0.129  -0.129   -0.102   -0.102
    ## 89      x7 ~~  x9  5.183 -0.187  -0.187   -0.170   -0.170
    ## 48 textual =~  x9  4.796  0.137   0.137    0.136    0.136
    ## 41  visual =~  x8  4.295 -0.189  -0.189   -0.187   -0.187
    ## 75      x3 ~~  x9  4.126  0.102   0.102    0.089    0.089
    ## 79      x4 ~~  x8  3.805 -0.069  -0.069   -0.059   -0.059
    ## 55      x1 ~~  x2  3.606 -0.184  -0.184   -0.134   -0.134
    ## 57      x1 ~~  x4  3.554  0.078   0.078    0.058    0.058
    ## 47 textual =~  x8  3.359 -0.120  -0.120   -0.118   -0.118

### Model Comparison

Sometimes you have two competing theories to test on the same variables.
In that situation, running more than one CFA and testing the fit of the
two models against each other can be a valuable part of your analysis.

#### Example 1: Compare to model without covariances

For example, let's say that the three-factor structure of ability as
modeled here is relatively new, and each of these abilities (visual,
textual, and speed) have typically been studied independently in the
past. As part of your analysis, you may want to test a model that
includes covariances among the three latent factors vs. one that treats
them as independent.

When comparing models, a very important consideration is whether or not
the models are
[nested](http://stats.stackexchange.com/questions/27560/what-is-the-difference-between-a-nested-and-non-nested-model-in-cfa)
--- regular model comparison techniques only work on nested models. In
this case, the model without the covariances is nested within the more
complex model.

We've already run the model that allows covariances among the latent
factors. To run the reduced model with no covariances, we could re-write
the model code, or in this case there's actually a handy shortcut we can
use in the `cfa()` command. For details, see `?lavOptions`.

    fit_orth <- cfa(HS.model, data=HolzingerSwineford1939, 
               std.lv=TRUE,  
               missing="fiml",
               orthogonal = TRUE)
    fit_orth

    ## lavaan (0.5-23.1097) converged normally after  26 iterations
    ## 
    ##   Number of observations                           301
    ## 
    ##   Number of missing patterns                         1
    ## 
    ##   Estimator                                         ML
    ##   Minimum Function Test Statistic              153.527
    ##   Degrees of freedom                                27
    ##   P-value (Chi-square)                           0.000

Note that we have three more df in this model (27, compared to 24 for
the other model); that's because we didn't estimate the three
covariances among the latent factors.

I won't go through all of the fit indices and parameter estimates for
`fit_orth`, because our main interest here is just whether the more
complex model (allowing covariances among the latent factors) is a
significantly better fit than this one, despite the fact that it has to
estimate more parameters. We can test that using the `anova()` function,
which is a general R function for comparing lots of kinds of nested
models, not just `lavaan` models.

    anova(fit, fit_orth)

    ## Chi Square Difference Test
    ## 
    ##          Df    AIC    BIC   Chisq Chisq diff Df diff Pr(>Chisq)    
    ## fit      24 7535.5 7646.7  85.305                                  
    ## fit_orth 27 7597.7 7697.8 153.527     68.222       3  1.026e-14 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

The model allowing covariances among the three latent ability factors
fits the data significantly better than a model treating the latent
factors as independent, *χ*<sup>2</sup>(3)=68.22, *p*&lt;.001.

#### Example 2: Compare to model with just one latent factor

Another common example of nested model comparisons is testing a CFA with
multiple latent factors against a CFA on the same indicators with just
one latent factor. In this case, the models are nested because the
reduced model (with just one latent factor) is the same as the full
model but with the correlations among latent factors set to exactly 1,
making all of the latent factors identical. Because these models are
nested, we can test them against each other directly.

In order to specify the CFA with just one latent factor, I'll re-write
the model code.

    HS.model.one <- ' ability  =~ x1 + x2 + x3 + x4 + x5 + x6 + x7 + x8 + x9 '

    fit_one <- cfa(HS.model.one, data=HolzingerSwineford1939, 
               std.lv=TRUE,  
               missing="fiml")
    fit_one

    ## lavaan (0.5-23.1097) converged normally after  41 iterations
    ## 
    ##   Number of observations                           301
    ## 
    ##   Number of missing patterns                         1
    ## 
    ##   Estimator                                         ML
    ##   Minimum Function Test Statistic              312.264
    ##   Degrees of freedom                                27
    ##   P-value (Chi-square)                           0.000

Again, you can see that we have more df in this model compared to the
full model, because it's estimating fewer parameters.

    anova(fit, fit_one)

    ## Chi Square Difference Test
    ## 
    ##         Df    AIC    BIC   Chisq Chisq diff Df diff Pr(>Chisq)    
    ## fit     24 7535.5 7646.7  85.305                                  
    ## fit_one 27 7756.4 7856.5 312.264     226.96       3  < 2.2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

The model with the three latent ability factors fits the data
significantly better than a model with only a single latent factor for
general ability, *χ*<sup>2</sup>(3)=226.96, *p*&lt;.001.

<h3 id="writeup">
Writing up the results
</h3>
There is no single agreed-upon way to write up a CFA (or any SEM model),
and practices vary by discipline and researcher. That said, there are
some standard pieces of information to include, so you don't have to
build it all from scratch. Here is a (slightly edited) list of things to
include in your write up from [Beaujean
(2014)](https://www.scribd.com/document/238478414/Beaujean-Latent-Variable-Modeling-Using-r):

> 1.  A theoretical and empirical justification for the hypothesized
>     model
> 2.  A complete description of how the model was specified (i.e., the
>     indicator variables for each latent variable, the scaling of the
>     latent variables, a description of what parameters were estimated
>     and constrained)
> 3.  A description of sample (i.e., demographic information, sample
>     size, sampling method)
> 4.  A description of the type of data used (e.g., nominal, continuous)
>     and descriptive statistics
> 5.  Tests of assumptions (specifically that the indicator variables
>     follow a multivariate normal distribution) and estimator used
> 6.  A description of missing data and how the missing data was handled
> 7.  The software and version used to fit the model
> 8.  Measures, and the criteria used, to judge model fit
> 9.  Any alterations made to the original model based on model fit or
>     modification indices
> 10. All parameter estimates (i.e., loadings, error variances,
>     latent (co)variances) and their standard errors, probably in a
>     table

#### Example write-up

I used a confirmatory factor analysis to test a three factor model of
students' mental ability, using data collected by Holzinger and
Swineford (1939). The data included scores on a variety of ability tests
from 301 seventh- and eighth-grade students in two different schools.
For the purposes of the current study, the school variable was ignored
and all students treated as one group. Prior research suggests that
mental ability can be meaningfully separated into at least three
distinct factors: visual ability, textual ability, and mental speed (Pen
& Teller, 1999; Crowd et al., 2007). Crowd and colleagues (2007) showed
that tests of mental ability relying primarily on visual ability can be
clearly differentiated from tests that rely primarily on textual ability
or mental speed, with patients who have experienced brain lesions
showing selective impairment depending on the location of the lesions.
Moreover, exploratory factor analyses on similar sets of ability tests
have suggested that a three factor solution provides a good fit for both
children (Extant, 2001) and college students (Extant & Student, 2003,
2005), underscoring the plausibility of a similar factor structure in
middle schoolers.

The data for the current study included nine different tests of mental
ability, three for each ability factor. To measure visual ability, I
used x1, x2, and x3. To measure textual ability, I used x4, x5, and x6.
Mental speed was measured by x7, x8, and x9. All nine tests are scored
on a scale from 0 (worst possible performance) to 10 (best possible
performance) and are treated as continuous variables in the analysis.
Exploratory data analysis revealed only minor deviations from normality
in their distributions (see Appendix A). Descriptives for all observed
variables are provided in Table 1.

    # erasing some data, so we have missingness to report
    HolzingerSwineford1939[sample(1:301, 10),sample(7:15, 3)] <- NA
    HolzingerSwineford1939[sample(1:301, 10),sample(7:15, 3)] <- NA
    HolzingerSwineford1939[sample(1:301, 10),sample(7:15, 3)] <- NA
    HolzingerSwineford1939[sample(1:301, 10),sample(7:15, 3)] <- NA

    # generate the descriptives table using dplyr and tidyr functions, and kable
    HolzingerSwineford1939 %>% 
      select(x1:x9) %>% 
      gather("Variable", "value") %>% 
      group_by(Variable) %>% 
      summarise(Mean=mean(value, na.rm=TRUE), 
                SD=sd(value, na.rm=TRUE), 
                min=min(value, na.rm=TRUE), 
                max=max(value, na.rm=TRUE), 
                '% Missing'=100*length(which(is.na(value)))/n()) %>% 
      kable(digits=2, format="pandoc", caption="Table 1: Descriptive Statistics for Observed Variables")

<table>
<caption>Table 1: Descriptive Statistics for Observed Variables</caption>
<thead>
<tr class="header">
<th align="left">Variable</th>
<th align="right">Mean</th>
<th align="right">SD</th>
<th align="right">min</th>
<th align="right">max</th>
<th align="right">% Missing</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">x1</td>
<td align="right">4.91</td>
<td align="right">1.16</td>
<td align="right">0.67</td>
<td align="right">8.50</td>
<td align="right">6.64</td>
</tr>
<tr class="even">
<td align="left">x2</td>
<td align="right">6.08</td>
<td align="right">1.19</td>
<td align="right">2.25</td>
<td align="right">9.25</td>
<td align="right">3.32</td>
</tr>
<tr class="odd">
<td align="left">x3</td>
<td align="right">2.25</td>
<td align="right">1.13</td>
<td align="right">0.25</td>
<td align="right">4.50</td>
<td align="right">0.00</td>
</tr>
<tr class="even">
<td align="left">x4</td>
<td align="right">3.06</td>
<td align="right">1.16</td>
<td align="right">0.00</td>
<td align="right">6.33</td>
<td align="right">0.00</td>
</tr>
<tr class="odd">
<td align="left">x5</td>
<td align="right">4.31</td>
<td align="right">1.28</td>
<td align="right">1.00</td>
<td align="right">6.75</td>
<td align="right">6.64</td>
</tr>
<tr class="even">
<td align="left">x6</td>
<td align="right">2.17</td>
<td align="right">1.11</td>
<td align="right">0.14</td>
<td align="right">6.14</td>
<td align="right">3.32</td>
</tr>
<tr class="odd">
<td align="left">x7</td>
<td align="right">4.20</td>
<td align="right">1.09</td>
<td align="right">1.30</td>
<td align="right">7.43</td>
<td align="right">6.31</td>
</tr>
<tr class="even">
<td align="left">x8</td>
<td align="right">5.53</td>
<td align="right">1.01</td>
<td align="right">3.05</td>
<td align="right">10.00</td>
<td align="right">6.64</td>
</tr>
<tr class="odd">
<td align="left">x9</td>
<td align="right">5.37</td>
<td align="right">1.02</td>
<td align="right">2.78</td>
<td align="right">9.25</td>
<td align="right">6.64</td>
</tr>
</tbody>
</table>

I fit the model using lavaan version 0.5-23 (Rosseel, 2012) in R version
3.3.1 (R Core Team, 2016). I used maximum likelihood estimation, with
full information maximum likelihood (FIML) for the missing data. I
standardized the latent factors, allowing free estimation of all factor
loadings. See Figure 1 for a diagram of the model tested. All R code for
the analysis is available in the Supplemental Materials.

![Alt
text](https://s3.amazonaws.com/www.understandingdata.net/CFA_model.jpg "Figure 1: CFA Model Diagram")

The model fit was acceptable but not excellent, with a TLI of .92 and
RMSEA of .074 90%CI(.052, .096). The full three factor model did fit the
data significantly better than a single-factor solution
(*χ*<sup>2</sup>(3)=226.96, *p*&lt;.001), or a three-factor solution
that did not allow covariances among the three latent factors
(*χ*<sup>2</sup>(3)=68.22, *p*&lt;.001). As expected, the indicators all
showed significant positive factor loadings, with standardized
coefficients ranging from .446 to .862 (see Table 2). There were also
significant positive correlations among all three latent factors (see
Table 3), indicating that students who showed high ability in one
dimension were more likely to show high ability in the others as well.
Taken together, these results are consistent with the characterization
of mental ability as comprising distinct factors for visual ability,
textual ability, and mental speed, as has been proposed in the
literature (Pen & Teller, 1999; Crowd et al. 2007).

    parameterEstimates(fit, standardized=TRUE) %>% 
      filter(op == "=~") %>% 
      mutate(stars = ifelse(pvalue < .001, "***", 
                            ifelse(pvalue < .01, "**", 
                                   ifelse(pvalue < .05, "*", "")))) %>%
      select('Latent Factor'=lhs, 
             Indicator=rhs, 
             B=est, 
             SE=se, Z=z, 
             Beta=std.all, 
             sig=stars) %>% 
      kable(digits = 3, format="pandoc", caption="Table 2: Factor Loadings")

<table>
<caption>Table 2: Factor Loadings</caption>
<thead>
<tr class="header">
<th align="left">Latent Factor</th>
<th align="left">Indicator</th>
<th align="right">B</th>
<th align="right">SE</th>
<th align="right">Z</th>
<th align="right">Beta</th>
<th align="left">sig</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">visual</td>
<td align="left">x1</td>
<td align="right">0.900</td>
<td align="right">0.083</td>
<td align="right">10.808</td>
<td align="right">0.772</td>
<td align="left">***</td>
</tr>
<tr class="even">
<td align="left">visual</td>
<td align="left">x2</td>
<td align="right">0.498</td>
<td align="right">0.081</td>
<td align="right">6.164</td>
<td align="right">0.424</td>
<td align="left">***</td>
</tr>
<tr class="odd">
<td align="left">visual</td>
<td align="left">x3</td>
<td align="right">0.656</td>
<td align="right">0.078</td>
<td align="right">8.458</td>
<td align="right">0.581</td>
<td align="left">***</td>
</tr>
<tr class="even">
<td align="left">textual</td>
<td align="left">x4</td>
<td align="right">0.990</td>
<td align="right">0.057</td>
<td align="right">17.458</td>
<td align="right">0.852</td>
<td align="left">***</td>
</tr>
<tr class="odd">
<td align="left">textual</td>
<td align="left">x5</td>
<td align="right">1.102</td>
<td align="right">0.063</td>
<td align="right">17.601</td>
<td align="right">0.855</td>
<td align="left">***</td>
</tr>
<tr class="even">
<td align="left">textual</td>
<td align="left">x6</td>
<td align="right">0.917</td>
<td align="right">0.054</td>
<td align="right">17.051</td>
<td align="right">0.838</td>
<td align="left">***</td>
</tr>
<tr class="odd">
<td align="left">speed</td>
<td align="left">x7</td>
<td align="right">0.619</td>
<td align="right">0.074</td>
<td align="right">8.337</td>
<td align="right">0.570</td>
<td align="left">***</td>
</tr>
<tr class="even">
<td align="left">speed</td>
<td align="left">x8</td>
<td align="right">0.731</td>
<td align="right">0.075</td>
<td align="right">9.682</td>
<td align="right">0.723</td>
<td align="left">***</td>
</tr>
<tr class="odd">
<td align="left">speed</td>
<td align="left">x9</td>
<td align="right">0.670</td>
<td align="right">0.078</td>
<td align="right">8.642</td>
<td align="right">0.665</td>
<td align="left">***</td>
</tr>
</tbody>
</table>

    parameterEstimates(fit, standardized=TRUE) %>% 
      filter(op == "~~", 
             lhs %in% c("visual", "textual", "speed"), 
             !is.na(pvalue)) %>% 
      mutate(stars = ifelse(pvalue < .001, "***", 
                            ifelse(pvalue < .01, "**", 
                                   ifelse(pvalue < .05, "*", "")))) %>% 
      select('Factor 1'=lhs, 
             'Factor 2'=rhs, 
             Correlation=est, 
             sig=stars) %>% 
      kable(digits = 3, format="pandoc", caption="Table 3: Latent Factor Correlations")

<table>
<caption>Table 3: Latent Factor Correlations</caption>
<thead>
<tr class="header">
<th align="left">Factor 1</th>
<th align="left">Factor 2</th>
<th align="right">Correlation</th>
<th align="left">sig</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="left">visual</td>
<td align="left">textual</td>
<td align="right">0.459</td>
<td align="left">***</td>
</tr>
<tr class="even">
<td align="left">visual</td>
<td align="left">speed</td>
<td align="right">0.471</td>
<td align="left">***</td>
</tr>
<tr class="odd">
<td align="left">textual</td>
<td align="left">speed</td>
<td align="right">0.283</td>
<td align="left">***</td>
</tr>
</tbody>
</table>

Notes:

-   In a real write up, you would refer to your variables by
    meaningful/informative names instead of x1, etc.
-   If there were more serious deviations from normality in the
    variables, you would want to present those results in the text (not
    off in an appendix or supplemental materials) and discuss what
    remedies if any you applied.
-   There actually aren't any missing data at all in this dataset, but
    I'm pretending there are some so I can show how to talk about it.
-   In a real write up, you would want to spend a little more time on
    the theoretical justification of your model (and probably include
    real cites instead of made up ones). Depending on your study, this
    might be its own paragraph, echoing the important points from the
    lit review in the introduction of your paper and presenting a few
    related models that have been tested in the literature.
-   I reported TLI and RMSEA since those are the two fit indices I see
    most commonly in my field. You should report those one or whichever
    ones are most common in your field. One thing to be wary of is that
    if you cherry-pick which fit measures to report based on which ones
    make your model look the best you will your bias your results and
    inflate your type-1 error rate. So while there's no hard rules about
    which stats to report, you do need to make sure you're not making
    the decision based on the results you see.
-   It's a great idea to provide the analysis code in an appendix or
    supplemental materials (such as a github repo), but you still do
    need to provide the relevant details of the analysis in your
    write up.
-   It's important to cite the software you use, and R makes it easy to
    get the citations. Many packages have a built-in citation, which you
    can see with the `citation()` command:

<!-- -->

    citation("lavaan") # here's the citation for your current version of lavaan

    ## 
    ## To cite lavaan in publications use:
    ## 
    ##   Yves Rosseel (2012). lavaan: An R Package for Structural
    ##   Equation Modeling. Journal of Statistical Software, 48(2), 1-36.
    ##   URL http://www.jstatsoft.org/v48/i02/.
    ## 
    ## A BibTeX entry for LaTeX users is
    ## 
    ##   @Article{,
    ##     title = {{lavaan}: An {R} Package for Structural Equation Modeling},
    ##     author = {Yves Rosseel},
    ##     journal = {Journal of Statistical Software},
    ##     year = {2012},
    ##     volume = {48},
    ##     number = {2},
    ##     pages = {1--36},
    ##     url = {http://www.jstatsoft.org/v48/i02/},
    ##   }

    citation() # here's the general citation for your current version of R

    ## 
    ## To cite R in publications use:
    ## 
    ##   R Core Team (2016). R: A language and environment for
    ##   statistical computing. R Foundation for Statistical Computing,
    ##   Vienna, Austria. URL https://www.R-project.org/.
    ## 
    ## A BibTeX entry for LaTeX users is
    ## 
    ##   @Manual{,
    ##     title = {R: A Language and Environment for Statistical Computing},
    ##     author = {{R Core Team}},
    ##     organization = {R Foundation for Statistical Computing},
    ##     address = {Vienna, Austria},
    ##     year = {2016},
    ##     url = {https://www.R-project.org/},
    ##   }
    ## 
    ## We have invested a lot of time and effort in creating R, please
    ## cite it when using it for data analysis. See also
    ## 'citation("pkgname")' for citing R packages.

    sessionInfo() # gives you the version numbers for R and any loaded packages

    ## R version 3.3.1 (2016-06-21)
    ## Platform: x86_64-apple-darwin13.4.0 (64-bit)
    ## Running under: OS X 10.10.5 (Yosemite)
    ## 
    ## locale:
    ## [1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8
    ## 
    ## attached base packages:
    ## [1] stats     graphics  grDevices utils     datasets  methods   base     
    ## 
    ## other attached packages:
    ## [1] tidyr_0.6.0        dplyr_0.5.0        knitr_1.15.1      
    ## [4] lavaan_0.5-23.1097
    ## 
    ## loaded via a namespace (and not attached):
    ##  [1] Rcpp_0.12.8     quadprog_1.5-5  assertthat_0.1  digest_0.6.10  
    ##  [5] rprojroot_1.1   R6_2.2.0        DBI_0.5-1       backports_1.0.4
    ##  [9] stats4_3.3.1    magrittr_1.5    evaluate_0.10   highr_0.6      
    ## [13] stringi_1.1.2   lazyeval_0.2.0  pbivnorm_0.6.0  rmarkdown_1.2  
    ## [17] tools_3.3.1     stringr_1.1.0   yaml_2.1.14     mnormt_1.5-5   
    ## [21] htmltools_0.3.5 tibble_1.2

### Troubleshooting

There are a number of error messages you may see when estimating
`lavaan` models. Here are a few to watch out for:

    Warning messages:
    1: In lav_object_post_check(lavobject) :
      lavaan WARNING: some estimated variances are negative
    2: In lav_object_post_check(lavobject) :
      lavaan WARNING: covariance matrix of latent variables is not positive definite; use inspect(fit,"cov.lv") to investigate.

You may see one or both of these messages if the model is struggling to
come up with reasonable estimates. Generally, the problem is
insufficiently informative data --- your N is too small, you have too
much missingness, and/or the covariances among your indicators are too
weak, leaving you with unstable latent factors.

Check for missing data. If you find you have substantial missingness,
note the pattern --- is it concentrated in a particular indicator or set
of indicators? Is it concentrated in a handful of participants? If so,
you may want to consider dropping the problematic variables or
participants from your analysis. You can also consider other missingness
remedies such as imputation, but note that if you followed the
instructions above `lavaan` is already using FIML to estimate around the
missing data (which works similarly to multiple imputation).

If you don't have much missingness, then the problem is likely due to
insufficient N and/or weak covariances among indicators, neither of
which you can fix easily. You may need to reduce the complexity of your
model (for example, by dropping indicators and/or latent factors) or
collect more data. As a rule of thumb, you should have at least 10-20
observations for each free parameter in your model (sometimes referred
to as [the N:q
rule](https://www.reddit.com/r/AskStatistics/comments/3xeluv/sem_and_sample_size_do_all_relationships/)).
For a typical CFA, the number of free parameters will be the number of
indicators \*2+ the number of covariances among the latent factors. For
example, with 3 factors and 3 indicators per factor, you would have
9 \* 2 + 3 = 21 free parameters, requiring [a minimum N of
210-420](https://www.safaribooksonline.com/library/view/structural-equation-modeling/9781118356302/c07anchor-1.html).
Note that in some cases this N:q rule is overkill; for more nuanced
discussion of appropriate minimum sample sizes for a variety of model
designs with and without missingness, see [Wolf et al.,
2013](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4334479/).

    lavaan WARNING: some observed variances are (at least) a factor 1000 times larger than others; use varTable(fit) to investigate

If you have indicator variables on very different scales, that can make
the covariance matrix problematic. An easy fix is to standardize some or
all of your variables before fitting the model with the `scale`
function.

    lavaan WARNING: could not compute standard errors!
    lavaan NOTE: this may be a symptom that the model is not identified.

If the model is not identified, that generally means its too complex
given the amount of information in the covariance matrix. Note that
increasing your N won't help here --- you actually need more indicators,
or to reduce the complexity of the model.

<h3 id="refs">
References and Further Reading
</h3>
Using `lavaan`: <http://lavaan.ugent.be/tutorial/cfa.html>

Interpreting CFA output (from mplus): [Notes from
IDRE](http://stats.idre.ucla.edu/mplus/seminars/intromplus-part2/mplus-class-notesconfirmatory-factor-analysis/),
[the CFA chapter from the Mplus User
Guide](https://www.statmodel.com/download/usersguide/Chapter5.pdf),
[David Kenny's quick
introduction](http://davidakenny.net/cm/mfactor.htm), and [this handy
blog post](http://www.thejuliagroup.com/blog/?p=3268)

For conducting power analyses for CFA or other SEM models, check out the
[simsem package](http://simsem.org/), designed to work elegantly with
`lavaan`.

You can automatically generate path diagrams from your lavaan models.
For a quick review of a few tools for doing that, see [this
appendix](https://blogs.baylor.edu/rlatentvariable/files/2015/09/AppendixCreatingPathModels-21y1cyz.pdf)
to Beaujean (2014).
