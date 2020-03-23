---
title: Resampling Methods
author: Phi nguyen
date: '2020-03-05'
slug: resampling-methods
categories: []
tags:
  - statistics
draft: no
---



Suppose you are asked the following question:

*"You have a sequence \\(X = {X_1, ..., X_N}\\) of independently and identically distributed random variables drawn from the same unknown distribution. How would you estimate the variance of the sample median?"*

This question was posed to me in a recent data science interview. At the time I didn't know the answer because I was pretty unfamiliar with computational statistics. But now, as a form of self-atonement, I want to answer this question in detail.

In order to solve this, we can actually use **resampling methods**, such as the bootstrap or jackknife estimator, to derive reliable estimates of tricky statistics such as the median. In this post, I'll go into how these methods work.

## Motivation

Often times, we are interested in estimating some unknown population parameter \\(\theta\\) from a set of sample data. Estimating this parameter is fairly straightforward, usually by calculating the sampling statistic of interest (in our case, the sample median). However, it is perhaps even more important to know the accuracy or the sampling statistic \\(\hat{\theta}\\), or how much variability we expect to see in the estimation of that parameter.

So how do we get a measure of this variability? Given that our sample is assumed to be drawn randomly, our estimate \\(\hat{\theta}\\) is also a random variable. This means that, if we were to collect a new sample, we would get a different estimate of \\(\hat{\theta}\\). Therefore, our estimate has a *sampling distribution* that describes all the possible values the estimate can take on, given that they are drawn from equally-sized samples from the same population.

In simple cases, such as for the sample mean, the sampling distribution is fairly well-understood and can be derived by using the *central limit theorem*. But for trickier cases, such as the sample median, it gets a little bit murkier because we have no knowledge of the theoretical form of the estimator's standard error. So how do we get a measure of this variability in trickier cases?

Enter **resampling**. 

Resampling allows us to approximate the sampling distribution of just about any statistic by using random sampling (bootstrap) or by using subsets of available data (jackknife). From there, we can construct confidence intervals and standard errors for these statistics to generate this measure of variability.

## Bootstrap

The first method I'll go into is the **bootstrap** method. Bootstrapping is a resampling technique whereby a pre-determined number of random samples (with replacement) are generated from the sample data set, each of size \\(n\\), where \\(n\\) is the size of the data set. From here, inference about the population can be modeled from this resampling of the sample data, provided that the sample accurately reflects the population from which it is drawn. After generating a large enough number of samples (say, \\(B\\)), we can generate a histogram, or calculate the sample variance, that would provide an estimate of the shape of the distribution of the sample statistic.

Let \\(\hat{\theta}_i\\) be equal to the sample median for one resample of the sample data set. Our estimate of the sample median could be represented as \\(\bar{\theta}\\), or the mean of the sample median for all \\(i \in 1, ... , B\\):

$$ \bar\theta=\frac{1}{B}\sum_{i=1}^B\hat\theta_i $$

The variance of the sample median can be calculated as followed:

$$ \hat{V} =\frac{1}{B-1}\sum_{i=1}^B(\hat\theta_i-\bar\theta)^2 $$

Note that the bootstrap method can be shown to be a consistent, unbiased estimator (of the statistic from which the samples were drawn).

## Jackknife

The jackknife method is another resampling technique, but instead of sampling with replacement, the technique samples without replacement. The technique is based on sequentially deleting one observation from the data set and recomputing the estimator \\(n\\) times. Since one observation is deleted each time, there are \\(n\\) subsamples of size \\(n-1\\).

Just like in the case of the bootstrap estimator, the estimate of the sample median would be: 

$$ \bar\theta=\frac{1}{B}\sum_{i=1}^B\hat\theta_i $$

It should be fairly evident that the estimator is *biased* given the leave-one-out nature. The bias of the jackknife estimator can be represented as:

$$ \hat{Bias}{(\theta)}=(n-1)(\bar\theta-\hat\theta) $$

We can correct for the bias by subtracting this bias from our median \\(\hat{\theta}\\):

$$ \bar\theta_{jackknife}=\hat\theta-\hat{Bias}{(\theta)}=n\hat\theta-(n-1)\bar\theta $$

And the variance (note that the \\(n-1\\) term in the numerator ensures the unbiasedness of the estimator of variance, see [here](http://people.bu.edu/aimcinto/jackknife.pdf) for more details):

$$ \hat{V}=\frac{n-1}{n}\sum_{i=1}^n (\hat\theta_i-\bar\theta)^2 $$

## Example

Let's now use both the bootstrap and jackknife methods to answer the question illuminated above. Let's start with a slightly non-trivial example. I'll start by generating a random sample of length 1000 drawn from a [log-normal distribution](https://en.wikipedia.org/wiki/Log-normal_distribution). 


```r
n     <- 1000
x     <- exp(rnorm(n))
x_df  <- data.frame(x = x)
theta <- median(x)
```

<img src="/blog/2020-03-05-resampling-methods_files/figure-html/graph-1.png" width="672" />

For the bootstrap method, one element that I haven't touched on much yet is the choice of B. Let's try a few different values.


```r
B  <- c(10, 100, 1000, 10000)

run_bootstrap <- function(B) {
  bs <- numeric(length = B)
  for (i in 1:B) {
    samp    <- sample(x, size = n, replace = T)
    theta_i <- median(samp)
    bs[i]   <- theta_i
  }
  
  theta_bar_bs <- mean(bs)
  var_bs       <- (1/(B-1))*sum((bs - theta_bar_bs)^2)
  
  # confidence interval (95%)
  ub_bs <- theta_bar_bs + 1.96*sqrt(var_bs)
  lb_bs <- theta_bar_bs - 1.96*sqrt(var_bs)
  
  df <- data.frame(
    theta_bar = theta_bar_bs,
    var       = var_bs,
    lb        = lb_bs,
    ub        = ub_bs
  )
  return(df)
}

res        <- map_df(B, run_bootstrap)
res$method <- paste0('bootstrap_', B)

res
```

```
##   theta_bar         var        lb       ub          method
## 1 0.9860688 0.003540136 0.8694507 1.102687    bootstrap_10
## 2 0.9844426 0.003160964 0.8742466 1.094639   bootstrap_100
## 3 0.9947406 0.002706489 0.8927737 1.096708  bootstrap_1000
## 4 0.9918430 0.002749921 0.8890612 1.094625 bootstrap_10000
```

As expected, the variance decreases as B increases. \\(\hat{\theta}\\) also becomes closer to the true value of \\(\theta\\). However, even as the size of B increases exponentially, the improvements in variance reduction are minimal. So long as the size of the initial sample is sufficiently large, one can achieve decent estimates even with small sizes of B.

Let's look at the jackknife method. Note here, I apply the bias adjustment.


```r
jk <- numeric(length = n)

for (i in 1:n) {
  theta_i   <- median(x[-i])
  jk[i]     <- theta_i
}

theta_bar_jk <- mean(jk)
var_jk       <- ((n-1)/n)*sum((jk - theta_bar_jk)^2)

# bias
bias_jk <- (n - 1)*(theta_bar_jk - theta)

# adjust theta_bar
theta_bar_jk_adjust <- theta_bar_jk - bias_jk

# confidence interval (95%)
ub_jk <- theta_bar_jk + 1.96*sqrt(var_jk)
lb_jk <- theta_bar_jk - 1.96*sqrt(var_jk)

# confidence interval bias-adjusted (95%)
ub_jk_adjust <- theta_bar_jk_adjust + 1.96*sqrt(var_jk)
lb_jk_adjust <- theta_bar_jk_adjust - 1.96*sqrt(var_jk)

res_jk <- data.frame(
  theta_bar = theta_bar_jk_adjust,
  var       = var_jk,
  lb        = lb_jk_adjust,
  ub        = ub_jk_adjust,
  method    = 'jackknife'
)

res <- bind_rows(res, res_jk)
orderr <- res$method
res$method <- factor(res$method, levels = orderr)
```

Let's plot all of those confidence intervals together (the different bootstrapping samples as well as the jackknife):

<img src="/blog/2020-03-05-resampling-methods_files/figure-html/graphresults-1.png" width="672" />

Here you can see that the bias-adjusted jackknife estimate will approximate the sample median best, and have the smallest variance. This is due to the leave-one-out nature of the technique, such that each resample will already be fairly close to the sample statistic.

## Conclusion

Resampling methods like the jackknife and the bootstrap are incredibly intuitive and simple solutions to derive the estimates of standard errors and confidence intervals for a wide range of complex estimators.

Resampling methods have uses outside of parametric estimation as well - they can also be extended for uses such as in hypothesis testing or in regression. You can perhaps already see parallels between the jackknife method and cross-validation in machine learning (the jackknife can also be generalized in such a way to increase the size of the number of elements left-out).

So now you can see why I said the answer to the question is fairly obvious! I'll get it next time.
