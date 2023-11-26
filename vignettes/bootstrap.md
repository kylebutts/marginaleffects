
# Bootstrap & Simulation

`marginaleffects` offers an `inferences()` function to compute
uncertainty estimates using the bootstrap and simulation-based
inference.

WARNING: The `inferences()` function is experimental. It may be renamed,
the user interface may change, or the functionality may migrate to
arguments in other `marginaleffects` functions.

Consider a simple model:

``` r
library(marginaleffects)

mod <- lm(Sepal.Length ~ Petal.Width * Petal.Length + factor(Species), data = iris)
```

We will compute uncertainty estimates around the output of
`comparisons()`, but note that the same approach works with the
`predictions()` and `slopes()` functions as well.

## Delta method

The default strategy to compute standard errors and confidence intervals
is the delta method. This is what we obtain by calling:

``` r
avg_comparisons(mod, by = "Species", variables = "Petal.Width")
#> 
#>         Term Contrast    Species Estimate Std. Error      z Pr(>|z|)   S  2.5 % 97.5 %
#>  Petal.Width mean(+1) setosa      -0.1103      0.285 -0.387    0.699 0.5 -0.669  0.449
#>  Petal.Width mean(+1) versicolor  -0.0201      0.160 -0.125    0.900 0.2 -0.334  0.293
#>  Petal.Width mean(+1) virginica    0.0216      0.169  0.128    0.898 0.2 -0.309  0.353
#> 
#> Columns: term, contrast, Species, estimate, std.error, statistic, p.value, s.value, conf.low, conf.high, predicted_lo, predicted_hi, predicted 
#> Type:  response
```

Since this is the default method, we obtain the same results if we add
the `inferences()` call in the chain:

``` r
avg_comparisons(mod, by = "Species", variables = "Petal.Width") |>
  inferences(method = "delta")
#> 
#>         Term Contrast    Species Estimate Std. Error      z Pr(>|z|)   S  2.5 % 97.5 %
#>  Petal.Width mean(+1) setosa      -0.1103      0.285 -0.387    0.699 0.5 -0.669  0.449
#>  Petal.Width mean(+1) versicolor  -0.0201      0.160 -0.125    0.900 0.2 -0.334  0.293
#>  Petal.Width mean(+1) virginica    0.0216      0.169  0.128    0.898 0.2 -0.309  0.353
#> 
#> Columns: term, contrast, Species, estimate, std.error, statistic, p.value, s.value, conf.low, conf.high, predicted_lo, predicted_hi, predicted 
#> Type:  response
```

## Bootstrap

`marginaleffects` supports three bootstrap frameworks in `R`: the
well-established `boot` package, the newer `rsample` package, and the
so-called “bayesian bootstrap” in `fwb`.

### `boot`

``` r
avg_comparisons(mod, by = "Species", variables = "Petal.Width") |>
  inferences(method = "boot")
#> 
#>         Term Contrast    Species Estimate Std. Error  2.5 % 97.5 %
#>  Petal.Width mean(+1) setosa      -0.1103      0.265 -0.644  0.400
#>  Petal.Width mean(+1) versicolor  -0.0201      0.156 -0.332  0.289
#>  Petal.Width mean(+1) virginica    0.0216      0.181 -0.331  0.370
#> 
#> Columns: term, contrast, Species, estimate, predicted_lo, predicted_hi, predicted, std.error, conf.low, conf.high 
#> Type:  response
```

All unknown arguments that we feed to `inferences()` are pushed forward
to `boot::boot()`:

``` r
est <- avg_comparisons(mod, by = "Species", variables = "Petal.Width") |>
  inferences(method = "boot", sim = "balanced", R = 500, conf_type = "bca")
est
#> 
#>         Term Contrast    Species Estimate Std. Error  2.5 % 97.5 %
#>  Petal.Width mean(+1) setosa      -0.1103      0.273 -0.580  0.422
#>  Petal.Width mean(+1) versicolor  -0.0201      0.171 -0.361  0.320
#>  Petal.Width mean(+1) virginica    0.0216      0.191 -0.321  0.408
#> 
#> Columns: term, contrast, Species, estimate, predicted_lo, predicted_hi, predicted, std.error, conf.low, conf.high 
#> Type:  response
```

We can extract the original `boot` object from an attribute:

``` r
attr(est, "inferences")
#> 
#> BALANCED BOOTSTRAP
#> 
#> 
#> Call:
#> bootstrap_boot(model = model, INF_FUN = INF_FUN, newdata = ..1, 
#>     vcov = ..2, variables = ..3, type = ..4, by = ..5, conf_level = ..6, 
#>     cross = ..7, comparison = ..8, transform = ..9, wts = ..10, 
#>     hypothesis = ..11, eps = ..12)
#> 
#> 
#> Bootstrap Statistics :
#>        original      bias    std. error
#> t1* -0.11025325 0.013461366   0.2730309
#> t2* -0.02006005 0.004991527   0.1714549
#> t3*  0.02158742 0.001080508   0.1907931
```

Or we can extract the individual draws with the `posterior_draws()`
function:

``` r
posterior_draws(est) |> head()
#>   drawid        draw        term contrast    Species    estimate predicted_lo predicted_hi predicted std.error   conf.low conf.high
#> 1      1  0.01562283 Petal.Width mean(+1)     setosa -0.11025325     4.957514     4.845263  4.957514 0.2730309 -0.5799103 0.4222341
#> 2      1 -0.02540313 Petal.Width mean(+1) versicolor -0.02006005     6.327949     6.322072  6.327949 0.1714549 -0.3605356 0.3196208
#> 3      1 -0.04434720 Petal.Width mean(+1)  virginica  0.02158742     7.015513     7.051542  7.015513 0.1907931 -0.3210064 0.4080948
#> 4      2  0.02506140 Petal.Width mean(+1)     setosa -0.11025325     4.957514     4.845263  4.957514 0.2730309 -0.5799103 0.4222341
#> 5      2  0.20196750 Petal.Width mean(+1) versicolor -0.02006005     6.327949     6.322072  6.327949 0.1714549 -0.3605356 0.3196208
#> 6      2  0.28365537 Petal.Width mean(+1)  virginica  0.02158742     7.015513     7.051542  7.015513 0.1907931 -0.3210064 0.4080948

posterior_draws(est, shape = "DxP") |> dim()
#> [1] 500   3
```

### `rsample`

As before, we can pass arguments to `rsample::bootstraps()` through
`inferences()`. For example, for stratified resampling:

``` r
est <- avg_comparisons(mod, by = "Species", variables = "Petal.Width") |>
  inferences(method = "rsample", R = 100, strata = "Species")
est
#> 
#>         Term Contrast    Species Estimate  2.5 % 97.5 %
#>  Petal.Width mean(+1) setosa      -0.1103 -0.634  0.405
#>  Petal.Width mean(+1) versicolor  -0.0201 -0.318  0.234
#>  Petal.Width mean(+1) virginica    0.0216 -0.287  0.287
#> 
#> Columns: term, contrast, Species, estimate, predicted_lo, predicted_hi, predicted, conf.low, conf.high 
#> Type:  response

attr(est, "inferences")
#> # Bootstrap sampling using stratification with apparent sample 
#> # A tibble: 101 × 3
#>    splits           id           estimates       
#>    <list>           <chr>        <list>          
#>  1 <split [150/50]> Bootstrap001 <tibble [3 × 7]>
#>  2 <split [150/56]> Bootstrap002 <tibble [3 × 7]>
#>  3 <split [150/57]> Bootstrap003 <tibble [3 × 7]>
#>  4 <split [150/51]> Bootstrap004 <tibble [3 × 7]>
#>  5 <split [150/59]> Bootstrap005 <tibble [3 × 7]>
#>  6 <split [150/58]> Bootstrap006 <tibble [3 × 7]>
#>  7 <split [150/48]> Bootstrap007 <tibble [3 × 7]>
#>  8 <split [150/59]> Bootstrap008 <tibble [3 × 7]>
#>  9 <split [150/54]> Bootstrap009 <tibble [3 × 7]>
#> 10 <split [150/56]> Bootstrap010 <tibble [3 × 7]>
#> # ℹ 91 more rows
```

Or we can extract the individual draws with the `posterior_draws()`
function:

``` r
posterior_draws(est) |> head()
#>   drawid        draw        term contrast    Species    estimate predicted_lo predicted_hi predicted   conf.low conf.high
#> 1      1 -0.08670198 Petal.Width mean(+1)     setosa -0.11025325     4.957514     4.845263  4.957514 -0.6339015 0.4048301
#> 2      1 -0.06447267 Petal.Width mean(+1) versicolor -0.02006005     6.327949     6.322072  6.327949 -0.3177457 0.2339422
#> 3      1 -0.05420809 Petal.Width mean(+1)  virginica  0.02158742     7.015513     7.051542  7.015513 -0.2869199 0.2865927
#> 4      2 -0.08363772 Petal.Width mean(+1)     setosa -0.11025325     4.957514     4.845263  4.957514 -0.6339015 0.4048301
#> 5      2 -0.17608698 Petal.Width mean(+1) versicolor -0.02006005     6.327949     6.322072  6.327949 -0.3177457 0.2339422
#> 6      2 -0.21877621 Petal.Width mean(+1)  virginica  0.02158742     7.015513     7.051542  7.015513 -0.2869199 0.2865927

posterior_draws(est, shape = "PxD") |> dim()
#> [1]   3 100
```

### Fractional Weighted Bootstrap (aka Bayesian Bootstrap)

[The `fwb` package](https://ngreifer.github.io/fwb/) implements
fractional weighted bootstrap (aka Bayesian bootstrap):

> “fwb implements the fractional weighted bootstrap (FWB), also known as
> the Bayesian bootstrap, following the treatment by Xu et al. (2020).
> The FWB involves generating sets of weights from a uniform Dirichlet
> distribution to be used in estimating statistics of interest, which
> yields a posterior distribution that can be interpreted in the same
> way the traditional (resampling-based) bootstrap distribution can be.”
> -Noah Greifer

The `inferences()` function makes it easy to apply this inference
strategy to `marginaleffects` objects:

``` r
avg_comparisons(mod) |> inferences(method = "fwb")
#> 
#>          Term            Contrast Estimate Std. Error  2.5 % 97.5 %
#>  Petal.Length +1                    0.8929     0.0768  0.746  1.046
#>  Petal.Width  +1                   -0.0362     0.1627 -0.345  0.278
#>  Species      versicolor - setosa  -1.4629     0.3289 -2.126 -0.854
#>  Species      virginica - setosa   -1.9842     0.3982 -2.812 -1.243
#> 
#> Columns: term, contrast, estimate, std.error, conf.low, conf.high 
#> Type:  response
```

## Simulation-based inference

This simulation-based strategy to compute confidence intervals was
described in Krinsky & Robb (1986) and popularized by King, Tomz,
Wittenberg (2000). We proceed in 3 steps:

1.  Draw `R` sets of simulated coefficients from a multivariate normal
    distribution with mean equal to the original model’s estimated
    coefficients and variance equal to the model’s variance-covariance
    matrix (classical, “HC3”, or other).
2.  Use the `R` sets of coefficients to compute `R` sets of estimands:
    predictions, comparisons, or slopes.
3.  Take quantiles of the resulting distribution of estimands to obtain
    a confidence interval and the standard deviation of simulated
    estimates to estimate the standard error.

Here are a few examples:

``` r
library(ggplot2)
library(ggdist)

avg_comparisons(mod, by = "Species", variables = "Petal.Width") |>
  inferences(method = "simulation")
#> 
#>         Term Contrast    Species Estimate Std. Error  2.5 % 97.5 %
#>  Petal.Width mean(+1) setosa      -0.1103      0.282 -0.663  0.461
#>  Petal.Width mean(+1) versicolor  -0.0201      0.164 -0.333  0.330
#>  Petal.Width mean(+1) virginica    0.0216      0.171 -0.321  0.352
#> 
#> Columns: term, contrast, Species, estimate, std.error, conf.low, conf.high, predicted_lo, predicted_hi, predicted, tmp_idx 
#> Type:  response
```

Since simulation based inference generates `R` estimates of the
quantities of interest, we can treat them similarly to draws from the
posterior distribution in bayesian models. For example, we can extract
draws using the `posterior_draws()` function, and plot their
distributions using packages like`ggplot2` and `ggdist`:

``` r
avg_comparisons(mod, by = "Species", variables = "Petal.Width") |>
  inferences(method = "simulation") |>
  posterior_draws("rvar") |>
  ggplot(aes(y = Species, xdist = rvar)) +
  stat_slabinterval()
```

<img
src="../bootstrap.markdown_strict_files/figure-markdown_strict/unnamed-chunk-13-1.png"
style="width:100.0%" />

## Multiple imputation and missing data

The same workflow and the same `inferences` function can be used to
estimate models with multiple imputation for missing data.
