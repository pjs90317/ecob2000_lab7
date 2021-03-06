Lab 7
================

## Econ B2000, MA Econometrics

## Kevin R Foster, the Colin Powell School at the City College of New York, CUNY

## Fall 2020

For this lab, we will shake things up – a different dataset\! We’ll
estimate a variety of models to try to predict if a person has health
insurance. Start with logit like last week then some different models:
Random Forest, Support Vector Machines, and Elastic Net (which is not
optimal for a 0/1 dependent but it works for a demonstration).

Download the NHIS data and load it into R. If you do a summary of the
data, you will see that it includes a variety of people of all ages. We
want to understand what factors make an adult more likely to have health
insurance. Some of the variable names are a bit mystifying:
disabl\_limit codes 0/1 if the person has any limitations from a
disability; RRP codes relationship to the person answering the question;
HHX, FMX, and FPX just are ID numbers; SCHIP is a children’s healthcare
system; sptn\_medical is a factor telling how much the person spent on
medical bills. This is only a tiny fraction of the information in that
survey; there are more than 1000 different variables.

The person’s earnings could use a bit of recoding,

``` r
data_use1$earn_lastyr <- as.factor(data_use1$ERNYR_P)
levels(data_use1$earn_lastyr) <- c("0","$01-$4999","$5000-$9999","$10000-$14999","$15000-$19999","$20000-$24999","$25000-$34999","$35000-$44999","$45000-$54999","$55000-$64999","$65000-$74999","$75000 and over",NA,NA,NA)
```

First decide on how you’re defining your subgroup (all adults? Within
certain age? Other?) then find some basic statistics – what fraction are
not covered? (Later go back to look at simple stats for subgroups to see
if there are sharp differences.)

Run a logit regression. An example is below.

``` r
model_logit1 <- glm(NOTCOV ~ AGE_P + I(AGE_P^2) + female + AfAm + Asian + RaceOther  
                    + Hispanic + educ_hs + educ_smcoll + educ_as + educ_bach + educ_adv 
                    + married + widowed + divorc_sep + veteran_stat + REGION + region_born,
                    family = binomial, data = data_use1)
```

First you should think about what subset of data that you’ll use. So
change “data\_use1”.

Some of the estimation procedures are not as tolerant about factors so
we need to set those as dummies. Some are also intolerant of NA values.
I’ll show the code for the basic set of explanatory variables, which you
can modify as you see fit.

``` r
d_region <- data.frame(model.matrix(~ data_use1$REGION))
d_region_born <- data.frame(model.matrix(~ factor(data_use1$region_born)))  # snips any with zero in the subgroup
dat_for_analysis_sub <- data.frame(
  data_use1$NOTCOV,
  data_use1$AGE_P,
  data_use1$female,
  data_use1$AfAm,
  data_use1$Asian,
  data_use1$RaceOther,
  data_use1$Hispanic,
  data_use1$educ_hs,
  data_use1$educ_smcoll,
  data_use1$educ_as,
  data_use1$educ_bach,
  data_use1$educ_adv,
  data_use1$married,
  data_use1$widowed,
  data_use1$divorc_sep,
  d_region[,2:4],
  d_region_born[,2:12]) # need [] since model.matrix includes intercept term

names(dat_for_analysis_sub) <- c("NOTCOV",
                                 "Age",
                                 "female",
                                 "AfAm",
                                 "Asian",
                                 "RaceOther",
                                 "Hispanic",
                                 "educ_hs",
                                 "educ_smcoll",
                                 "educ_as",
                                 "educ_bach",
                                 "educ_adv",
                                 "married",
                                 "widowed",
                                 "divorc_sep",
                                 "Region.Midwest",
                                 "Region.South",
                                 "Region.West",
                                 "born.Mex.CentAm.Carib",
                                 "born.S.Am",
                                 "born.Eur",
                                 "born.f.USSR",
                                 "born.Africa",
                                 "born.MidE",
                                 "born.India.subc",
                                 "born.Asia",
                                 "born.SE.Asia",
                                 "born.elsewhere",
                                 "born.unknown")
```

Next create a common data object that is standardized (check what it
does\! run summary(sobj$data) ) and split into training and test sets. I
have to use a very small training set to prevent my little laptop from
running out of memory. You can try a bigger value like max=0.75 or
similar. Summary(restrict\_1) will tell you how many are in the training
set vs test.

``` r
require("standardize")
set.seed(654321)
NN <- length(dat_for_analysis_sub$NOTCOV)
restrict_1 <- as.logical(round(runif(NN,min=0,max=0.6))) # use fraction as training data
summary(restrict_1)
dat_train <- subset(dat_for_analysis_sub, restrict_1)
dat_test <- subset(dat_for_analysis_sub, !restrict_1)
sobj <- standardize(NOTCOV ~ Age + female + AfAm + Asian + RaceOther + Hispanic + 
                      educ_hs + educ_smcoll + educ_as + educ_bach + educ_adv + 
                      married + widowed + divorc_sep + 
                      Region.Midwest + Region.South + Region.West + 
                      born.Mex.CentAm.Carib + born.S.Am + born.Eur + born.f.USSR + 
                      born.Africa + born.MidE + born.India.subc + born.Asia + 
                      born.SE.Asia + born.elsewhere + born.unknown, dat_train, family = binomial)

s_dat_test <- predict(sobj, dat_test)
```

Then start with some models. I’ll give code for the Linear Probability
Model (ie good ol’ OLS) and logit, to show how to call those with the
standarized object.

``` r
# LPM
model_lpm1 <- lm(sobj$formula, data = sobj$data)
summary(model_lpm1)
pred_vals_lpm <- predict(model_lpm1, s_dat_test)
pred_model_lpm1 <- (pred_vals_lpm > 0.5)
table(pred = pred_model_lpm1, true = dat_test$NOTCOV)
# logit 
model_logit1 <- glm(sobj$formula, family = binomial, data = sobj$data)
summary(model_logit1)
pred_vals <- predict(model_logit1, s_dat_test, type = "response")
pred_model_logit1 <- (pred_vals > 0.5)
table(pred = pred_model_logit1, true = dat_test$NOTCOV)
```

You can play around to see if the “predvals \> 0.5” cutoff is best.
These give a table about how the models predict.

Here is code for a Random Forest, which takes a bit of computing,

``` r
require('randomForest')
set.seed(54321)
model_randFor <- randomForest(as.factor(NOTCOV) ~ ., data = sobj$data, importance=TRUE, proximity=TRUE)
print(model_randFor)
round(importance(model_randFor),2)
varImpPlot(model_randFor)
# look at confusion matrix for this too
pred_model1 <- predict(model_randFor,  s_dat_test)
table(pred = pred_model1, true = dat_test$NOTCOV)
```

Note that the estimation prints out a Confusion Matrix first but that’s
within the training data; the later one calculates how well it does on
the test data.

Next is Support Vector Machines. First it tries to find optimal tuning
parameter, next uses those optimal values to train. (Tuning takes a long
time so skip for now\!)

``` r
require(e1071)
# tuned_parameters <- tune.svm(as.factor(NOTCOV) ~ ., data = sobj$data, gamma = 10^(-3:0), cost = 10^(-2:1)) 
# summary(tuned_parameters)
# figure best parameters and input into next
svm.model <- svm(as.factor(NOTCOV) ~ ., data = sobj$data, cost = 10, gamma = 0.1)
svm.pred <- predict(svm.model, s_dat_test)
table(pred = svm.pred, true = dat_test$NOTCOV)
```

Here is Elastic Net. It combines LASSO with Ridge and the alpha
parameter (from 0 to 1) determines the relative weight. Begin with alpha
= 1 so just LASSO.

``` r
# Elastic Net
require(glmnet)
model1_elasticnet <-  glmnet(as.matrix(sobj$data[,-1]),sobj$data$NOTCOV) 
# default is alpha = 1, lasso

par(mar=c(4.5,4.5,1,4))
plot(model1_elasticnet)
vnat=coef(model1_elasticnet)
vnat=vnat[-1,ncol(vnat)] # remove the intercept, and get the coefficients at the end of the path
axis(4, at=vnat,line=-.5,label=names(sobj$data[,-1]),las=1,tick=FALSE, cex.axis=0.5) 

plot(model1_elasticnet, xvar = "lambda")
plot(model1_elasticnet, xvar = "dev", label = TRUE)
print(model1_elasticnet)

cvmodel1_elasticnet = cv.glmnet(data.matrix(sobj$data[,-1]),data.matrix(sobj$data$NOTCOV)) 
cvmodel1_elasticnet$lambda.min
log(cvmodel1_elasticnet$lambda.min)
coef(cvmodel1_elasticnet, s = "lambda.min")

pred1_elasnet <- predict(model1_elasticnet, newx = data.matrix(s_dat_test), s = cvmodel1_elasticnet$lambda.min)
pred_model1_elasnet <- (pred1_elasnet < mean(pred1_elasnet)) 
table(pred = pred_model1_elasnet, true = dat_test$NOTCOV)

model2_elasticnet <-  glmnet(as.matrix(sobj$data[,-1]),sobj$data$NOTCOV, alpha = 0) 
# or try different alpha values to see if you can improve
```

When you summarize, you should be able to explain which models predict
best (noting if there is a tradeoff of false positive vs false negative)
and if there are certain explanatory variables that are consistently
more or less useful. Also try other lists of explanatory variables.
