---
title: "ANOVA tables in R"
author: "Rose Hartman"
date: "2017-05-11"
categories: [ "Tutorials"]
tags: [ "annotated_output", "ANOVA", "R", "tables", "APA_style" ]
Draft: false
output: 
  md_document: 
    preserve_yaml: true
---

I don't know what fears keep you up at night, but for me it's worrying
that I might have copy-pasted the wrong values over from my output. No
matter how carefully I check my work, there's always the nagging
suspicion that I could have confused the contrasts for two different
factors, or missed a decimal point or a negative sign.

<!--more-->
Although I'm usually overreacting, I think my paranoia isn't completely
misplaced --- little errors are much too easy to make, and [they can
have horrifying
consequences](http://www.economist.com/blogs/graphicdetail/2016/09/daily-chart-3).

Through the years, I've learned that the only sure way to reduce human
error is to give humans (including myself) as little opportunity to
interfere in the process as possible. Happily, R integrates beautifully
with output documents, allowing you to ask the computer to fill in the
numbers in your tables and text for you, so you never have to wake up in
a cold sweat panicking about a typo in your correlation matrix. These
are called [dynamic
documents](https://www.amazon.ca/Dynamic-Documents-R-knitr-Second/dp/1498716962),
and they're awesome.

I almost never type out my results anymore; I let R do it for me. I
wrote my entire dissertation in R Studio, in fact, using
[sweave](https://support.rstudio.com/hc/en-us/articles/200552056-Using-Sweave-and-knitr)
to integrate my R code with [LaTeX](https://www.latex-project.org/)
typesetting. I'm writing this blog post in R Studio as an [R-markdown
document](http://rmarkdown.rstudio.com/); if you want to see the raw
.rmd file for this post, it's available on my github:
[ANOVA\_tables.Rmd](https://raw.githubusercontent.com/rosemm/understandingdata/master/content/post/ANOVA_tables.Rmd)

Even if you've never used markdown or R-markdown before, you can jump
right in and start getting APA output from R. In this tutorial, I'll
cover examples for one common model (an analysis of variance, or ANOVA)
and show you how you can get APA style output automatically.

We'll use one of the most basic functions for creating tables, `kable`,
which is from one of the most user-friendly packages for combining R
code and output together, `knitr`. (Side note to the fiber enthusiasts:
Yes, you're not imagining it --- pretty much all of this stuff is
playfully named after yarn, knitting, and textile references. Enjoy.) I
recommend `knitr` and `kable` for people just getting into writing
dynamic documents because they're the easiest and most flexible tools,
especially since they can be used to create Word documents (not just
pdfs or html pages). Depending on your desired output, though, you may
find other packages better suited to your needs. For example, if you're
creating pdf documents, you may prefer
[pander](https://cran.r-project.org/web/packages/pander/vignettes/pandoc_table.html),
[xtable](https://cran.r-project.org/web/packages/xtable/vignettes/xtableGallery.pdf)
or
[stargazer](https://cran.r-project.org/web/packages/stargazer/vignettes/stargazer.pdf),
all of which are much more powerful and elegant. Although these are
excellent packages, I find they don't work consistently (or at all) for
Word output, which is a deal breaker for a lot of people.

Quick links to content in this tutorial:

[Running an ANOVA in R](#model)

[Annotated output](#output)

[Creating an APA style ANOVA table from R output](#APAtable)

[Inline APA-style output](#APAinline)

[Recommended references](#refs)

### This tutorial assumes...

-   That you are using R Studio. If you don't have it already, [it's
    free to downloada nd
    install](https://www.rstudio.com/products/rstudio/download/), just
    like R.
-   That you already have a basic understanding of what an
    [ANOVA](https://en.wikipedia.org/wiki/Analysis_of_variance) is and
    when you might use it.
-   That you're not [brand new to
    R](http://blogs.uoregon.edu/rclub/2014/09/29/welcome-to-the-wonderful-world-of-r/).
    If you are, the descriptions may still be useful to you, but you may
    run into problems replicating the analysis on your own computer or
    editing the code to suit your needs.

<h3 id="model">
</h3>
### Running an ANOVA in R

#### Set up

Since the `kable` function is part of the `knitr` package, you'll need
to load `knitr` before you can use it:

    library(knitr)

We'll use a data set that comes built into R: `warpbreaks`. Fittingly,
it's about yarn. It gives the number of breaks in yarn tested under
conditions of low, medium, or high tension, for two types of wool (A and
B). This data comes standard with R, so you already have it on your
computer. You can read the help documentation about this data set by
typing `?warpbreaks`.

    str(warpbreaks) # check out the structure of the data

We'll run a 2x3 ANOVA to test if there are differences in the number of
breaks based on the type of wool and the amount of tension.

#### Exploratory data analysis

Before running a model, you always want to plot the data, to check that
your assumptions look okay. Here are a couple plots I might generate
while analyzing these data:

    library(ggplot2)

    ## Warning: package 'ggplot2' was built under R version 3.3.2

    # histograms, to check out the distribution within each group
    ggplot(warpbreaks, aes(x=breaks)) + 
      geom_histogram(bins=10) + 
      facet_grid(wool ~ tension) + 
      theme_classic()

![](ANOVA_tables_files/figure-markdown_strict/unnamed-chunk-3-1.png)

    # boxplot, to highlight the group means
    ggplot(warpbreaks, aes(y=breaks, x=tension, fill = wool)) + 
      geom_boxplot() + 
      theme_classic()

![](ANOVA_tables_files/figure-markdown_strict/unnamed-chunk-3-2.png)

The box plot gives me an idea of what I might find in the ANOVA. It
looks like there are differences between groups, with fewer breaks at
higher tension, and perhaps fewer breaks in wool B vs. wool A at both
low and high tension.

The distributions within each cell look pretty wonky, but that's not
particularly surprising given the small sample size (n=9):

    xtabs(~ wool + tension, data = warpbreaks)

    ##     tension
    ## wool L M H
    ##    A 9 9 9
    ##    B 9 9 9

#### Running the model

One important consideration when running ANOVAs in R is the coding of
factors (in this case, wool and tension). By default, R uses traditional
dummy coding (also called "treatment" coding), which works great for
regression-style output but can produce weird sums of squares estimates
for ANOVA style output.

To be on the safe side, always use effects coding (`contr.sum`) or
orthogonal contrast coding (e.g. `contr.helmert`, `contr.poly`) for
factors when running an ANOVA. Here, I'm choosing to use effects coding
for wool, and polynomial trend contrasts for tension.

    model <- lm(breaks ~ wool * tension, 
                data = warpbreaks, 
                contrasts = list(wool = "contr.sum", tension = "contr.poly"))

<h3 id="output">
</h3>
### Annotated ANOVA output

APA style ANOVA tables generally include the sums of squares, degrees of
freedom, *F* statistic, and *p* value for each effect. You can get all
of those calculations with the `Anova` function from the `car` package.
It's important to use the `Anova` function rather than the `summary.aov`
function in base R because `Anova` allows you to control the type of
sums of squares you want to calculate, whereas `summary.aov` only uses
Type 1 ([generally not what you want, especially if you have an
unblanced design and/or any missing
data](https://stats.stackexchange.com/questions/20452/how-to-interpret-type-i-type-ii-and-type-iii-anova-and-manova)).

    library(car)
    sstable <- Anova(model, type = 3) # Type III sums of squares is typical in social science research (it's the default in SPSS)

    sstable 

    ## Anova Table (Type III tests)
    ## 
    ## Response: breaks
    ##              Sum Sq Df  F value    Pr(>F)    
    ## (Intercept)   42785  1 357.4672 < 2.2e-16 ***
    ## wool            451  1   3.7653 0.0582130 .  
    ## tension        2034  2   8.4980 0.0006926 ***
    ## wool:tension   1003  2   4.1891 0.0210442 *  
    ## Residuals      5745 48                       
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

The above code runs the `Anova` function on the model I saved before,
using Type III sums of squares, and saves the resulting table as a new
object called `sstable`.

In `sstable`, you can see a row for each predictor in the model,
including the intercept, and the error term (`Residuals`) at the bottom.
The `wool:tension` term is the interaction between wool and tension (R
uses `:` to specify interaction terms, and `*` as shorthand for an
interaction term with both main effects). There are two levels of wool
(A and B), so you'll see 1 degree of freedom for that effect. There are
three levels of tension (low, medium, and high), so that has 2 degrees
of freedom. The interaction has the df for both terms multiplied
together, i.e. 1 \* 2 = 2. The degrees of freedom for the residual are
based on the total number of observations in the data (N=54) minus the
number of groups, i.e. 54-6=48.

The *F*-statistic for each effect is the *S**S*/*d**f* for that effect
divided by the *S**S*/*d**f* for the residual. The `Pr(>F)` gives the
*p* value for that test, i.e. the probability of observing an *F* ratio
greater than that given the null hypothesis is true.

#### Contrast estimates

    summary.aov(model, split = list(tension=list(L=1, Q=2)))

    ##                   Df Sum Sq Mean Sq F value   Pr(>F)    
    ## wool               1    451   450.7   3.765 0.058213 .  
    ## tension            2   2034  1017.1   8.498 0.000693 ***
    ##   tension: L       1   1951  1950.7  16.298 0.000194 ***
    ##   tension: Q       1     84    83.6   0.698 0.407537    
    ## wool:tension       2   1003   501.4   4.189 0.021044 *  
    ##   wool:tension: L  1    251   250.7   2.095 0.154327    
    ##   wool:tension: Q  1    752   752.1   6.284 0.015626 *  
    ## Residuals         48   5745   119.7                     
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

(Note that if you run `summary(model)` instead, you'll get the default
regression-style output, which is the same information, but represented
as regression coefficients with standard errors and *t*-tests instead of
sums of squares estimates with *F* ratios.)

What we see here is a significant linear trend in the main effect for
tension --- from the box plot we made before, we know that it's a
negative linear trend. There's no overall quadratic trend in tension,
but there is a quadratic trend in the interaction between wool and
tension. That means the estimate of the quadratic trend contrast is
different for wool A compared to wool B. Referencing the box plot again,
you can see that the direction of the quadratic trend appears to differ
in wool A compared to wool B (wool A looks like a positive quadratic
trend, with an upward swoop, whereas wool B looks like a negative
quadratic trend, with an upside-down U swoop). There is no linear trend
for the interaction, however, so that the estimate of the linear trend
in wool A is not significantly different from the estimate of the linear
trend in wool B.

Remember that `summary.aov` is using Type I sums of squares, so the
estimates for some effects may not be what we want. In this example, the
design is balanced and there are no missing data, so the SS estimates
using Type I and Type III work out to be the same, but in your own data
there may be a difference. Note that our orthogonal contrasts here are
simple comparisons between means, and aren't affected by the type of SS
used. If you are concerned about Type of SS, you may want to grab the
contrast estimates from this output and put them into your other
`sstable` object. Here's how you could do that:

Note which rows in the output correspond to the contrasts you want. In
this case, it's rows 3 and 4 for the contrasts on the main effect of
tension, and rows 6 and 7 for the contrasts on the interaction. I select
those rows with `c(3, 4, 6, 7)`. I'm also selecting and reordering the
columns in the output, so they'll match what we have in `sstable`. I
select the 2nd column (Sum Sq), then the first (Df), then the fourth (F
value), then the fifth (Pr(&gt;F)) with `c(2, 1, 4, 5)`.

Remember that you can use `[ , ]` to select particular combinations of
rows and columns from a given matrix or dataframe. Just put the rows you
want as the first argument, and the columns as the second, i.e.
`[r, c]`. If you leave either the rows or the columns blank, it will
return all (so `[r, ]` will return row r and all columns).

    # this pulls out just the specified rows
    contrasts <- summary.aov(model, split = list(tension=list(L=1, Q=2)))[[1]][c(3, 4, 6, 7), c(2, 1, 4, 5)]

    contrasts

    ##                    Sum Sq Df F value    Pr(>F)    
    ##   tension: L      1950.69  1 16.2979 0.0001938 ***
    ##   tension: Q        83.56  1  0.6982 0.4075366    
    ##   wool:tension: L  250.69  1  2.0945 0.1543266    
    ##   wool:tension: Q  752.08  1  6.2836 0.0156262 *  
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

Now use `rbind` to create a sort of Frankenstein table, splicing the
contrasts estimate rows in with the other rows of `sstable`.

    # select the rows to combine
    maineffects <- sstable[c(1,2,3), ]
    me_contrasts <- contrasts[c(1,2), ]
    interaction <- sstable[4, ]
    int_contrasts <- contrasts[c(3,4), ]
    resid <- sstable[5, ]

    # bind the rows together in the desired order
    sstable <- rbind(maineffects, me_contrasts, interaction, int_contrasts, resid)

    sstable # ta-da!

    ## Anova Table (Type III tests)
    ## 
    ## Response: breaks
    ##                   Sum Sq Df  F value    Pr(>F)    
    ## (Intercept)        42785  1 357.4672 < 2.2e-16 ***
    ## wool                 451  1   3.7653 0.0582130 .  
    ## tension             2034  2   8.4980 0.0006926 ***
    ##   tension: L        1951  1  16.2979 0.0001938 ***
    ##   tension: Q          84  1   0.6982 0.4075366    
    ## wool:tension        1003  2   4.1891 0.0210442 *  
    ##   wool:tension: L    251  1   2.0945 0.1543266    
    ##   wool:tension: Q    752  1   6.2836 0.0156262 *  
    ## Residuals           5745 48                       
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

#### Estimates of effect size

A popular measure of effect size for ANOVAs (and other linear models) is
partial eta-squared. It's the sums of squares for each effect divided by
the error SS. The following code adds a column to the `sstable` object
with partial eta-squared estimates for each effect:

    sstable$pes <- c(sstable$'Sum Sq'[-nrow(sstable)], NA)/(sstable$'Sum Sq' + sstable$'Sum Sq'[nrow(sstable)]) # SS for each effect divided by the last SS (SS_residual)

    sstable

    ## Anova Table (Type III tests)
    ## 
    ## Response: breaks
    ##                   Sum Sq Df  F value  Pr(>F)     pes
    ## (Intercept)        42785  1 357.4672 0.00000 0.88162
    ## wool                 451  1   3.7653 0.05821 0.07274
    ## tension             2034  2   8.4980 0.00069 0.26149
    ##   tension: L        1951  1  16.2979 0.00019 0.25348
    ##   tension: Q          84  1   0.6982 0.40754 0.01434
    ## wool:tension        1003  2   4.1891 0.02104 0.14861
    ##   wool:tension: L    251  1   2.0945 0.15433 0.04181
    ##   wool:tension: Q    752  1   6.2836 0.01563 0.11576
    ## Residuals           5745 48

Okay great! There's your output, but you don't want to just copy-paste
that mess into your manuscript. Let's get R to generate a nice, clean
table we can use in Word.

<h3 id="APAtable">
</h3>
### Creating an APA style ANOVA table from R output

    kable(sstable, digits = 3) # the digits argument controls rounding

<table>
<thead>
<tr class="header">
<th></th>
<th align="right">Sum Sq</th>
<th align="right">Df</th>
<th align="right">F value</th>
<th align="right">Pr(&gt;F)</th>
<th align="right">pes</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>(Intercept)</td>
<td align="right">42785.185</td>
<td align="right">1</td>
<td align="right">357.467</td>
<td align="right">0.000</td>
<td align="right">0.882</td>
</tr>
<tr class="even">
<td>wool</td>
<td align="right">450.667</td>
<td align="right">1</td>
<td align="right">3.765</td>
<td align="right">0.058</td>
<td align="right">0.073</td>
</tr>
<tr class="odd">
<td>tension</td>
<td align="right">2034.259</td>
<td align="right">2</td>
<td align="right">8.498</td>
<td align="right">0.001</td>
<td align="right">0.261</td>
</tr>
<tr class="even">
<td>tension: L</td>
<td align="right">1950.694</td>
<td align="right">1</td>
<td align="right">16.298</td>
<td align="right">0.000</td>
<td align="right">0.253</td>
</tr>
<tr class="odd">
<td>tension: Q</td>
<td align="right">83.565</td>
<td align="right">1</td>
<td align="right">0.698</td>
<td align="right">0.408</td>
<td align="right">0.014</td>
</tr>
<tr class="even">
<td>wool:tension</td>
<td align="right">1002.778</td>
<td align="right">2</td>
<td align="right">4.189</td>
<td align="right">0.021</td>
<td align="right">0.149</td>
</tr>
<tr class="odd">
<td>wool:tension: L</td>
<td align="right">250.694</td>
<td align="right">1</td>
<td align="right">2.095</td>
<td align="right">0.154</td>
<td align="right">0.042</td>
</tr>
<tr class="even">
<td>wool:tension: Q</td>
<td align="right">752.083</td>
<td align="right">1</td>
<td align="right">6.284</td>
<td align="right">0.016</td>
<td align="right">0.116</td>
</tr>
<tr class="odd">
<td>Residuals</td>
<td align="right">5745.111</td>
<td align="right">48</td>
<td align="right">NA</td>
<td align="right">NA</td>
<td align="right">NA</td>
</tr>
</tbody>
</table>

Wait, what? That was so easy!

Yes, yes it was. :)

In a lot of cases, that will be all you need to get a workable ANOVA
table in your document. Just for fun, though, let's play around with
customizing it a little.

#### Hide missing values

By default, `kable` displays missing values in a table as NA, but in
this case we'd rather have them just be blank. You can control that with
the `options` command:

    options(knitr.kable.NA = '') # this will hide missing values in the kable table

    kable(sstable, digits = 3)

<table>
<thead>
<tr class="header">
<th></th>
<th align="right">Sum Sq</th>
<th align="right">Df</th>
<th align="right">F value</th>
<th align="right">Pr(&gt;F)</th>
<th align="right">pes</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>(Intercept)</td>
<td align="right">42785.185</td>
<td align="right">1</td>
<td align="right">357.467</td>
<td align="right">0.000</td>
<td align="right">0.882</td>
</tr>
<tr class="even">
<td>wool</td>
<td align="right">450.667</td>
<td align="right">1</td>
<td align="right">3.765</td>
<td align="right">0.058</td>
<td align="right">0.073</td>
</tr>
<tr class="odd">
<td>tension</td>
<td align="right">2034.259</td>
<td align="right">2</td>
<td align="right">8.498</td>
<td align="right">0.001</td>
<td align="right">0.261</td>
</tr>
<tr class="even">
<td>tension: L</td>
<td align="right">1950.694</td>
<td align="right">1</td>
<td align="right">16.298</td>
<td align="right">0.000</td>
<td align="right">0.253</td>
</tr>
<tr class="odd">
<td>tension: Q</td>
<td align="right">83.565</td>
<td align="right">1</td>
<td align="right">0.698</td>
<td align="right">0.408</td>
<td align="right">0.014</td>
</tr>
<tr class="even">
<td>wool:tension</td>
<td align="right">1002.778</td>
<td align="right">2</td>
<td align="right">4.189</td>
<td align="right">0.021</td>
<td align="right">0.149</td>
</tr>
<tr class="odd">
<td>wool:tension: L</td>
<td align="right">250.694</td>
<td align="right">1</td>
<td align="right">2.095</td>
<td align="right">0.154</td>
<td align="right">0.042</td>
</tr>
<tr class="even">
<td>wool:tension: Q</td>
<td align="right">752.083</td>
<td align="right">1</td>
<td align="right">6.284</td>
<td align="right">0.016</td>
<td align="right">0.116</td>
</tr>
<tr class="odd">
<td>Residuals</td>
<td align="right">5745.111</td>
<td align="right">48</td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
</tr>
</tbody>
</table>

#### Add a table caption

You can add a title for the table if you change the format to "pandoc".
Depending on your final document output (pdf, html, Word, etc.), you can
get automatic table numbering this way as well, which saves much time
and many headaches.

    kable(sstable, digits = 3, format = "pandoc", caption = "ANOVA table")

<table>
<caption>ANOVA table</caption>
<thead>
<tr class="header">
<th></th>
<th align="right">Sum Sq</th>
<th align="right">Df</th>
<th align="right">F value</th>
<th align="right">Pr(&gt;F)</th>
<th align="right">pes</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>(Intercept)</td>
<td align="right">42785.185</td>
<td align="right">1</td>
<td align="right">357.467</td>
<td align="right">0.000</td>
<td align="right">0.882</td>
</tr>
<tr class="even">
<td>wool</td>
<td align="right">450.667</td>
<td align="right">1</td>
<td align="right">3.765</td>
<td align="right">0.058</td>
<td align="right">0.073</td>
</tr>
<tr class="odd">
<td>tension</td>
<td align="right">2034.259</td>
<td align="right">2</td>
<td align="right">8.498</td>
<td align="right">0.001</td>
<td align="right">0.261</td>
</tr>
<tr class="even">
<td>tension: L</td>
<td align="right">1950.694</td>
<td align="right">1</td>
<td align="right">16.298</td>
<td align="right">0.000</td>
<td align="right">0.253</td>
</tr>
<tr class="odd">
<td>tension: Q</td>
<td align="right">83.565</td>
<td align="right">1</td>
<td align="right">0.698</td>
<td align="right">0.408</td>
<td align="right">0.014</td>
</tr>
<tr class="even">
<td>wool:tension</td>
<td align="right">1002.778</td>
<td align="right">2</td>
<td align="right">4.189</td>
<td align="right">0.021</td>
<td align="right">0.149</td>
</tr>
<tr class="odd">
<td>wool:tension: L</td>
<td align="right">250.694</td>
<td align="right">1</td>
<td align="right">2.095</td>
<td align="right">0.154</td>
<td align="right">0.042</td>
</tr>
<tr class="even">
<td>wool:tension: Q</td>
<td align="right">752.083</td>
<td align="right">1</td>
<td align="right">6.284</td>
<td align="right">0.016</td>
<td align="right">0.116</td>
</tr>
<tr class="odd">
<td>Residuals</td>
<td align="right">5745.111</td>
<td align="right">48</td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
</tr>
</tbody>
</table>

#### Modify column and row names

Often, the automatic row names and column names aren't quite what you
want. If so, you'll need to modify them for the `sstable` object itself,
and then run `kable` on the updated object.

    colnames(sstable) <- c("SS", "df", "$F$", "$p$", "partial $\\eta^2$")

    rownames(sstable) <- c("(Intercept)", "Wool", "Tension", "Tension: Linear Trend", "Tension: Quadratic Trend", "Wool x Tension", "Wool x Tension: Linear Trend", "Wool x Tension: Quadratic Trend", "Residuals")

    kable(sstable, digits = 3, format = "pandoc", caption = "ANOVA table")

<table>
<caption>ANOVA table</caption>
<thead>
<tr class="header">
<th></th>
<th align="right">SS</th>
<th align="right">df</th>
<th align="right"><span class="math inline"><em>F</em></span></th>
<th align="right"><span class="math inline"><em>p</em></span></th>
<th align="right">partial <span class="math inline"><em>η</em><sup>2</sup></span></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>(Intercept)</td>
<td align="right">42785.185</td>
<td align="right">1</td>
<td align="right">357.467</td>
<td align="right">0.000</td>
<td align="right">0.882</td>
</tr>
<tr class="even">
<td>Wool</td>
<td align="right">450.667</td>
<td align="right">1</td>
<td align="right">3.765</td>
<td align="right">0.058</td>
<td align="right">0.073</td>
</tr>
<tr class="odd">
<td>Tension</td>
<td align="right">2034.259</td>
<td align="right">2</td>
<td align="right">8.498</td>
<td align="right">0.001</td>
<td align="right">0.261</td>
</tr>
<tr class="even">
<td>Tension: Linear Trend</td>
<td align="right">1950.694</td>
<td align="right">1</td>
<td align="right">16.298</td>
<td align="right">0.000</td>
<td align="right">0.253</td>
</tr>
<tr class="odd">
<td>Tension: Quadratic Trend</td>
<td align="right">83.565</td>
<td align="right">1</td>
<td align="right">0.698</td>
<td align="right">0.408</td>
<td align="right">0.014</td>
</tr>
<tr class="even">
<td>Wool x Tension</td>
<td align="right">1002.778</td>
<td align="right">2</td>
<td align="right">4.189</td>
<td align="right">0.021</td>
<td align="right">0.149</td>
</tr>
<tr class="odd">
<td>Wool x Tension: Linear Trend</td>
<td align="right">250.694</td>
<td align="right">1</td>
<td align="right">2.095</td>
<td align="right">0.154</td>
<td align="right">0.042</td>
</tr>
<tr class="even">
<td>Wool x Tension: Quadratic Trend</td>
<td align="right">752.083</td>
<td align="right">1</td>
<td align="right">6.284</td>
<td align="right">0.016</td>
<td align="right">0.116</td>
</tr>
<tr class="odd">
<td>Residuals</td>
<td align="right">5745.111</td>
<td align="right">48</td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
</tr>
</tbody>
</table>

#### Omit the intercept row

For many models, the intercept is not of any theoretical interest, and
you may want to omit it from the output. If you just want to drop one
row (or column), the easiest approach is to indicate that row's number
and put a minus sign before it:

    kable(sstable[-1, ], digits = 3, format = "pandoc", caption = "ANOVA table")

<table>
<caption>ANOVA table</caption>
<thead>
<tr class="header">
<th></th>
<th align="right">SS</th>
<th align="right">df</th>
<th align="right"><span class="math inline"><em>F</em></span></th>
<th align="right"><span class="math inline"><em>p</em></span></th>
<th align="right">partial <span class="math inline"><em>η</em><sup>2</sup></span></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>Wool</td>
<td align="right">450.667</td>
<td align="right">1</td>
<td align="right">3.765</td>
<td align="right">0.058</td>
<td align="right">0.073</td>
</tr>
<tr class="even">
<td>Tension</td>
<td align="right">2034.259</td>
<td align="right">2</td>
<td align="right">8.498</td>
<td align="right">0.001</td>
<td align="right">0.261</td>
</tr>
<tr class="odd">
<td>Tension: Linear Trend</td>
<td align="right">1950.694</td>
<td align="right">1</td>
<td align="right">16.298</td>
<td align="right">0.000</td>
<td align="right">0.253</td>
</tr>
<tr class="even">
<td>Tension: Quadratic Trend</td>
<td align="right">83.565</td>
<td align="right">1</td>
<td align="right">0.698</td>
<td align="right">0.408</td>
<td align="right">0.014</td>
</tr>
<tr class="odd">
<td>Wool x Tension</td>
<td align="right">1002.778</td>
<td align="right">2</td>
<td align="right">4.189</td>
<td align="right">0.021</td>
<td align="right">0.149</td>
</tr>
<tr class="even">
<td>Wool x Tension: Linear Trend</td>
<td align="right">250.694</td>
<td align="right">1</td>
<td align="right">2.095</td>
<td align="right">0.154</td>
<td align="right">0.042</td>
</tr>
<tr class="odd">
<td>Wool x Tension: Quadratic Trend</td>
<td align="right">752.083</td>
<td align="right">1</td>
<td align="right">6.284</td>
<td align="right">0.016</td>
<td align="right">0.116</td>
</tr>
<tr class="even">
<td>Residuals</td>
<td align="right">5745.111</td>
<td align="right">48</td>
<td align="right"></td>
<td align="right"></td>
<td align="right"></td>
</tr>
</tbody>
</table>

<h3 id="APAinline">
</h3>
### Inline APA-style output

You can also knit R output right into your typed sentences! To include
inline R output, use back-ticks, like this:

    Here's a sentence, and I want to let you know that the total number of cats I own is `r length(my_cats)`.

The back-ticks mark out the code to run, and the `r` after the first
back-tick tells `knitr` that it's R code (if you feel the need, you can
incorporate code from pretty much any language you like, not just R).
Assuming there's a vector called `my_cats`, when we knit the document,
that line of code will be evaluated and the result (the number of items
in the vector `my_cats`) will be printed right in that sentence.

Let's work this into our ANOVA reporting.

Here's an example write-up of this ANOVA, using inline code to plug in
the stats. Since the inline code can be a little hard to read, I like to
save all of the variables I want to use inline with convenient names
first.

    fstat <- unname(summary(model)$fstatistic[1])
    df_model <- unname(summary(model)$fstatistic[2])
    df_res <- unname(summary(model)$fstatistic[3])
    rsq <- summary(model)$r.squared
    p <- pf(fstat, df_model, df_res, lower.tail = FALSE)

Then I can plug them in as needed in my writing.

    A 2x2 factorial ANOVA revealed that tension (low, medium, or high) and wool type (A or B) predict a significant amount of variane in number of breaks, $R^2$=`r round(rsq, 2)`, $F(`r round(df_model, 0)`, `r round(df_res, 0)`)=`r round(fstat, 2)`$, $p=`r ifelse(round(p, 3) == 0, "<.001", round(p, 3))`$.

Here's how that code renders when knit: A 2x2 factorial ANOVA revealed
that tension (low, medium, or high) and wool type (A or B) predict a
significant amount of variane in number of breaks, *R*<sup>2</sup>=0.38,
*F*(5, 48)=5.83, *p* = &lt;.001.

<h3 id="refs">
</h3>
### References and Further Reading

Coming soon. :)
