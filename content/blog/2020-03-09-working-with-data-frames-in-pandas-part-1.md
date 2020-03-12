---
title: Working with Data Frames in Pandas, Part 1
author: 'Phi Nguyen'
date: '2020-03-09'
slug: working-with-data-frames-in-pandas-part-1
categories: []
tags:
  - statistics
draft: no
---

In the past few months, I've been working on beefing up my data wrangling skills with Python and the `pandas` library. Coming primarily from an R background, I'm a huge advocate of the `tidyverse` family of packages (namely, `dplyr` for wrangling, `tidyr` for tidy data sets, and `ggplot2` for plotting), from the great and powerful [Hadley Wickham](https://en.wikipedia.org/wiki/Hadley_Wickham). For me, these packages heavily simplify data manipulation by providing a common "grammar" for which to operate on data, and for enabling method chaining using the pipe operator (`%>%`).

As a result from using predominantly R for the better part of the last decade, switching over to `pandas` has been a bit bumpy. Some of the key verbs and conventions are different (and even counterintuitive, in my opinion), method chaining is slightly more verbose and less readable, and the different data structures take some time getting used to.

Therefore, in this post, I wanted to go over some of the functionality in `pandas` and map them back to their `tidyverse` equivalents. I'll tackle some of the basic verbs, then go in depth with more complex data wrangling challenges. In Part 2, I'll try to recreate some basic `ggplot2` graphs using `matplotlib` in Python.

## The Basic Verbs

`dplyr` is organized around a few key verbs corresponding to actions you may want to execute on the data set:

1. `select()`: subset columns (variables)
2. `filter()`: subset rows (observations)
3. `mutate()`: edit or create new variables
4. `summarize()`: reduce multiple values into a single summary
5. `arrange()`: order the data frame

All those prior verbs can be extended with `group_by()` which allows you to perform any operation “by group”.

### Subset Variables (Columns)

In R:

```r
df %>% select(var1)                   # select var1
df %>% select(-var1)                  # drop var1
df %>% select(contains("str"))        # select columns containing "str"
df %>% select(starts_with("str"))     # select columns that start with "str"
df %>% select_if(is.numeric)          # select based on a condition
```

Here I'm already starting to use the pipe operator. It's a handy way to chain commands in a linear fashion. It also helps emphasize the framework that *everything starts with the data frame*. Note that `select(df, var1)` is equivalent to `df %>% select(var1)`. `dplyr` also makes use of ["non-standard evaluation"](https://dplyr.tidyverse.org/articles/programming.html), which basically means you can pass arguments as expressions rather than just as values. It's a bit confusing to grasp at first, but once you get the hang of it, it saves a LOT of typing time (no need to add quotes around all the objects).

In Python:

```python
df[["var1"]]
df.drop(['var1'], axis = 1)
df.filter(regex = "^str")
df.filter(regex = "str")
df.select_dtypes(include='float64')
```

I very much dislike how Python's verb for subsetting columns is known as 'filter'. To me, that seems *sooo* counterintuitive.

### Subset Observations (Rows)

In R:

```r
df %>% filter(var1 > 100)                  # keep only rows where var1 > 100
df %>% filter(var2 == "str")               # keep only rows where var2 == "str"
df %>% filter(var1 > 100 & var2 == "str")  # keep only rows that satisfy both
df %>% distinct()                          # keep only distinct rows
df %>% slice(rows = 100:150)               # keep only rows 100 to 150
df %>% drop_na()                           # drop all rows containing NA vals
df %>% sample_frac(0.1)                    # randomly keep 10% of rows
df %>% sample_n(100)                       # randomly keep 100 rows
```

In Python:

```python
df[df["var1"] > 100]
df[df["var2"] == "string"]
df[(df["var1"] > 100 & df["var2"] == "string")]
df.drop_duplicates()
df.dropna()
df.sample(frac = 0.1)
df.sample(n = 100)
```

For subsetting in `pandas`, you could also use `query()`. For example, `df[df["var1"] > 100]` is equivalent to `df.query("var1 > 100")`.

### Edit or Create New Variables

In R:

```r
df %>% mutate(newvar = var1 + var2)  # create newvar1
df %>% rename(newvar = oldvar)       # rename "oldvar" as "newvar"
```

In Python:

```python
df["newvar"] = df["var1"] + df["var2"]
df.rename({"oldvar":"newvar"})
```

You can also use `assign()` in Python to create new variables. For example, `df["newvar"] = df["var1"] + df["var2"]` is equivalent to `df = df.assign(newvar = df["var1"] + df["var2"])`. Note here that `assign()` does not modify the object, so you will have to re-assign the updated data frame to the old data frame object.

With renaming variables, the convention for renaming is inverted. That is, in R, the new variable name is on the left hand side, while in Python the new variable name is on the right hand side. Another one of those annoying oddities that you just have to get used to.

Additionally, often times you might want to update not just a single variable, but a series of variables. For example, let's say you wanted to apply a function to all numeric variables. In R, this can be accomplished with a single line of code.

```R
df %>% mutate_if(is.numeric, ~ .^2)
```

In `pandas`, this is slightly more involved. Another one in the win column for the `tidyverse`!

```python
for column in df.columns:
  if df[column].dtype == "float64":
    df[column] = df[column] ** 2
```

### Summarize the Data

Summarize works best in conjunction with `group_by()`. Note that if you use `mutate()` instead of `summarize()` in R, you create a data frame with the same number of observations as the original data frame. This can be incredibly useful for window functions (things like cumulative sum, rank, and row number). In Python, what's great is that you don't need to change anything. If you use a window function (e.g. `cumsum()`), Python "knows" that you are retaining the same number of rows.

In R:

```r
# calculate the mean of var1 by grouped vars group1 and group2
df %>% group_by(group1, group2) %>% summarize(meanvar1 = mean(var1))
# get counts
df %>% group_by(group1, group2) %>% count()
# remove grouped attributes
df %>% ungroup()
# window function
df %>% group_by(group1) %>% mutate(cumulative_sum = cumsum(var1))

```

In Python:

```python
df.groupby(["group1", "group2"])["var1"].mean()
df.groupby(["group1", "group2"]).size()
df.reset_index()
df.groupby(["group1"])["var1"].cumsum()
```

### Order the Data Frame

In R:

```r
df %>% arrange("var1")        # Arrange ascending by var1
df %>% arrange(desc("var1"))  # Arrange descending by var1
```

In Python:

```python
df.sort_values("var1")
df.sort_values("var1", ascending = False)
```

## Chaining Verbs Together

All of the prior verbs can be chained together using the pipe operator in R (`%>%`) to do more complex wrangling. In Python, you can do chaining by combining methods, but it gets a bit hard to read if you're a lot of methods. However, there is a workaround using parentheses.

In R:

```r
# For full time jobs in CA, NY, find mean+max wages by position, then sort
df %>%
  drop_na() %>%
  filter(job_type == "full_time" & state %in% c('California', 'New York'))
  group_by(state, position) %>%
  summarize(
    mean_wage = mean(wage),
    max_wage  = max(wage)
    ) %>%
  arrange(desc(mean_wage)) %>%
  ungroup()
```

In Python:

```python
(df
  .dropna()
  .query("job_type == 'full_time' & state in ['California', 'New York]")
  .groupby(["state", "position"])["wage"]
  .agg(["mean", "max"])
  .sort_values('mean', ascending = False)
  .reset_index()
)
```

## Joining Data

The last thing I'll cover is joining data. In `dplyr` this is accomplished with specific verbs for the type of join you want to achieve.

In R: 

```r
left_join(df1, df2, by = c("var1", "var2"))
left_join(df1, df2, by = c("df1_var1" = "df2_var1")) # if different var names
right_join(df1, df2, by = c("var1", "var2"))
inner_join(df1, df2, by = c("var1", "var2"))
full_join(df1, df2, by = c("var1", "var2"))
bind_rows(df1, df2)
bind_cols(df1, df2)
```

In Python:

```python
pd.merge(df1, df2, how = 'left', on = ["var1", "var2"])
pd.merge(df1, df2, how = 'left', left_on = "df1_var1", right_on = "df2_var1")
pd.merge(df1, df2, how = 'right', on = ["var1", "var2"])
pd.merge(df1, df2, how = 'inner', on = ["var1", "var2"])
pd.merge(df1, df2, how = 'outer', on = ["var1", "var2"])
df1.append(df2)
pd.concat([df1, df2], axis = 1)
```

## Conclusion

That's all for now. I hope that gives you R folks some help when learning data wrangling in Python. I'm still an R guy through and through, but I gotta stay ahead of the data science curve and keep my Python skills up to snuff. Next time I'll show how to recreate some `ggplot2` graphics in Python. See you then!
