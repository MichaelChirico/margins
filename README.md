---
title: "Marginal Effects for Model Objects"
output: github_document
---

<img src="man/figures/logo.png" align="right" />

The **margins** and **prediction** packages are a combined effort to port the functionality of Stata's (closed source) [`margins`]( https://www.stata.com/help.cgi?margins) command to (open source) R. These tools provide ways of obtaining common quantities of interest from regression-type models. **margins** provides "marginal effects" summaries of models and **prediction** provides unit-specific and sample average predictions from models. Marginal effects are partial derivatives of the regression equation with respect to each variable in the model for each unit in the data; average marginal effects are simply the mean of these unit-specific partial derivatives over some sample. In ordinary least squares regression with no interactions or higher-order term, the estimated slope coefficients are marginal effects. In other cases and for generalized linear models, the coefficients are not marginal effects at least not on the scale of the response variable. **margins** therefore provides ways of calculating the marginal effects of variables to make these models more interpretable.

The major functionality of Stata's `margins` command - namely the estimation of marginal (or partial) effects - is provided here through a single function, `margins()`. This is an S3 generic method for calculating the marginal effects of covariates included in model objects (like those of classes "lm" and "glm"). Users interested in generating predicted (fitted) values, such as the "predictive margins" generated by Stata's `margins` command, should consider using `prediction()` from the sibling project, [**prediction**](https://cran.r-project.org/package=prediction).

## Motivation

With the introduction of Stata's `margins` command, it has become incredibly simple to estimate average marginal effects (i.e., "average partial effects") and marginal effects at representative cases. Indeed, in just a few lines of Stata code, regression results for almost any kind model can be transformed into meaningful quantities of interest and related plots:

```
. import delimited mtcars.csv
. quietly reg mpg c.cyl##c.hp wt
. margins, dydx(*)
------------------------------------------------------------------------------
             |            Delta-method
             |      dy/dx   Std. Err.      t    P>|t|     [95% Conf. Interval]
-------------+----------------------------------------------------------------
         cyl |   .0381376   .5998897     0.06   0.950    -1.192735     1.26901
          hp |  -.0463187    .014516    -3.19   0.004     -.076103   -.0165343
          wt |  -3.119815    .661322    -4.72   0.000    -4.476736   -1.762894
------------------------------------------------------------------------------
. marginsplot
```

![marginsplot](http://i.imgur.com/VhoaFGp.png)

Stata's `margins` command is incredibly robust. It works with nearly any kind of statistical model and estimation procedure, including OLS, generalized linear models, panel regression models, and so forth. It also represents a significant improvement over Stata's previous marginal effects command - `mfx` - which was subject to various well-known bugs. While other Stata modules have provided functionality for deriving quantities of interest from regression estimates (e.g., [Clarify](https://gking.harvard.edu/clarify)), none has done so with the simplicity and genearlity of `margins`.

By comparison, R has no robust functionality in the base tools for drawing out marginal effects from model estimates (though the S3 `predict()` methods implement some of the functionality for computing fitted/predicted values). The closest approximation is [modmarg](https://cran.r-project.org/package=modmarg), which does one-variable-at-a-time estimation of marginal effects is quite robust. Other than this relatively new package on the scene, no packages implement appropriate marginal effect estimates. Notably, several packages provide estimates of marginal effects for different types of models. Among these are [car](https://cran.r-project.org/package=car), [alr3](https://cran.r-project.org/package=alr3), [mfx](https://cran.r-project.org/package=mfx), [erer](https://cran.r-project.org/package=erer), among others. Unfortunately, none of these packages implement marginal effects correctly (i.e., correctly account for interrelated variables such as interaction terms (e.g., `a:b`) or power terms (e.g., `I(a^2)`) and the packages all implement quite different interfaces for different types of models. [interflex](https://cran.r-project.org/package=interflex), [interplot](https://cran.r-project.org/package=interplot), and [plotMElm](https://cran.r-project.org/package=plotMElm) provide functionality simply for plotting quantities of interest from multiplicative interaction terms in models but do not appear to support general marginal effects displays (in either tabular or graphical form), while [visreg](https://cran.r-project.org/package=visreg) provides a more general plotting function but no tabular output. [interactionTest](https://cran.r-project.org/package=interactionTest) provides some additional useful functionality for controlling the false discovery rate when making such plots and interpretations, but is again not a general tool for marginal effect estimation.

Given the challenges of interpreting the contribution of a given regressor in any model that includes quadratic terms, multiplicative interactions, a non-linear transformation, or other complexities, there is a clear need for a simple, consistent way to estimate marginal effects for popular statistical models. This package aims to correctly calculate marginal effects that include complex terms and provide a uniform interface for doing those calculations. Thus, the package implements a single S3 generic method (`margins()`) that can be easily generalized for any type of model implemented in R.

Some technical details of the package are worth briefly noting. The estimation of marginal effects relies on numerical approximations of derivatives produced using `predict()` (actually, a wrapper around `predict()` called `prediction()` that is type-safe). Variance estimation, by default is provided using the delta method a numerical approximation of [the Jacobian matrix](https://en.wikipedia.org/wiki/Jacobian_matrix_and_determinant). While symbolic differentiation of some models (e.g., basic linear models) is possible using `D()` and `deriv()`, R's modelling language (the "formula" class) is sufficiently general to enable the construction of model formulae that contain terms that fall outside of R's symbolic differentiation rule table (e.g., `y ~ factor(x)` or `y ~ I(FUN(x))` for any arbitrary `FUN()`). By relying on numeric differentiation, `margins()` supports *any* model that can be expressed in R formula syntax. Even Stata's `margins` command is limited in its ability to handle variable transformations (e.g., including `x` and `log(x)` as predictors) and quadratic terms (e.g., `x^3`); these scenarios are easily expressed in an R formula and easily handled, correctly, by `margins()`.

## Simple code examples



Replicating Stata's results is incredibly simple using just the `margins()` method to obtain average marginal effects:


```r
library("margins")
mod1 <- lm(mpg ~ cyl * hp + wt, data = mtcars)
(marg1 <- margins(mod1))
```

```
## Average marginal effects
```

```
## lm(formula = mpg ~ cyl * hp + wt, data = mtcars)
```

```
##      cyl       hp    wt
##  0.03814 -0.04632 -3.12
```

```r
summary(marg1)
```

```
##  factor     AME     SE       z      p   lower   upper
##     cyl  0.0381 0.5999  0.0636 0.9493 -1.1376  1.2139
##      hp -0.0463 0.0145 -3.1909 0.0014 -0.0748 -0.0179
##      wt -3.1198 0.6613 -4.7175 0.0000 -4.4160 -1.8236
```

With the exception of differences in rounding, the above results match identically what Stata's `margins` command produces. A slightly more concise expression relies on the syntactic sugar provided by `margins_summary()`:


```r
margins_summary(mod1)
```

```
##  factor     AME     SE       z      p   lower   upper
##     cyl  0.0381 0.5999  0.0636 0.9493 -1.1376  1.2139
##      hp -0.0463 0.0145 -3.1909 0.0014 -0.0748 -0.0179
##      wt -3.1198 0.6613 -4.7175 0.0000 -4.4160 -1.8236
```

If you are only interested in obtaining the marginal effects (without corresponding variances or the overhead of creating a "margins" object), you can call `marginal_effects(x)` directly. Furthermore, the `dydx()` function enables the calculation of the marginal effect of a single named variable:


```r
# all marginal effects, as a data.frame
head(marginal_effects(mod1))
```

```
##     dydx_cyl     dydx_hp   dydx_wt
## 1 -0.6572244 -0.04987248 -3.119815
## 2 -0.6572244 -0.04987248 -3.119815
## 3 -0.9794364 -0.08777977 -3.119815
## 4 -0.6572244 -0.04987248 -3.119815
## 5  0.5747624 -0.01196519 -3.119815
## 6 -0.7519926 -0.04987248 -3.119815
```

```r
# subset of all marginal effects, as a data.frame
head(marginal_effects(mod1, variables = c("cyl", "hp")))
```

```
##     dydx_cyl     dydx_hp
## 1 -0.6572244 -0.04987248
## 2 -0.6572244 -0.04987248
## 3 -0.9794364 -0.08777977
## 4 -0.6572244 -0.04987248
## 5  0.5747624 -0.01196519
## 6 -0.7519926 -0.04987248
```

```r
# marginal effect of one variable
head(dydx(mtcars, mod1, "cyl"))
```

```
##     dydx_cyl
## 1 -0.6572244
## 2 -0.6572244
## 3 -0.9794364
## 4 -0.6572244
## 5  0.5747624
## 6 -0.7519926
```

These functions may be useful for plotting, getting a quick impression of the results, or for using unit-specific marginal effects in further analyses.


### Counterfactual Datasets (`at`) and Subgroup Analyses

The package also implement's one of the best features of `margins`, which is the `at` specification that allows for the estimation of average marginal effects for counterfactual datasets in which particular variables are held at fixed values:


```r
# webuse margex
library("webuse")
```

```
## Error in library("webuse"): there is no package called 'webuse'
```

```r
webuse::webuse("margex")
```

```
## Error in loadNamespace(x): there is no package called 'webuse'
```

```r
# logistic outcome treatment##group age c.age#c.age treatment#c.age
mod2 <- glm(outcome ~ treatment * group + age + I(age^2) * treatment, data = margex, family = binomial)
```

```
## Error in eval(mf, parent.frame()): object 'margex' not found
```

```r
# margins, dydx(*)
summary(margins(mod2))
```

```
## Error in eval(expr, envir, enclos): object 'mod2' not found
```

```r
# margins, dydx(treatment) at(age=(20(10)60))
summary(margins(mod2, at = list(age = c(20, 30, 40, 50, 60)), variables = "treatment"))
```

```
## Error in eval(expr, envir, enclos): object 'mod2' not found
```

This functionality removes the need to modify data before performing such calculations, which can be quite unwieldy when many specifications are desired.

If one desires *subgroup* effects, simply pass a subset of data to the `data` argument:


```r
# effects for men
summary(margins(mod2, data = subset(margex, sex == 0)))
```

```
## Error in eval(expr, envir, enclos): object 'mod2' not found
```

```r
# effects for wommen
summary(margins(mod2, data = subset(margex, sex == 1)))
```

```
## Error in eval(expr, envir, enclos): object 'mod2' not found
```


### Plotting and Visualization

The package implements several useful additional features for summarizing model objects, including:

 - A `plot()` method for the new "margins" class that ports Stata's `marginsplot` command. 
 - A plotting function `cplot()` to provide the commonly needed visual summaries of predictions or average marginal effects conditional on a covariate.
 - A `persp()` method for "lm", "glm", and "loess" objects to provide three-dimensional representations of response surfaces or marginal effects over two covariates.
 - An `image()` method for the same that produces flat, two-dimensional heatmap-style representations of `persp()`-type plots.

Using the `plot()` method yields an aesthetically similar result to Stata's `marginsplot`:


```r
library("webuse")
```

```
## Error in library("webuse"): there is no package called 'webuse'
```

```r
webuse::webuse("nhanes2")
```

```
## Error in loadNamespace(x): there is no package called 'webuse'
```

```r
mod3 <- glm(highbp ~ sex * agegrp * bmi, data = nhanes2, family = binomial)
```

```
## Error in eval(mf, parent.frame()): object 'nhanes2' not found
```

```r
summary(marg3 <- margins(mod3))
```

```
## Error in eval(expr, envir, enclos): object 'mod3' not found
```

```r
plot(marg3)
```

```
## Error in eval(expr, envir, enclos): object 'marg3' not found
```

In addition to the estimation procedures and `plot()` generic, **margins** offers several plotting methods for model objects. First, there is a new generic `cplot()` that displays predictions or marginal effects (from an "lm" or "glm" model) of a variable conditional across values of third variable (or itself). For example, here is a graph of predicted probabilities from a logit model:


```r
mod4 <- glm(am ~ wt*drat, data = mtcars, family = binomial)
cplot(mod4, x = "wt", se.type = "shade")
```

![plot of chunk cplot1](https://i.imgur.com/E2hyOMh.png)

And fitted values with a factor independent variable:


```r
cplot(lm(Sepal.Length ~ Species, data = iris))
```

![plot of chunk cplot2](https://i.imgur.com/91Wz84U.png)

and a graph of the effect of `drat` across levels of `wt`:


```r
cplot(mod4, x = "wt", dx = "drat", what = "effect", se.type = "shade")
```

![plot of chunk cplot3](https://i.imgur.com/0KWdMKv.png)

`cplot()` also returns a data frame of values, so that it can be used just for calculating quantities of interest before plotting them with another graphics package, such as **ggplot2**:


```r
library("ggplot2")
dat <- cplot(mod4, x = "wt", dx = "drat", what = "effect", draw = FALSE)
head(dat)
```

```
##   xvals  yvals  upper   lower factor
##  1.5130 0.3250 1.3927 -0.7426   drat
##  1.6760 0.3262 1.1318 -0.4795   drat
##  1.8389 0.3384 0.9214 -0.2447   drat
##  2.0019 0.3623 0.7777 -0.0531   drat
##  2.1648 0.3978 0.7110  0.0846   drat
##  2.3278 0.4432 0.7074  0.1789   drat
```

```r
ggplot(dat, aes(x = xvals)) +
  geom_ribbon(aes(ymin = lower, ymax = upper), fill = "gray70") +
  geom_line(aes(y = yvals)) +
  xlab("Vehicle Weight (1000s of lbs)") +
  ylab("Average Marginal Effect of Rear Axle Ratio") +
  ggtitle("Predicting Automatic/Manual Transmission from Vehicle Characteristics") +
  theme_bw()
```

![plot of chunk cplot_ggplot2](https://i.imgur.com/O9Bw8FD.png)

Second, the package implements methods for "lm" and "glm" class objects for the `persp()` generic plotting function. This enables three-dimensional representations of predicted outcomes:


```r
persp(mod1, xvar = "cyl", yvar = "hp")
```

![plot of chunk persp1](https://i.imgur.com/Y2QEvxt.png)

and marginal effects:


```r
persp(mod1, xvar = "cyl", yvar = "hp", what = "effect", nx = 10)
```

![plot of chunk persp2](https://i.imgur.com/QCqecrO.png)

And if three-dimensional plots aren't your thing, there are also analogous methods for the `image()` generic, to produce heatmap-style representations:


```r
image(mod1, xvar = "cyl", yvar = "hp", main = "Predicted Fuel Efficiency,\nby Cylinders and Horsepower")
```

![plot of chunk image11](https://i.imgur.com/Gg5IMrV.png)

The numerous package vignettes and help files contain extensive documentation and examples of all package functionality.

### Performance

While there is still work to be done to improve performance, **margins** is reasonably speedy:


```r
library("microbenchmark")
microbenchmark(marginal_effects(mod1))
```

```
## Unit: milliseconds
##                    expr      min       lq     mean   median       uq      max neval
##  marginal_effects(mod1) 2.219172 2.321284 2.588372 2.392883 2.525417 5.871072   100
```

```r
microbenchmark(margins(mod1))
```

```
## Unit: milliseconds
##           expr      min       lq     mean   median       uq      max neval
##  margins(mod1) 16.34965 16.62798 17.41705 16.90862 17.23319 24.19629   100
```

The most computationally expensive part of `margins()` is variance estimation. If you don't need variances, use `marginal_effects()` directly or specify `margins(..., vce = "none")`.


## Requirements and Installation

[![CRAN](http://www.r-pkg.org/badges/version/margins)](https://cran.r-project.org/package=margins)
![Downloads](http://cranlogs.r-pkg.org/badges/margins)
<!-- [![Build Status](https://app.travis-ci.org/leeper/margins.svg?branch=master)](https://app.travis-ci.org/leeper/margins) -->
[![Build status](https://ci.appveyor.com/api/projects/status/t6nxndmvvcw3gw7f/branch/master?svg=true)](https://ci.appveyor.com/project/leeper/margins/branch/master)
[![codecov.io](http://app.codecov.io/github/leeper/margins/coverage.svg?branch=master)](https://app.codecov.io/github/leeper/margins?branch=master)
[![Project Status: Active - The project has reached a stable, usable state and is being actively developed.](http://www.repostatus.org/badges/latest/active.svg)](https://www.repostatus.org/)

The development version of this package can be installed directly from GitHub using `remotes`:

```R
if (!require("remotes")) {
    install.packages("remotes")
    library("remotes")
}
install_github("leeper/prediction")
install_github("leeper/margins")

# building vignettes takes a moment, so for a quicker install set:
install_github("leeper/margins", build_vignettes = FALSE)
```

