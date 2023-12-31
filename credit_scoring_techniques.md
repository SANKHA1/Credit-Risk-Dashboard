This work is to illustrate some useful techniques in Credit Scoring Modelling, namely:

1. Data Preparation and Exploration at Univariate Level with rank plots for continuous variables, and the Information Value Analysis to do variable transformation and selection
2. Multivariate Analysis with several feature engineering along the analytics 
3. **Synthetic Minority Over-sampling Techique (SMOTE)** for the unbalance data
4. **Standard Logistic Regression** with Step-wise selection
5. **Tree-based Recursive Partitioning** 
6. **(Unbiased Non-parametric Model-Based (MOB)** with two-layer of recursive partitioning, and logistic regression at terminal node as the hybrid of (4) and (5)
7. **Chi-squared Automation Interaction Detection** as another (and very interesting) approach of tree-based algorithms, comparing to the popular Recursive Partitioning CART or Regression Tree. The algorithms is designed to be more analytical-oriented than predict-oriented. 

**Practical Meanings**:

The purpose of this work is to briefly overview some practical and handy techniques which are applicable for credit scoring. Rather than the thorough calibriating and validating the model to obtain higher accuracy, this work is focusing on walk-through and discuss around the practical applications of these techniques.

* Data Exploration and Preparation would be mentioned
* `Logistic Regression` is covered as this is one of the most conventional approach in Credit Scoring
* However, the limitation of Logit is its dependencies in lineary and monotone relationship, and not efficient in capturining the non-linearity and interaction among features. This is a big limitation, especially in Credit Scoring for SMEs where we would expect the pattern is more varied, non-linear, and less straighforward than that of bigger companies
* In that sense, tree-based alogrithms might be good ideas as it conducts the classification taking into account the interactions, non-linearity and working well with categorical variables. The visual presentation of decision trees is easy to commcate, and could play the role as a framework for rule-based credit risk models. We would walk-through the `Recursive Partitioning` and `Chi-squared Automatic Detection (CHAID)`.
* As a hybrid approach, we discuss about `Model-Based Recursive Partitioning (MOB)` based on Logit model, to combine the advantages of both trees and logit regression. This method is also potential to be applied for segmentation in Risk modelling, and to validating the stability of model parameters among different subsets of population. 



```{r setup}
knitr::opts_chunk$set(echo = TRUE)
library(readr)
library(dplyr)
library(knitr)
library(kableExtra)
library(magrittr)
library(DT)
library(mltools)
library(ape)
library(caret)
library(ROCR)
library(DMwR)
library(rpart)
library(ggplot2)
library(ggthemes)
library(RColorBrewer)
library(rattle)
library(vcd)
library(party)
library(partykit)
library(plotROC)
library(CHAID)

source("./Handy_functions/summary_handy.R")
source("./Handy_functions/rank_plot.R")
source("./Handy_functions/var_treat.R")
source("./Handy_functions/gb_integer_cat.R")
source("./Handy_functions/split_sample.R")
source("./Handy_functions/univariate_cont_plot.R")

```



## 1. Data Set

The data set used as the illustration in this document is the public dataset in Kaggle, under the competition named **Give Me Some Credit** (2012) <https://www.kaggle.com/c/GiveMeSomeCredit>, which contains 150,000 observations with 11 features. 

> Based on this Data, Credit scoring algorithms is built, which make a guess at the probability of default, are the method banks use to determine whether or not a loan should be granted. This competition requires participants to improve on the state of the art in credit scoring, by predicting the probability that somebody will experience financial distress in the next two years.

The structure and summary of data as below:

* **Good/Bad Target Variable**: `SeriousDlqnin2yrs`, Person experienced 90 days past due delinquency or worse (Technical Default), take value 0 (good) and 1 (bad)
* All preditors are numeric/integer (which make our life a little bit easier, but we can try another data sets with more categorical variables)


```{r load_data}
cs_training <- read.csv("./Data/cs-training.csv") %>% select(-X) 
train_set = cs_training
str(train_set) 
```

```{r summary}
train_set %>% summary_handy(quantile = T) %>%
  select(Variable, Type, Unique, NA_rate, Mean, Min, Max, Q0.25, Q0.50, Q0.75, Q0.95) %>%
  mutate_if(is.numeric, funs(round(.,2))) %>%
  kable %>% kable_styling()
```
## 2. Initial Data Cleaning

* `MontlyIncome` and `NumerOfDependents` has the missing rate of 20% and 3%, respectively. We would need to do imputation for these missing values by median values
* The `Min`, `Max` and Distribution shows that some cap/floor should be applied for the numeric variable, considering the table of distribution, we cap/floor at 5% and 95% quantile

```{r Data Clean}
train_set %<>%
  mutate(
    RevolvingUtilizationOfUnsecuredLines = var_treat(RevolvingUtilizationOfUnsecuredLines,
                                                     q_low = 0.05, q_up = 0.95, 
                                                     impute_val = median(train_set$RevolvingUtilizationOfUnsecuredLines, 
                                                                         na.rm=T)),
    MonthlyIncome = var_treat(MonthlyIncome, q_low = 0.05, q_up = 0.95,
                              impute_val = median(train_set$MonthlyIncome, na.rm = T)),
    NumberOfDependents = ifelse(is.na(NumberOfDependents), 
                                median(NumberOfDependents, na.rm = T),
                                NumberOfDependents)
  )
```


## 2. Univariate Analysis

For numeric variables, we conduct the rank plots to compare the association between the predictor and Good/Bad flag. We see that: 

* `RevolvingUtilizationOfUnsecuredLines`: very predictive, with single gini up to 54.1%, this predictor represent the utilization of credit limits in credit card after adjusting several factors (See: Data Dictionary). Thus, the correlation is positive, which is intuitive.
* `MonthlyIncome`: moderately predictive with Single Gini of 14.3%, with the negative correlation with Good/Bad. It means that individuals with higher monthly income is less likely to have the financial distress.
* `Debt Ratio`: gives us a U-curve relationship. That is reasonable what a 'good' borrower is likely to be granted some 'healthy' debt, yet when the Debt Ratio increases to over 0.287 (as in graph after capping), the credit risk increase.

### 2.1. Continuous Variables 

```{r RevolvingUtilizationOfUnsecuredLines, fig.width=15, fig.height=10}
univariate_cont_plot(x = train_set$RevolvingUtilizationOfUnsecuredLines, 
                     y = train_set$SeriousDlqin2yrs,
                     varname = 'RevolvingUtilizationOfUnsecuredLines')
```
```{r MonthlyIncome, fig.width=15, fig.height=10}
univariate_cont_plot(x = train_set$MonthlyIncome, 
                     y = train_set$SeriousDlqin2yrs,
                     varname = 'MonthlyIncome')
```
```{r DebtRatio, fig.width=15, fig.height=10}
univariate_cont_plot(x = train_set$DebtRatio, 
                     y = train_set$SeriousDlqin2yrs,
                     varname = 'DebtRatio')
```

#### Variable Treatments

Considering the rank plots, we cap `DebtRatio` at 4
```{r DebtRatio_cap}
train_set %<>%
  mutate(DebtRatio = ifelse(DebtRatio > 4, 4, DebtRatio))

```
```{r DebtRatio_plot, echo=F, fig.width=15, fig.height=10}
# check the rank.plots again
univariate_cont_plot(x = train_set$DebtRatio, 
                     y = train_set$SeriousDlqin2yrs,
                     varname = 'DebtRatio')

```
### 2.2. Discrete Variables

For Discrete Variables, we consider the Univariate Analysis by below metrics:

* **Weight of Evidence (WOE):** shows predictive power of predictors. It increases with credit scoring for the power of seperating good/bad.
* **Information Value (IV):** based on WOE, this helps to select variables by the IV (higher IV, more predictive)
* **Efficiency**: Abs(%Good - %Bad)/2

For `Age` and `NumberOfOpenCreditLinesAndLoans`, we cut them into 5 bins by quantiles. For the others, we consider the table of IV to cut at the value of IV below 2. 
```{r bin_data}
train_set %<>%
  mutate_at(vars(-RevolvingUtilizationOfUnsecuredLines,
               -MonthlyIncome,
               -DebtRatio,
               -SeriousDlqin2yrs,
               -NumberOfTime30.59DaysPastDueNotWorse,
               -NumberOfTime60.89DaysPastDueNotWorse,
               -NumberOfTimes90DaysLate,
               -NumberOfDependents
               ),
            funs(bin_data(., bins = 3, binType = 'quantile')))
  
```

```{r Age}
gb_integer_cat(train_set$age, train_set$SeriousDlqin2yrs) %>%
  mutate_if(is.numeric, funs(round(.,2))) %>%
  datatable()
```
```{r NumberOfOpenCreditLinesAndLoans}
gb_integer_cat(train_set$NumberOfOpenCreditLinesAndLoans, train_set$SeriousDlqin2yrs) %>%
  mutate_if(is.numeric, funs(round(.,2))) %>%
  datatable()
```
```{r NumberOfDependents}
gb_integer_cat(train_set$NumberOfDependents, train_set$SeriousDlqin2yrs) %>%
  mutate_if(is.numeric, funs(round(.,2))) %>%
  datatable()
```
```{r NumberOfTime30.59DaysPastDueNotWorse}
gb_integer_cat(train_set$NumberOfTime30.59DaysPastDueNotWorse, train_set$SeriousDlqin2yrs) %>%
  mutate_if(is.numeric, funs(round(.,2))) %>%
  datatable()
```
```{r NumberOfTime60.89DaysPastDueNotWorse}
gb_integer_cat(train_set$NumberOfTime60.89DaysPastDueNotWorse, train_set$SeriousDlqin2yrs) %>%
  mutate_if(is.numeric, funs(round(.,2))) %>%
  datatable()
```
```{r NumberOfTimes90DaysLate}
gb_integer_cat(train_set$NumberOfTimes90DaysLate, train_set$SeriousDlqin2yrs) %>%
  mutate_if(is.numeric, funs(round(.,2))) %>%
  datatable()
```

#### Variable Treatments

Based on the table of IV, we would group `NumberOfDependents`, `NumberOfTime30.59DaysPastDueNotWorse`,`NumberOfTime60.89DaysPastDueNotWorse`,`NumberOfTimes90DaysLate` at the cut-off that the IV declines to below 2.

```{r group_variable}
train_set %<>%
  mutate(NumberOfDependents = ifelse(NumberOfDependents > 0, 1, NumberOfDependents),
         NumberOfTime30.59DaysPastDueNotWorse = ifelse(NumberOfTime30.59DaysPastDueNotWorse > 4, 
                                                       4, NumberOfTime30.59DaysPastDueNotWorse),
         NumberOfTime60.89DaysPastDueNotWorse = ifelse(NumberOfTime60.89DaysPastDueNotWorse > 3,
                                                       3, NumberOfTime60.89DaysPastDueNotWorse),
         NumberOfTimes90DaysLate = ifelse(NumberOfTimes90DaysLate > 3, 
                                          3, NumberOfTimes90DaysLate)
         )

```

We can see that after treating, the IV generally shows up better among levels of variables. 

```{r NumberOfDependents2}
gb_integer_cat(train_set$NumberOfDependents, train_set$SeriousDlqin2yrs) %>%
  mutate_if(is.numeric, funs(round(.,2))) %>%
  datatable()
```
```{r NumberOfTime30.59DaysPastDueNotWorse2}
gb_integer_cat(train_set$NumberOfTime30.59DaysPastDueNotWorse, train_set$SeriousDlqin2yrs) %>%
  mutate_if(is.numeric, funs(round(.,2))) %>%
  datatable()
```
```{r NumberOfTime60.89DaysPastDueNotWorse2}
gb_integer_cat(train_set$NumberOfTime60.89DaysPastDueNotWorse, train_set$SeriousDlqin2yrs) %>%
  mutate_if(is.numeric, funs(round(.,2))) %>%
  datatable()
```
```{r NumberOfTimes90DaysLate2}
gb_integer_cat(train_set$NumberOfTimes90DaysLate, train_set$SeriousDlqin2yrs) %>%
  mutate_if(is.numeric, funs(round(.,2))) %>%
  datatable()
```

#### Discrete Variable Plots

##### Age Groups

The bad rate declines by the higher age group. It means that the older borrowers become, the safer they are. This is interesting pattern, as the older people might be more cautious in spending and saving, that make them less likely to have a financial distress (comparing to young age).

```{r Age_plot}
Age <- gb_integer_cat(train_set$age, train_set$SeriousDlqin2yrs)
op1<-par(mfrow=c(1,2), new=TRUE)
plot(as.factor(train_set$age), as.factor(train_set$SeriousDlqin2yrs), 
     ylab="Good-Bad", xlab="category", 
     main="Age vs. Good-Bad ")
barplot(Age$WOE, col="brown", names.arg=c(Age$Levels),
        main="Age Group",
        xlab="Good-Bad",
        ylab="WOE")
```

##### Open Loans

The bad rate is higher for the group with < 4 open loans, whom are likely to be out of risk appetite, then generally they do not win a loan/credit for themselves. However, the existence of loans have the diminishing positive signal, as once borrowers have too much loans, it means that they could easier to face a financial distress.

```{r realestate_plot}
real_estate <- gb_integer_cat(train_set$NumberRealEstateLoansOrLines, train_set$SeriousDlqin2yrs)
op1<-par(mfrow=c(1,2), new=TRUE)
plot(as.factor(train_set$NumberRealEstateLoansOrLines), as.factor(train_set$SeriousDlqin2yrs), 
     ylab="Good-Bad", xlab="category", 
     main="#Open-Loans vs. Good-Bad ")
barplot(real_estate$WOE, col="brown", names.arg=c(real_estate$Levels),
        main="#Open-Loans Group",
        xlab="Good-Bad",
        ylab="WOE")
```

```{r CreditLines_plot}
credit_lines <- gb_integer_cat(train_set$NumberOfOpenCreditLinesAndLoans, train_set$SeriousDlqin2yrs)
op1<-par(mfrow=c(1,2), new=TRUE)
plot(as.factor(train_set$NumberOfOpenCreditLinesAndLoans), as.factor(train_set$SeriousDlqin2yrs), 
     ylab="Good-Bad", xlab="category", 
     main="#Open-Loans vs. Good-Bad ")
barplot(credit_lines$WOE, col="brown", names.arg=c(credit_lines$Levels),
        main="#Open-Loans Group",
        xlab="Good-Bad",
        ylab="WOE")
```

##### Number of Dependents

Obviously, the dependents are likely financial burden. That WOE is negative for people having no dependents.

```{r dependent_plot}
dependent <- gb_integer_cat(train_set$NumberOfDependents, train_set$SeriousDlqin2yrs)
op1<-par(mfrow=c(1,2), new=TRUE)
plot(as.factor(train_set$NumberOfDependents), train_set$SeriousDlqin2yrs, 
     ylab="Good-Bad", xlab="category", 
     main="NumberOfDependents vs. Good-Bad ")
barplot(dependent$WOE, col="brown", names.arg=c(dependent$Levels),
        main="NumberOfDependents Group",
        xlab="Good-Bad",
        ylab="WOE")
```

##### Number of Late Payments

For different windows of late payments, the pattern is the same. People with No ever-late payment is much better, while the number of being late increase, their probability of being bad (by the definition) also increases.

```{r l30_plot}
l30 <- gb_integer_cat(train_set$NumberOfTime30.59DaysPastDueNotWorse, train_set$SeriousDlqin2yrs)
op1<-par(mfrow=c(1,2), new=TRUE)
plot(as.factor(train_set$NumberOfTime30.59DaysPastDueNotWorse), 
     as.factor(train_set$SeriousDlqin2yrs), 
     ylab="Good-Bad", xlab="category", 
     main="Late-30plus vs. Good-Bad ")
barplot(l30$WOE, col="brown", names.arg=c(l30$Levels),
        main="Count-of-late30+ Group",
        xlab="Good-Bad",
        ylab="WOE")
```
```{r l60_plot}
l60 <- gb_integer_cat(train_set$NumberOfTime60.89DaysPastDueNotWorse, train_set$SeriousDlqin2yrs)
op1<-par(mfrow=c(1,2), new=TRUE)
plot(as.factor(train_set$NumberOfTime60.89DaysPastDueNotWorse), 
     as.factor(train_set$SeriousDlqin2yrs), 
     ylab="Good-Bad", xlab="category", 
     main="Late-60plus vs. Good-Bad ")
barplot(l60$WOE, col="brown", names.arg=c(l30$Levels),
        main="Count-of-late60+ Group",
        xlab="Good-Bad",
        ylab="WOE")
```
```{r l90_plot}
l90 <- gb_integer_cat(train_set$NumberOfTimes90DaysLate, train_set$SeriousDlqin2yrs)
op1<-par(mfrow=c(1,2), new=TRUE)
plot(as.factor(train_set$NumberOfTimes90DaysLate), as.factor(train_set$SeriousDlqin2yrs), 
     ylab="Good-Bad", xlab="category", 
     main="Late-90plus vs. Good-Bad ")
barplot(l90$WOE, col="brown", names.arg=c(l90$Levels),
        main="Count-of-late90+ Group",
        xlab="Good-Bad",
        ylab="WOE")
```

### 2. Feature Engineering

We are further motivated to do the feature engineering, in the sense that:

* Interaction between Monthly Income and Utilization of Credit Limits (after cleaning no `MonthlyIncome` is zero)
* Interaction between Monthly Income and Debt Ratio

```{r feature_engineering}
train_set %<>%
  mutate(income_util = RevolvingUtilizationOfUnsecuredLines / MonthlyIncome,
         income_debt = MonthlyIncome*DebtRatio)
```

```{r rankplot_income_util, echo=F, fig.width=15, fig.height=10}
univariate_cont_plot(x = train_set$income_util, 
                     y = train_set$SeriousDlqin2yrs,
                     varname = 'income_util')
```


```{r rankplot_income_debt, echo=F, fig.width=15, fig.height=10}
univariate_cont_plot(x = train_set$income_debt, 
                     y = train_set$SeriousDlqin2yrs,
                     varname = 'income_debt')
```

## 3. Multivariate Analysis

We do the hierarchical Clusterning of Variables to understand their interactions.


```{r tree_plot, fig.height=10, fig.width=10}
tree <- ClustOfVar::hclustvar(X.quanti=train_set %>% 
                                select_if(is.numeric) %>% select(-SeriousDlqin2yrs),
                              X.quali=train_set %>% select_if(is.factor))
plot(as.phylo(tree), type = "fan",
     tip.color = hsv(runif(15, 0.65,  0.95), 1, 1, 0.7),
     edge.color = hsv(runif(10, 0.65, 0.75), 1, 1, 0.7), 
     edge.width = runif(20,  0.5, 3), use.edge.length = TRUE, col = "gray80")
```

* By this we see that `income_debt` and `income_util` correlated with their original variables (which is not surprising), yet the generated variables do not have better single gini than the original one. We would remove `income_debt`, but `income_util` would be kept (instead of `RevolvingUtilizationOfUnsecuredLines`) as we prefer ratio than absolute value
* `NumberRealEstateLoansOrLines` and `NumberOfOpenCreditLinesAndLoans` are correlated, which make sense that they are typically same type of information. We take the original values, sum up all loans, and take 5 bins by quantile
* While variables of late payments are correlated as the longer window of being late might involve the shorter window, we transform being late at 30+, 60+, 90+ as a binary, and sum up all count of late times. But we kept them at these steps.

```{r final_trans}
train_set$num_loans_realestate = cs_training$NumberRealEstateLoansOrLines + 
  cs_training$NumberOfOpenCreditLinesAndLoans

train_set %<>%
  mutate(num_loans_realestate = bin_data(num_loans_realestate, bins=3, binType = 'quantile')) %>%
  select(-NumberRealEstateLoansOrLines,
         -NumberOfOpenCreditLinesAndLoans,
         -RevolvingUtilizationOfUnsecuredLines,
         -income_debt)

```
## 3. Model Development

Use the built function to split the `train_set` by 60:40:

```{r split_data}
train <- data.frame()
test <- data.frame()
split_data(train_set, seed=1234, p=0.6, target='SeriousDlqin2yrs')

```
## 3.0. SMOTE for Unbalanced Data

The nature of data set is unbalance, with only 7% of bad rate. In this situation, the classification categories are not approximately equally represented, which migh lead to the overfitting, also some machine learning techniques are particularly sensitive to the balance of data (but not likely in our case). We deal with this situation by conducting SMOTE to adjust the class distribution of the dat set. 
The idea of SMOTE is taking the numeric/continuous variables in the features. The algorithm doing the oversampling by generating the synthetic data point from the vector among k-nearest neighbors. 
However, due to this manner of generating over-sampled data points, it might break the natural structure of data, in our case, the variables are integer. 

This issue is clearer when we apply SMOTE for Recursive Partitioning. 

```{r before_smote}
## The bad ratio before SMOTE ()
train$SeriousDlqin2yrs %>% table()

```

```{r smote}
train_smote <- train %>%
  mutate(SeriousDlqin2yrs = as.factor(SeriousDlqin2yrs)) ## for SMOTE, target should be factor

train_smote <- DMwR::SMOTE(SeriousDlqin2yrs ~ ., train_smote, perc.over = 100, perc.under = 500)
train_smote$SeriousDlqin2yrs %>% table()


```

## 3.1. Logistic Regression

In this part, we simply process the Logistic Regression, and then stepwise for variable selection. 
```{r logistic}
m1 <- glm(SeriousDlqin2yrs ~ .,
          data=train,
          family=binomial())
m1 <- step(m1)

```
```{r logit_sum, echo=F}
summary(m1)
```

Now, scoring on `test` set
```{r logit1_score}
test$m1_score <- predict(m1,type='response',test)
m1_pred <- ROCR::prediction(test$m1_score, test$SeriousDlqin2yrs)
m1_perf <- ROCR::performance(m1_pred,"tpr","fpr")

m1_KS <- round(max(attr(m1_perf,'y.values')[[1]]-attr(m1_perf,'x.values')[[1]])*100, 2)
m1_AUROC <- round(performance(m1_pred, measure = "auc")@y.values[[1]]*100, 2)
m1_Gini <- (2*m1_AUROC - 100)
cat("Test set: AUROC: ",m1_AUROC,"\tKS: ", m1_KS, "\tGini:", m1_Gini, "\n")

```
```{r logit1_score2}
m1_score2 <- predict(m1,type='response',train)
m1_pred2 <- ROCR::prediction(m1_score2, train$SeriousDlqin2yrs)
m1_perf2 <- ROCR::performance(m1_pred2,"tpr","fpr")

m1_KS2 <- round(max(attr(m1_perf2,'y.values')[[1]]-attr(m1_perf2,'x.values')[[1]])*100, 2)
m1_AUROC2 <- round(performance(m1_pred2, measure = "auc")@y.values[[1]]*100, 2)
m1_Gini2 <- (2*m1_AUROC2 - 100)
cat("Test set: AUROC: ",m1_AUROC2,"\tKS: ", m1_KS2, "\tGini:", m1_Gini2, "\n")

```

Most of variables are strongly significant, `MonthlyIncome` is only significant at 5% and this information is already involved in `income_util`, we remove it. Also, the information of debt/loans are included in other features, we remove `DebtRatio`:

```{r logit2}
train2 <- train %>% select(-MonthlyIncome, -DebtRatio)
m1b <- glm(SeriousDlqin2yrs ~ .,
          data=train2,
          family=binomial())
m1b <- step(m1b)

```
```{r logit_sum2, echo=F}
summary(m1b)

```

```{r logit2_score, echo=F}
test$m1b_score <- predict(m1b,type='response',test)
m1b_pred <- prediction(test$m1b_score, test$SeriousDlqin2yrs)
m1b_perf <- performance(m1b_pred,"tpr","fpr")

m1b_KS <- round(max(attr(m1b_perf,'y.values')[[1]]-attr(m1b_perf,'x.values')[[1]])*100, 2)
m1b_AUROC <- round(performance(m1b_pred, measure = "auc")@y.values[[1]]*100, 2)
m1b_Gini <- (2*m1b_AUROC - 100)

m1b_score2 <- predict(m1b,type='response',train2)
m1b_pred2 <- prediction(m1b_score2, train2$SeriousDlqin2yrs)
m1b_perf2 <- performance(m1b_pred2,"tpr","fpr")

m1b_KS2 <- round(max(attr(m1b_perf2,'y.values')[[1]]-attr(m1b_perf2,'x.values')[[1]])*100, 2)
m1b_AUROC2 <- round(performance(m1b_pred2, measure = "auc")@y.values[[1]]*100, 2)
m1b_Gini2 <- (2*m1b_AUROC2 - 100)

cat("Train set - AUROC: ",m1b_AUROC2,"\tKS: ", m1b_KS2, "\tGini:", m1b_Gini2, "\n")
cat("Test set - AUROC: ",m1b_AUROC,"\tKS: ", m1b_KS, "\tGini:", m1b_Gini, "\n")
```
Now, we run the logistic on SMOTE train set.

```{r logit3}
train_smote <- train_smote %>% select(-MonthlyIncome, -DebtRatio)
m1c <- glm(SeriousDlqin2yrs ~ .,
          data=train_smote,
          family=binomial())
m1c <- step(m1b)

```
```{r logit_sum3, echo=F}
summary(m1c)

```

```{r logit3_score, echo=F}
test$m1c_score <- predict(m1c,type='response',test)
m1c_pred <- prediction(test$m1c_score, test$SeriousDlqin2yrs)
m1c_perf <- performance(m1c_pred,"tpr","fpr")

m1c_KS <- round(max(attr(m1c_perf,'y.values')[[1]]-attr(m1c_perf,'x.values')[[1]])*100, 2)
m1c_AUROC <- round(performance(m1c_pred, measure = "auc")@y.values[[1]]*100, 2)
m1c_Gini <- (2*m1c_AUROC - 100)


m1c_score2 <- predict(m1c,type='response',train_smote)
m1c_pred2 <- prediction(m1c_score2, train_smote$SeriousDlqin2yrs)
m1c_perf2 <- performance(m1c_pred2,"tpr","fpr")

m1c_KS2 <- round(max(attr(m1c_perf2,'y.values')[[1]]-attr(m1c_perf2,'x.values')[[1]])*100, 2)
m1c_AUROC2 <- round(performance(m1c_pred2, measure = "auc")@y.values[[1]]*100, 2)
m1c_Gini2 <- (2*m1c_AUROC2 - 100)

cat("Train set - AUROC: ",m1c_AUROC2,"\tKS: ", m1c_KS2, "\tGini:", m1c_Gini2, "\n")
cat("Test set - AUROC: ",m1c_AUROC,"\tKS: ", m1c_KS, "\tGini:", m1c_Gini, "\n")
```

## 3.2. Recursive Partitioning

Recursive Paritioning is a tree-based algorithm to create a decision tree, which segments the population into sub-populations. The split is by a set of predictors. With our dataset, we use the regression tree for numeric variables. 

The overfitting is avoided by pruning the tree. We do that by looking into the cross-validation relative error `X-val Relative Error` and select the `cp` value at the minimum error considering the size of the tree. 

```{r recursive_tree1}
m2 <- rpart(SeriousDlqin2yrs~.,data=train2)
# Print tree detail
plotcp(m2)

```

```{r prunning}
## Prunning the tree
ctrl <- rpart::rpart.control(cp = 0.018)
m2 <- rpart(SeriousDlqin2yrs~.,data=train2, control = ctrl)
rattle::fancyRpartPlot(m2)
```


```{r rtree1}
test$m2_score <- predict(m2,type='vector',test)
m2_pred <- prediction(test$m2_score, test$SeriousDlqin2yrs)
m2_perf <- performance(m2_pred,"tpr","fpr")

m2_KS <- round(max(attr(m2_perf,'y.values')[[1]]-attr(m2_perf,'x.values')[[1]])*100, 2)
m2_AUROC <- round(performance(m2_pred, measure = "auc")@y.values[[1]]*100, 2)
m2_Gini <- (2*m2_AUROC - 100)

m2_score2 <- predict(m2,type='vector',train2)
m2_pred2 <- prediction(m2_score2, train2$SeriousDlqin2yrs)
m2_perf2 <- performance(m2_pred2,"tpr","fpr")

m2_KS2 <- round(max(attr(m2_perf2,'y.values')[[1]]-attr(m2_perf2,'x.values')[[1]])*100, 2)
m2_AUROC2 <- round(performance(m2_pred2, measure = "auc")@y.values[[1]]*100, 2)
m2_Gini2 <- (2*m2_AUROC2 - 100)

cat("Train set - AUROC: ",m2_AUROC2,"\tKS: ", m2_KS2, "\tGini:", m2_Gini2, "\n")
cat("Test set - AUROC: ",m2_AUROC,"\tKS: ", m2_KS, "\tGini:", m2_Gini, "\n")

```

As mentioned above, SMOTE does over-sampling by creating the synthetic data point from k-nearest neighbor. There, we see the tree split the population at some 'weird' cut-off. Considering that our original variable taking integer values, the cut-off is not integer, such as 0.0044 of `NumberOfTime60.89DaysPastDueNotWorse`. In fact, we could consider it to be zero. 

```{r recursive_tree_smote}
m2_smote <- rpart(SeriousDlqin2yrs~.,data=train_smote)
# Print tree detail
printcp(m2_smote)

```
```{r print_tree}
rattle::fancyRpartPlot(m2_smote)

```

```{r rtree_smote, echo=F}
test$m2b_score <- predict(m2_smote,type='vector',test)
m2b_pred <- prediction(test$m2b_score, test$SeriousDlqin2yrs)
m2b_perf <- performance(m2b_pred,"tpr","fpr")

m2b_KS <- round(max(attr(m2b_perf,'y.values')[[1]]-attr(m2b_perf,'x.values')[[1]])*100, 2)
m2b_AUROC <- round(performance(m2b_pred, measure = "auc")@y.values[[1]]*100, 2)
m2b_Gini <- (2*m2b_AUROC - 100)

m2b_score2 <- predict(m2_smote,type='vector',train_smote)
m2b_pred2 <- prediction(m2b_score2, train_smote$SeriousDlqin2yrs)
m2b_perf2 <- performance(m2b_pred2,"tpr","fpr")

m2b_KS2 <- round(max(attr(m2b_perf2,'y.values')[[1]]-attr(m2b_perf2,'x.values')[[1]])*100, 2)
m2b_AUROC2 <- round(performance(m2b_pred2, measure = "auc")@y.values[[1]]*100, 2)
m2b_Gini2 <- (2*m2b_AUROC2 - 100)

cat("Train set - AUROC: ",m2b_AUROC2,"\tKS: ", m2b_KS2, "\tGini:", m2b_Gini2, "\n")
cat("Test set - AUROC: ",m2b_AUROC,"\tKS: ", m2b_KS, "\tGini:", m2b_Gini, "\n")

```

## 3.3. Unbiased Non-parametric Model-Based (Logistic)

**Model-based Recurive Partitioning (MOB)** is the combination of Logistic Regression with the tree-based algorithms. Thus, it combines the advantages and augmented the disadvantages of both methods. Generally speaking, MOB combines two main layers: (1) the recursive partitioning by the features (could be categorical or binned numeric variables), then within each terminal node (2), it conduct the regression (logistic) by the determined features within each node. 

**The algorithm follows:** 

1. Fitting the model on regressors (i.e. by logistic), using the sample at the current node
2. Assess the stability of the model parameters with resoect to the parition by the paritioning variables. Once it is instable, split at the partitioning variable with smallest p-val. 
3. Then searching the cut-off to split within the specific partitionaing variable (selected at Step 2)

**Conceptual Meanings:**

* The tree-based layer augments the disadvantage of Logistics (and GLM in general) favouring linearity and mono-tone relationsip in predictions. The tree-based layer helps to capture the non-linearity and interaction between predictors
* The tree-based layers is effective to deal with categorical variables, which ease the burden of GLM to do the variable selection among levels of categorical variables. 
* The bottom-layer of Logistics would still capture some strong numeric, linear, monotone variables (such as: `RevolvingUtilizationOfUnsecuredLines`)

**Practical Meanings:**

* MOB could be used for segmentations (such as by industries, ages of business, levels of revenue, etc.) in credit scoring modelling. Within each segment, the Logistic, which is popular in the industry could be applied as normally
* The tendency to rely on the stability of model parameters aligns with common framework in credit scoring model validation
* The visualisation of MOB, if not being applied for the main model development, is still be beneficial to be used for model validation to test the stability and changes of slopes across different segmetns. Also, it is useful to exploring the data to advice whether we need the segmentation in model development. 

### 3.3.1. Model-Based Recursive Partitioning (Feature Engineering)

In this part, we do some feature engineering to combine the count of late payments in 30+, 60+, and 90+ by arbitrarily assigning weight for them. Intuitively, being late at 90+ would give stronger signals about the risk of the observation, we assign 3 for 90+, then 2, and 1 for 60+ and 30+ (this is totally arbitrarily, later we would try some more sophisticated weighting method). 
Also, we keep a flag of ever being late 60+ or 90+, to reserve a part of discrimination. 

When running mob, we put `income_util` and `late_count` as regression of the logistic layer, and the other binned variables as partitioning features of the tree-based layers. 
One could imagine it as the process of segmenting by being late, age, and number of loans, then within each segment, we run the logit model on two variables. 

As the task of model validation, we could look into the graph to see if the stability of slopes in each features. 
* Here, obviously, we see the slope of `income_util` changes across segments, while the the slope is steep (it means the feature is more predictive) in the segment without ever-being late 60+ and 90+, young age, and have fewer number of loans. This factor is not discriminative in the riskier population (being late 60 and 90+). Also, even within the same branch, this utilization over income seems to be more predictive in older people. We can guess that once the older people over-use their credit limits, it is a very strong signal of being 'bad', while the over-using of credit limit seems to be more popular (then less discriminative) among young people. 
* Similar, the number of times being late is more preditive among the group of 'better' people, while it is not predictive in the group that everyone are 'bad' at the quite similar level.

Similar to other algorithm, MOB could be tuned by `mob_control()`, by doing some grid search. Yet, considering the limited number of predictors, and the data is quite simple, no tuning exercise is taken.

```{r mob1, fig.width=30, fig.height=20}
train <- train %>%
  mutate(late_count = NumberOfTime30.59DaysPastDueNotWorse*1 +
           NumberOfTime60.89DaysPastDueNotWorse*2 +
           NumberOfTimes90DaysLate*3,
         late60plus = ifelse(NumberOfTime60.89DaysPastDueNotWorse > 0 |
                               NumberOfTimes90DaysLate > 0, 1, 0))

m3 <- party::mob(SeriousDlqin2yrs ~ income_util + late_count |
                   num_loans_realestate +
                   late60plus +
                   #NumberOfTime30.59DaysPastDueNotWorse +
                   #NumberOfDependents +
                   age, 
                 #control = mob_control(maxdepth = 4),
                 data=train,
                 family=binomial())

plot(m3, main="Model 3: Model based Tree with GLM")
```


```{r m3_perf, echo=F}
test <- test %>%
  #mutate(bad_late = ifelse(NumberOfTime30.59DaysPastDueNotWorse > 3 |
  #                           NumberOfTime30.59DaysPastDueNotWorse > 1 |
  #                           NumberOfTimes90DaysLate > 1, 1, 0))
  mutate(late_count = NumberOfTime30.59DaysPastDueNotWorse*1 +
           NumberOfTime60.89DaysPastDueNotWorse*2 +
           NumberOfTimes90DaysLate*3,
         late60plus = ifelse(NumberOfTime60.89DaysPastDueNotWorse > 0 |
                               NumberOfTimes90DaysLate > 0, 1, 0))

test$m3_score <- predict(m3,type='response',test)
m3_pred <- prediction(test$m3_score, test$SeriousDlqin2yrs)
m3_perf <- performance(m3_pred,"tpr","fpr")

m3_KS <- round(max(attr(m3_perf,'y.values')[[1]]-attr(m3_perf,'x.values')[[1]])*100, 2)
m3_AUROC <- round(performance(m3_pred, measure = "auc")@y.values[[1]]*100, 2)
m3_Gini <- (2*m3_AUROC - 100)

m3_score2 <- predict(m3, type='response',train)
m3_pred2 <- prediction(m3_score2, train$SeriousDlqin2yrs)
m3_perf2 <- performance(m3_pred2,"tpr","fpr")

m3_KS2 <- round(max(attr(m3_perf2,'y.values')[[1]]-attr(m3_perf2,'x.values')[[1]])*100, 2)
m3_AUROC2 <- round(performance(m3_pred2, measure = "auc")@y.values[[1]]*100, 2)
m3_Gini2 <- (2*m3_AUROC2 - 100)

cat("Train set - AUROC: ",m3_AUROC2,"\tKS: ", m3_KS2, "\tGini:", m3_Gini2, "\n")
cat("Test set - AUROC: ",m3_AUROC,"\tKS: ", m3_KS, "\tGini:", m3_Gini, "\n")

```

### 3.3.2. Model-Based Recursive Partitioning (Logit Score)

As the previous promise, now we decide the weights of counts of being lates by doing the logistic regression of target variable on these variables. For more simple version, we also combine `income_util` into the logistic score. Same as the previous part, we see that the slope (model parameter of `logit_late`) changes acoss the segment. 


```{r model_based2_glm}

logit_late <- glm(SeriousDlqin2yrs ~ 
                    NumberOfTime30.59DaysPastDueNotWorse +
                    NumberOfTime60.89DaysPastDueNotWorse +
                    NumberOfTimes90DaysLate +
                    income_util,
                  data = train, 
                  family = binomial)
summary(logit_late)
```
```{r mob2, fig.width=20, fig.height=10}
train$late_count_logit = predict(m1b, train, type = 'response')
train <- train %>%
  mutate(late60plus = ifelse(NumberOfTime60.89DaysPastDueNotWorse > 0 |
                               NumberOfTimes90DaysLate > 0, 1, 0))

m3b <- party::mob(SeriousDlqin2yrs ~ late_count_logit |
                   num_loans_realestate +
                   late60plus +
                   #NumberOfTime30.59DaysPastDueNotWorse +
                   #NumberOfDependents +
                   age, 
           #control = mob_control(maxdepth = 3, alpha = 0.05),
           data=train,
           family=binomial())

plot(m3b, main="Model 3b: Model based Tree with GLM (Late-count Logit Score)")

```

```{r m3b_perf, echo=F}
test$late_count_logit = predict(m1b, test, type = 'response')
test <- test %>%
  mutate(late60plus = ifelse(NumberOfTime60.89DaysPastDueNotWorse > 0 |
                               NumberOfTimes90DaysLate > 0, 1, 0))

test$m3b_score <- predict(m3b,type='response',test)
m3b_pred <- prediction(test$m3b_score, test$SeriousDlqin2yrs)
m3b_perf <- performance(m3b_pred,"tpr","fpr")

m3b_KS <- round(max(attr(m3b_perf,'y.values')[[1]]-attr(m3b_perf,'x.values')[[1]])*100, 2)
m3b_AUROC <- round(performance(m3b_pred, measure = "auc")@y.values[[1]]*100, 2)
m3b_Gini <- (2*m3b_AUROC - 100)

m3b_score2 <- predict(m3b, type='response',train)
m3b_pred2 <- prediction(m3b_score2, train$SeriousDlqin2yrs)
m3b_perf2 <- performance(m3b_pred2,"tpr","fpr")

m3b_KS2 <- round(max(attr(m3b_perf2,'y.values')[[1]]-attr(m3b_perf2,'x.values')[[1]])*100, 2)
m3b_AUROC2 <- round(performance(m3b_pred2, measure = "auc")@y.values[[1]]*100, 2)
m3b_Gini2 <- (2*m3b_AUROC2 - 100)

cat("Train set - AUROC: ",m3b_AUROC2,"\tKS: ", m3b_KS2, "\tGini:", m3b_Gini2, "\n")
cat("Test set - AUROC: ",m3b_AUROC,"\tKS: ", m3b_KS, "\tGini:", m3b_Gini, "\n")
```

## 3.4. Chi-square Automatic Interaction Detector (CHAID)

As CHAID only take the caterogical variables, to conduct the chi-squared among features. It is more analytical-oriented, as it is a tool used ot discover the relationship between categorical variables (hence, all variables are converted to categorical for CHAID algorithm).  

With the same manner as CART, CHAID figures out the way to merge and split among different categorical variables, and within different levels of one categorical variables to segment the population into different subset, corresponding to the target variables. 

Different from CART/Regression Tree, CHAID splits the population by features, depending on the statistical significance of chi-square of independency among two categorical features/two levels of one categorical feature. That makes CHAID to be more favoured for analytics, comparing to other predictive tree-based. Also, different from CART, CHAID is able to split non-binary at each node. 

In this part, we would illustrate the process of tuning a tree model by grid search (which are similar to other machine learning techniques, within and out of this work). Yet, depending on the business sense and purpose of analytics/predictics, we can also determine the desired level of complexity of the model. For CHAID, we would compare grid-search tunned vs. arbitrary parameter controls. 

```{r chaid_feature_eng}

y = c(train$SeriousDlqin2yrs, test$SeriousDlqin2yrs)
traintest_chaid <- rbind(train, test[,names(train)]) %>% 
  mutate_if(function(v) length(unique(v)) < 10, as.factor) %>%
  mutate_if(is.numeric, funs(as.factor(as.numeric((bin_data(., bins = 5, binType = 'quantile')))))) 

traintest_chaid$SeriousDlqin2yrs = as.factor(y)

train_chaid = traintest_chaid[1:nrow(train),]
test_chaid = traintest_chaid[(nrow(train)+1):nrow(traintest_chaid),]

summary(train_chaid)
```

```{r chaid_tune}
train_control <- trainControl(method = "cv",
                              number = 5,
                              verboseIter = TRUE,
                              savePredictions = "final")

chaid.m1 <- train(
   x = train_chaid %>% select(-SeriousDlqin2yrs),
   y = train_chaid$SeriousDlqin2yrs,
   method = "chaid",
   metric = "Accuracy",
   trControl = train_control
 )

chaid_graph <- function(model, title){
  
  plot(
    model, 
    main = title, ## title
    gp = grid::gpar(
      col = "blue",
      lty = "solid",
      lwd = 3,
      fontsize = 8
    )
  )  
}

chaid_graph(chaid.m1, title = 'Model 4: CHAID Tuned')
```

```{r chaid_predict0}
test$m4_score <- predict(chaid.m1,type='prob',test_chaid)
m4_pred <- prediction(test$m4_score[,2], test$SeriousDlqin2yrs)
m4_perf <- performance(m4_pred,"tpr","fpr")

m4_KS <- round(max(attr(m4_perf,'y.values')[[1]]-attr(m4_perf,'x.values')[[1]])*100, 2)
m4_AUROC <- round(performance(m4_pred, measure = "auc")@y.values[[1]]*100, 2)
m4_Gini <- (2*m4_AUROC - 100)

m4_score2 <- predict(chaid.m1, type='prob',train_chaid)
m4_pred2 <- prediction(m4_score2[,2], train$SeriousDlqin2yrs)
m4_perf2 <- performance(m4_pred2,"tpr","fpr")

m4_KS2 <- round(max(attr(m4_perf2,'y.values')[[1]]-attr(m4_perf2,'x.values')[[1]])*100, 2)
m4_AUROC2 <- round(performance(m4_pred2, measure = "auc")@y.values[[1]]*100, 2)
m4_Gini2 <- (2*m4_AUROC2 - 100)

cat("Train set - AUROC: ",m4_AUROC2,"\tKS: ", m4_KS2, "\tGini:", m4_Gini2, "\n")
cat("Test set - AUROC: ",m4_AUROC,"\tKS: ", m4_KS, "\tGini:", m4_Gini, "\n")
```


```{r chaid_fit}
# install.packages("CHAID", repos="http://R-Forge.R-project.org")
ctrl.alpha <- CHAID::chaid_control(alpha2 = 0.05, alpha4 = 0.05, maxheight = 3)
model4 <- CHAID::chaid(SeriousDlqin2yrs ~ ., data = train_chaid, control = ctrl.alpha)

```
```{r chaid_graph, fig.height=15, fig.width=20}
chaid_graph(model4, title = 'Model 4b: CHAID with alpha2 = 0.05 and alpha4 = 0.05')

```
```{r chaid_predict}
test$m4b_score <- predict(model4,type='prob',test_chaid)
m4b_pred <- prediction(test$m4b_score[,2], test$SeriousDlqin2yrs)
m4b_perf <- performance(m4b_pred,"tpr","fpr")

m4b_KS <- round(max(attr(m4b_perf,'y.values')[[1]]-attr(m4b_perf,'x.values')[[1]])*100, 2)
m4b_AUROC <- round(performance(m4b_pred, measure = "auc")@y.values[[1]]*100, 2)
m4b_Gini <- (2*m4b_AUROC - 100)

m4b_score2 <- predict(model4, type='prob',train_chaid)
m4b_pred2 <- prediction(m4b_score2[,2], train$SeriousDlqin2yrs)
m4b_perf2 <- performance(m4b_pred2,"tpr","fpr")

m4b_KS2 <- round(max(attr(m4b_perf2,'y.values')[[1]]-attr(m4b_perf2,'x.values')[[1]])*100, 2)
m4b_AUROC2 <- round(performance(m4b_pred2, measure = "auc")@y.values[[1]]*100, 2)
m4b_Gini2 <- (2*m4b_AUROC2 - 100)

cat("Train set - AUROC: ",m4b_AUROC2,"\tKS: ", m4b_KS2, "\tGini:", m4b_Gini2, "\n")
cat("Test set - AUROC: ",m4b_AUROC,"\tKS: ", m4b_KS, "\tGini:", m4b_Gini, "\n")
```


## 4. Model Performance

Different algorithms in this work are compared by ROC and Gini. However, please keep in mind that the best method in this exercise not necessary be the superior for other situations. 
The performance of methods heavily depends on the status of data and the purpose of modellers. 

### 4.1. ROC Curve
```{r roc_curve, fig.width=10, fig.height=8}
plot(m1_perf, col='blue', lty=1, main='ROCs: Model Performance Comparision') # logistic regression
plot(m1b_perf, col='gold',lty=2, add=TRUE); # Logit selected var
plot(m1c_perf, col='dark orange',lty=3, add=TRUE); # Logit + SMOTE
plot(m2_perf, col='green',add=TRUE,lty=4); # tree
plot(m2b_perf, col='dark gray',add=TRUE,lty=5); # Tree SMOTE
plot(m3_perf, col='dark green',add=TRUE,lty=6); # MOB
plot(m3b_perf, col='dark red',add=TRUE,lty=6); # MOB (logit score)
plot(m4_perf, col='black',add=TRUE,lty=6, size = 1); # CHAID Tuned
plot(m4b_perf, col='yellow',add=TRUE,lty=6); # CHAID Arbitrate
    legend(0.6,0.5,
           c('Model 1 :Logistic regression', 
             'Model 1b:Logistic regression - selected vars',
             'Model 1c: Logistic + SMOTE',
             'Model 2 : Recursive Partitioning',
             'Model 2b: Recursive Partitioning + SMOTE',
             'Model 3 : MOB',
             'Model 3b: MOB (logit score)',
             'Model 4 : CHAID (Tuned)',
             'Model 4b: CHAID (Arbitrary)'),
           col=c('blue','gold', 'dark orange','green', 'dark gray', 'dark green','dark red',
                 'black','yellow'),
           lwd=3);
lines(c(0,1),c(0,1),col = "gray", lty = 4 ) # random line

```



### 4.2. Table of Performance Metrics
```{r model_performance}
# Performance Table
models <- c('Model 1 :Logistic regression', 
            'Model 1b:Logistic regression - selected vars',
            'Model 1c: Logistic + SMOTE',
            'Model 2 : Recursive Partitioning',
            'Model 2b: Recursive Partitioning + SMOTE',
            'Model 3 : MOB',
            'Model 3b: MOB (logit score)',
            'Model 4 : CHAID Tuned',
            'Model 4b : CHAID Arbitrary')

# Gini_train
models_Gini_train <- c(m1_Gini2, m1b_Gini2, m1c_Gini2,
                 m2_Gini2, m2b_Gini2,
                 m3_Gini2, m3b_Gini2,
                 m4_Gini2, m4b_Gini2)
# Gini_test
models_Gini_test <- c(m1_Gini, m1b_Gini, m1c_Gini,
                 m2_Gini, m2b_Gini,
                 m3_Gini, m3b_Gini,
                 m4_Gini, m4b_Gini)

# gini change
modesl_Gini_ch <- (((models_Gini_test - models_Gini_train)/models_Gini_train)*100) %>% round(2)

# Combine AUC and KS
model_performance_metric <- as.data.frame(cbind(models, models_Gini_train, 
                                                models_Gini_test,
                                                modesl_Gini_ch))

# Colnames 
colnames(model_performance_metric) <- c("Model", "Gini Train", "Gini Test", "%Gini change")

# Display Performance Reports
kable(model_performance_metric, caption ="Comparision of Model Performances") %>%
  kable_styling()

```
## 5. References

1. *Statistics Solutions*, CHAID: Non-parametric Analysis, 
<https://www.statisticssolutions.com/non-parametric-analysis-chaid/)> 
2. *Chuck Powell* CHAID and CARET: A Good Combo (June 6, 2018) <https://www.r-bloggers.com/chaid-and-caret-a-good-combo-june-6-2018/>
3. *Ariful Mondal* Classifications in R: Response Modeling/Credit Scoring/Credit Rating using Machine Learning Techniques <https://rstudio-pubs-static.s3.amazonaws.com/225209_df0130c5a0614790b6365676b9372c07.html#32_recursive_partitioning_for_classification> 
4. *Zeileis, Hothorn, & Hornik 2006* party with the mob: Model-Based Recursive Partitioning in R <https://cran.r-project.org/web/packages/party/vignettes/MOB.pdf>
