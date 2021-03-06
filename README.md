# Instruction to estimate median deductively

Constantine Frangakis, Tianchen Qian, Zhenke Wu, Ivan Diaz

Friday, November 21, 2014

This is a walk-through of using the code `fcn_median_estimation.R` to deductively estimate the median outcome of a treatment arm of an ignorable assignment design (see paper "Deductive derivation and Turing-computerization of semiparametric efficient estimation" by Constantine Frangakis, Tianchen Qian, Zhenke Wu, and Ivan Diaz), and to obtain an estimated standard error.

To run this example, you may clone the entire repository, and open the R file Instruction_code.R and follow/run the steps below.

Also, you can download "Instruction.html" and open it using a web browser to see the code and the expected output from R. 

## Data Preparation

We use `data_example_252pts.csv` as the data example for demonstration, and we are interested in estimating median outcome of a specific treatment, under double sampling design. The CSV file contains 252 rows, i.e. 252 observations. This data set has several columns of baseline varialbe $X$, a column of outcome of interest $Y$, a column of missingness indicator $R$ ($R = 1$ if and only if $Y$ is observed), and a column of estimated propensity score $e$ ($e(X)=P\{R=1|X\}$).

```{r}
# load data and source code
library(bootstrap) # for jackknife standard error
dt <- read.csv("data_example_252pts.csv")
source("fcn_median_estimation.R")
names(dt)
```

In this dataset, column 1 - 25 are baseline variables; column 26 `r_vec` is missingness indicator; column 27 `y_vec` is outcome of interest; column 28 `e_vec` is the estimated propensity score. Note that the estimated propensity score needs to be specified by the user (either using logistic regression, or some fancier techniques), before calling the main function.

## Specify arguments to be passed into the main function

The main function for median estimation is `deduct_est_median()`. We first specify the necessary components to be passed into this function.

```{r}
# Specify baseline covariate data frame
x_df <- subset(dt, select = -c(r_vec, y_vec, e_vec))

# Estimate working model of expected Y given X
# Here we use linear regression, using all the patients with Y observed
terms <- paste(colnames(x_df), collapse = "+")
formula <- as.formula(paste("y_vec~", terms))  # just an easier way to throw 
                                               # all the X into the regression model
print(formula)
fit_glm <- lm(formula, data = subset(dt, r_vec == 1))

# Specify the initial working model coefficients
beta_init <- coefficients(fit_glm)

print(beta_init)

# Specify the form of distribution of Y given X
# Here we assume Y|X follows a normal distribution with mean X*beta and standard error 1.
py_vec_fcn = function(t, x, beta){
  pnorm(t, mean = x%*%beta[-1] + beta[1], sd = 1) # beta[1] is intercept
}
```

`py_vec_fcn` needs to be a function that takes in arguments: 1) scalar $t$; 2) $n \times m$ matrix $X$; 3) regression coefficient $\beta$. And it returns a vector of $P\{Y \leq t | X, \beta\}$, which is of length $n$. The probability distribution function of $Y$ given $X$ is determined by the user. Here we assume $Y|X \sim N(X\beta,1)$.

## Let's estimate median!

Note that, in doing the estimation, the function will first automatically delete the possible `NA`'s in `beta_init`, which results from linearly dependent columns in `x_df` in this example - and also delete corresponding redundant columns in `x_df`. By default, the function will free-up the intercept in `beta_init`.

```{r, cache = TRUE}
est <- deduct_est_median(x_df, dt$r_vec, dt$y_vec,
                         dt$e_vec, py_vec_fcn,
                         beta_init)
print(est)
```

To free-up other coefficient in `beta_init`, the user can specify `free_idx` argument with the desired variable name whose coefficient is intended to be freed-up.

```{r, cache = TRUE}
deduct_est_median(x_df, dt$r_vec, dt$y_vec,
                dt$e_vec, py_vec_fcn,
                beta_init, free_idx = "sf36_phy_scr") # sf36_phy_scr is continuous

deduct_est_median(x_df, dt$r_vec, dt$y_vec,
                dt$e_vec, py_vec_fcn,
                beta_init, free_idx = "female") # female is binary
```

We can ask the function to give jackknife standard error of the estimate, by specifying `jack_se = TRUE` in the arguments. This might take a while.

```{r, cache = TRUE}
est_w_se <- deduct_est_median(x_df, dt$r_vec, dt$y_vec,
                              dt$e_vec, py_vec_fcn,
                              beta_init, jack_se = TRUE)
print(est_w_se)
```


One can also use `data_example_1260pts.csv`, which consists 1260 observations. This will take longer time.

```{r, include=FALSE}
   # add this chunk to end of mycode.rmd
   file.rename(from="Instruction.rmd", 
               to="README2.md")
```