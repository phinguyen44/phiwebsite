---
title: Visualizing Decision Boundaries
author: Phi Nguyen
date: '2020-01-30'
slug: visualizing-decision-boundaries
categories: []
tags:
  - statistics
draft: no
---



_**Note**: I'm going to be doing a series of posts on interesting algorithms that I've either used in the past or intend to use in the future. Most of this stuff is fairly new to me or something I haven't explored in depth, so please correct me if I am entirely off base. I fully believe that trying to explain a concept engenders a deeper understanding of said concept, so this is mostly a training tool for me!_

## Introduction

In this post, I wanted to talk about **decision boundaries** for classification problems. Let's assume, for example, that we wanted to classify candidates applying for a job as either *GOOD* or *BAD* candidates. The decision boundary would be the boundary that partitions observations on one side of the decision boundary as *GOOD* candidates and observations on the other side as *BAD* candidates.

Mathematically speaking, this would be the point where:

$$ P(Y=1|X) = P(Y=0|X) $$

Given that different classifiers will estimate the conditional distribution \\(P(Y=y|X)\\) differently, they will all yield slightly different decision boundaries (some linear, some not). For the purposes of this post, I will focus on three different classifiers, one of which is purely theoretical, and two that must be estimated from sample data. These are:

1. Optimal Bayes classifier
2. Naive Bayes classifier
3. Logistic regression

## Data

Let's first set up the data to analyze these classifiers. I'll be using the following packages to work on this data set.


```r
library(tidyverse)
library(mvtnorm)
library(naivebayes)
library(ggplot2)
```

Going back to our prior example, let's assume that candidacy for a job is entirely dependent upon two variables, \\(X_1\\) and \\(X_2\\). That is, we are trying to model \\(P(Y=y|X_1,X_2)\\). Assume also, that \\(Y\\) comes from a Bernoulli distribution, meaning it can take either equal 1 (GOOD candidate) or 0 (BAD candidate). Assume finally that both of \\(X_1\\) and \\(X_2\\) are normally distributed, and that their joint distribution conditional on the class is also a multivariate normal.

That is,

$$ Y \sim Bernoulli(p) $$
$$ (X_1,X_2) | Y = 0 \sim N(\mu_0, \Sigma_0) $$
$$ (X_1,X_2) | Y = 1 \sim N(\mu_1, \Sigma_1) $$


```r
# Parameters to play with:
py0     <- 0.5
n       <- 500
mu0     <- c(3,5)
sigma0  <- matrix(c(1,-0.4,-0.4,1), nrow = 2, byrow = T)
mu1     <- c(5,3)
sigma1  <- matrix(c(1,0.2,0.2,1), nrow = 2, byrow = T)

# Generate sample data
py1 <- 1 - py0
n0  <- rbinom(1, n, py0)
n1  <- n - n0

sample0 <- rmvnorm(n0, mu0, sigma0)
sample1 <- rmvnorm(n1, mu1, sigma1)

# Combine
combined_sample <- rbind2(sample0, sample1) %>% 
  as.data.frame() %>% 
  mutate(class = c(rep(0, n0), rep(1, n1))) %>% 
  rename(
    X1 = V1,
    X2 = V2
  )
```


```r
theme_set(
  theme_minimal() +
	theme(panel.grid.minor = element_blank()) + 
  theme(plot.title = element_text(size=16))
)

# Sample data
ggplot(combined_sample) +
  geom_point(
    aes(x = X1, y = X2, color = factor(class), shape = factor(class)),
    alpha = 0.7
  ) +
  coord_fixed(ratio = 1) +
  scale_color_brewer(type = "div", palette = "Dark2") +
  scale_shape_manual(values = c(4, 18)) +
  scale_y_continuous(limits = c(0, 8)) +
  scale_x_continuous(limits = c(0, 8)) +
  labs(title = "Sample Data")
```

<img src="/blog/2020-01-30-visualizing-decision-boundaries_files/figure-html/sampleoutput-1.png" width="80%" style="display: block; margin: auto;" />

This graph shows a sampling of 500 observations that belong to each class, projected onto the \\(X_1\\), \\(X_2\\) plane. Obviously this is a pretty simplistic case, which probably rarely occurs in practice, but it's mostly an illustrative exercise.

## Optimal Bayes Classifier

First off, let's start with the **optimal Bayes classifier**. I'm sure we're all familiar with the standard Baye's rule:

$$ P(Y|X) = \frac{P(X|Y)*P(Y)}{P(X)}$$
The term \\(P(Y|X)\\) is known as the *posterior distribution*, \\(P(X|Y)\\) is known as the *likelihood*, \\(P(Y)\\) is known as the *prior distribution*, and \\(P(X)\\) is known as the *normalizer*.

We use this formula to calculate the decision boundary for the optimal Bayes classifier. Using our example, the decision boundary occurs when:

$$ P(X_1,X_2|Y=1)P(Y=1) = P(X_1,X_2|Y=0)P(Y=0) $$

For example, we would assign a new observation to class 1 if

$$ P(X_1,X_2|Y=1)P(Y=1) > P(X_1,X_2|Y=0)P(Y=0) $$ 

The optimal Bayes classifier is called "optimal" because it chooses the class that has greatest a posteriori probability of occurrence (*Maximum a Posteriori Estimation*, or *MAP*. In otherwords, it finds the most probable hypothesis given the training data. Note that it is optimal in the sense that it lowers the misclassification error (see [here](http://www.cs.wichita.edu/~sinha/teaching/fall14/cs697AB/slide/supervised_bayes_optimal.pdf)), not that it leads to the most societally-desirable solution. I'll go into this a bit later in a future post.

<!-- # -> y* = arg max y P(Y=y|X1,X2) = arg max y P(X1,X2|Y=y)P(Y=y) -->

Given that we know the posterior distribution in this case, we can draw the contour plot of the true posterior distribution around the sample data.


```r
# Generate PDF (likelihood) of data
x1_range <- seq(0, 8, by = 0.05)
x2_range <- seq(0, 8, by = 0.05)
combined_range <- expand.grid(X1 = x1_range, X2 = x2_range)
# conditional probability of (x1, x2) given y = 0
px_y0 <- dmvnorm(combined_range, mean = mu0, sigma = sigma0)
# conditional probability of (x1, x2) given y = 1
px_y1 <- dmvnorm(combined_range, mean = mu1, sigma = sigma1)
# Generate PDF (likelihood) of data
x1_range <- seq(0, 8, by = 0.05)
x2_range <- seq(0, 8, by = 0.05)
combined_range <- expand.grid(X1 = x1_range, X2 = x2_range)
# conditional probability of (x1, x2) given y = 0
px_y0 <- dmvnorm(combined_range, mean = mu0, sigma = sigma0)
# conditional probability of (x1, x2) given y = 1
px_y1 <- dmvnorm(combined_range, mean = mu1, sigma = sigma1)

# Predicted class (posterior)
py0_x         <- px_y0 * py0
py1_x         <- px_y1 * py1
optimal       <- py1_x - py0_x
predict_class <- ifelse(py0_x > py1_x, 0, 1)
predict_df <- data.frame(
  py0_x         = py0_x,
  py1_x         = py1_x,
  optimal       = optimal,
  predict_class = predict_class
)

# Combine to big DF
combined_result <- combined_range %>% 
  bind_cols(predict_df)

# Sample data + posterior dist
ggplot(combined_result) +
  geom_contour(aes(x = X1, y = X2, z = py0_x), color = "#1b9e77") +
  geom_contour(aes(x = X1, y = X2, z = py1_x), color = "#d95f02") +
  geom_point(
    data = combined_sample,
    aes(x = X1, y = X2, color = factor(class), shape = factor(class)),
    alpha = 0.4
  ) +
  coord_fixed(ratio = 1) +
  scale_color_brewer(type = "div", palette = "Dark2") +
  scale_shape_manual(values = c(4, 18)) +
  scale_y_continuous(limits = c(0, 8)) +
  scale_x_continuous(limits = c(0, 8)) +
  labs(title = "Sample Data") +
  labs(subtitle = "Overlaid with contour of actual posterior distribution") + 
  theme(plot.subtitle = element_text(size=10, color = "#7F7F7F"))
```

<img src="/blog/2020-01-30-visualizing-decision-boundaries_files/figure-html/plotcontour-1.png" width="672" />

From there, we can calculate the decision boundary, where the posterior distributions equal one another. Note that the optimal decision boundary is non-linear.


```r
ggplot(combined_result) +
  geom_point(
    data = combined_sample,
    aes(x = X1, y = X2, color = factor(class), shape = factor(class)),
    alpha = 0.7
  ) + 
  geom_contour(
    aes(x = X1, y = X2, z = optimal), 
    color    = "black",
    linetype = "dashed",
    breaks   = 0 # layer where optimal = 0
  ) +
  coord_fixed(ratio = 1) +
  scale_color_brewer(type = "div", palette = "Dark2") + 
  scale_shape_manual(values = c(4, 18)) + 
  scale_y_continuous(limits = c(0, 8)) + 
  scale_x_continuous(limits = c(0, 8)) + 
  labs(title = "Decision Boundary for Optimal Bayes Classifier")
```

<img src="/blog/2020-01-30-visualizing-decision-boundaries_files/figure-html/plotoptimal-1.png" width="672" />

## Naive Bayes

Obviously, in practice, we don't know the true posterior distribution, we have to use the empirical distribution from the sample data. The Naive Bayes classifier approximates the optimal Bayes classifier, once some clarifying assumptions are made. This classifier is an example of a **generative classifier**. Classifiers of this type are probabilistic models that try to describe the full joint probability distribution \\(P(Y,X)\\). In order to do so, these classifiers must generally model the distribution of the inputs (the constituent \\(X\\)'s) characteristic of a given class \\(Y\\).

In our case, our posterior distribution looks like this:

$$ P(Y|X_1,X_2)=\frac{P(X_1,X_2|Y)*P(Y)}{P(X_1,X_2)} $$
$$ P(X_1,X_2|Y)=P(X_1|Y)*P(X_2|Y,X_1) $$
The major assumption that enables the Naive Bayes (and also why it is called "Naive") is to assume that the features are conditionally independent given the class. That is:

$$ P(X_2|Y,X1) = P(X_2|Y) $$

Or more generally:

$$ P(X|Y=y) = \prod_{i=1}^{n} P(X_i|Y=y) $$ 

This simplifying assumption allows us to more easily calculate the decision boundary from the sample data.


```r
fit_nb       <- naive_bayes(
  formula = factor(class) ~ X1 + X2, 
  data    = combined_sample
)
predict_nb  <- predict(fit_nb, newdata = combined_range, type = "prob")
combined_nb <- combined_range %>% 
  bind_cols(data.frame(predict_class = predict_nb[,2])) %>% 
  mutate(optimal = predict_class - 0.5)

ggplot(combined_nb) +
  geom_point(
    data = combined_sample,
    aes(x = X1, y = X2, color = factor(class), shape = factor(class)),
    alpha = 0.7
  ) + 
  geom_contour(
    aes(x = X1, y = X2, z = optimal), 
    color    = "black",
    breaks   = 0 # layer where optimal = 0
  ) +
  coord_fixed(ratio = 1) +
  scale_color_brewer(type = "div", palette = "Dark2") + 
  scale_shape_manual(values = c(4, 18)) + 
  scale_y_continuous(limits = c(0, 8)) + 
  scale_x_continuous(limits = c(0, 8)) + 
  labs(title = "Decision Boundary for Naive Bayes")
```

<img src="/blog/2020-01-30-visualizing-decision-boundaries_files/figure-html/naivebayes-1.png" width="672" />

Note here that the decision boundary is non-linear, but would be linear if the posterior distributions were Gaussian. They technically are in this case, but since we're using sample data, the empirical distribution of the sample data may not be entirely normal, but will approach a normal if we were to increase the sample size.

## Logistic Regression

The last example I wanted to show was a *logistic regression classifier*, which is actually the one I'm most familiar with, coming from a statistics background. This classifier is an example of a **discriminative classifier**. These type of classifiers try to directly learn the parameters of the decision boundary or class, by modeling the conditional probability directly.

Logistic regression takes the form:

$$ P(Y=1|X) = \frac{1}{1+e^{-X\beta}} $$

Here the decision boundary is actually linear because we would predict GOOD if the resulting probability exceeded 0.5. 


```r
fit_lm       <- glm(
  formula = class ~ X1 + X2, 
  data    = combined_sample, 
  family  = binomial
)
predict_glm  <- predict(fit_lm, newdata = combined_range, type = "response")
combined_glm <- combined_range %>% 
  bind_cols(data.frame(predict_class = predict_glm)) %>% 
  mutate(optimal = predict_class - 0.5)

ggplot(combined_glm) +
  geom_point(
    data = combined_sample,
    aes(x = X1, y = X2, color = factor(class), shape = factor(class)),
    alpha = 0.7
  ) + 
  geom_contour(
    aes(x = X1, y = X2, z = optimal), 
    color    = "black",
    breaks   = 0 # layer where optimal = 0
  ) +
  coord_fixed(ratio = 1) +
  scale_color_brewer(type = "div", palette = "Dark2") + 
  scale_shape_manual(values = c(4, 18)) + 
  scale_y_continuous(limits = c(0, 8)) + 
  scale_x_continuous(limits = c(0, 8)) + 
  labs(title = "Decision Boundary for Logistic Regression")
```

<img src="/blog/2020-01-30-visualizing-decision-boundaries_files/figure-html/logistic-1.png" width="672" />

## Conclusion

Understanding decision boundaries, and how they would be generated for different estimators, is important in understanding classification problems in machine learning. 

Some things I didn't discuss, which I may at a future point, include how the decision boundary changes depending on the sample size or in the face of class imbalance. I also wanted to discuss these decision boundaries in the context of algorithmic fairness, but will also leave that for another time.

The code for this post can be found [here](https://github.com/phister/blog_posts/blob/master/DecisionBoundaries.R). Hope this all made sense!

## Useful Resources

1. https://mathformachines.com/posts/decision/
2. https://xavierbourretsicotte.github.io/Optimal_Bayes_Classifier.html
3. https://homes.cs.washington.edu/~marcotcr/blog/linear-classifiers/
4. http://pages.cs.wisc.edu/~jerryzhu/cs769/nb.pdf
