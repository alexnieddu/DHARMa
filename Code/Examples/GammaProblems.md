# Playing around with gamma glmer problems
Florian Hartig  
25 Nov 2016  




This was motivated by some issues with the Gamma reported in https://github.com/florianhartig/DHARMa/issues/8


## Gamma example from SE

This is a simple example with simulated Gamma data, based on code from http://stats.stackexchange.com/questions/47502/if-using-glmm-with-gamma-distribution-do-i-need-to-transform-my-data-to-be-betwe



```r
# random effect
u <- rep(rnorm(50, 0, .1), each = 25)
# predictor
x <- rnorm(50 * 25, 0, .1)
# outcome
y <- rgamma(50 * 25, shape = 1/(1 + u + x), scale = 1)
# id
id <- factor(rep(1:50, each = 25))
```

Fitting this with the "perfect" model (no model error)


```r
m <- glmer(y ~ x + (1 | id), family = Gamma)
```

DHARMa residuals


```r
res = simulateResiduals(m)
plot(res)
```

![](GammaProblems_files/figure-html/unnamed-chunk-4-1.png)<!-- -->

For comparison - Pearson residuals


```r
plot(m)
```

![](GammaProblems_files/figure-html/unnamed-chunk-5-1.png)<!-- -->

```r
hist(residuals(m, type = "pearson" ), breaks = 100)
```

![](GammaProblems_files/figure-html/unnamed-chunk-5-2.png)<!-- -->

Works completely fine

## Gamma example from Issue 8

See https://github.com/florianhartig/DHARMa/issues/8

Data


```r
pop <- as.character(c("BF", "BF", "BF", "BF", "BF", "BF", "BF", "BF", "BF", "BF", "BF", "BF", "BF", "BF", "BF", "BF", "BF", "BF", "BF", "BF", "BF", "MA", "MA",
                      "MA", "MA", "MA", "MA", "MA", "MA", "MA", "MA", "MA", "MA", "MA", "MA", "MA", "NU", "NU", "NU", "NU", "NU", "NU", "NU", "NU", "NU", "SA",
                      "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA", "SA",
                      "SA", "SA", "SA", "SA", "SA"))

season <- as.character(c( "fall", "spring", "fall", "spring", "fall", "spring", "fall", "spring", "fall", "spring", "fall", "fall", "spring", "fall", "fall", "spring", "fall",
                          "spring", "spring", "fall", "spring", "fall", "spring", "fall", "spring", "fall", "spring", "fall", "spring", "spring", "fall", "spring", "spring", "fall",
                          "spring", "spring", "fall", "fall", "fall", "fall", "fall", "fall", "fall", "spring", "fall", "fall", "fall", "spring", "fall", "spring", "fall",
                          "spring", "spring", "fall", "fall", "spring", "fall", "spring", "spring", "fall", "fall", "fall", "fall", "spring", "fall", "fall", "spring", "spring",
                          "fall", "fall", "spring", "fall", "fall", "spring"))

id <- "2 2 4 4 7 7 9 9 10 10 84367 84367 84367 84368 84368 84368 84368 84368 84368 84369 84369 33073 33073 33073 33073 33073 33073 33073 33073 33073 80149 80149 80149 80150 80150 80150 57140 57141 126674 126677 126678 126680 137152 137152 137157 115925 115925 115925 115925 115925 115925 115925 115925 115926 115926 115926 115926 115926 115926 115927 115928 115929 115929 115929 115930 115930 115930 115930 115931 115931 115931 115932 115932 115932"
id <- strsplit(id, " ")
id <- as.numeric(unlist(id))

distance <- "0.2970136 0.2813103 0.2409127 0.2461686 0.3392629 0.3246902 0.2938654 0.4403401 0.3935010 0.8161045 0.4622339 0.5448272 0.4347536 0.3623991 0.5014513 0.3961407 0.4285523 0.5033465 0.3668231 0.4008644 0.3642039 0.5428035 0.6348236 0.5461090 0.5763835 0.4907923 0.4349144 0.4891743 0.6423068 0.4663140 0.5226629 0.4855906 0.5868346 0.6429156 0.6363822 0.7002516 2.8778679 1.9055360 3.5048864 2.0234082 1.9940036 2.4991125 2.0742525 2.4859194 2.2326559 0.5232152 0.4835573 0.4421921 0.6048358 1.0315084 0.4935351 0.5886613 0.4821023 0.9571798 0.5721407 0.5219413 0.4243556 0.6960064 0.4713459 0.6254402 0.4887114 1.0324105 0.5536996 0.5539310 0.9808605 0.4164348 0.4658780 0.4707927 0.3722258 0.3805717 0.4608752 0.5829011 0.5095774 0.4262177"
distance <- strsplit(distance, " ")
distance <- as.numeric(unlist(distance))
distdata <- data.frame(pop = pop, season = season, id = id, distance = distance)
```

Model


```r
dist.glmm <- glmer(distance ~ pop * season + (1|id), data= distdata, family = Gamma, control=glmerControl(optimizer="bobyqa",optCtrl=list(maxfun=2e5)))
```

Residuals


```r
simulationOutput <- simulateResiduals(fittedModel = dist.glmm)
```

```
## Warning in rgamma(nsim * length(ftd), shape = shape, rate = shape/ftd): NAs
## produced
```

```r
plotSimulatedResiduals(simulationOutput = simulationOutput)
```

![](GammaProblems_files/figure-html/unnamed-chunk-8-1.png)<!-- -->

OK, this throws a warning (NAs), and looks underdispersed

### Solving the NA problem

The warning is created because the standard setting in DHARMa is to re-simulate the random effects. This can in some cases produce negative values on the linear predictor, which then produces NAs with the inverse link that is the default for Gamma. 

Solutions are either to choose a link that allows negative values


```r
dist.glmm <- glmer(distance ~ pop * season + (1|id), data= distdata, family = Gamma(link = "log"), control=glmerControl(optimizer="bobyqa",optCtrl=list(maxfun=2e5)))
simulationOutput <- simulateResiduals(fittedModel = dist.glmm, use.u = F)
plot(simulationOutput)
```

![](GammaProblems_files/figure-html/unnamed-chunk-9-1.png)<!-- -->

or to not re-simulate the random effects (i.e. calculate residuals conditional on the fitted random effects),


```r
dist.glmm <- glmer(distance ~ pop * season + (1|id), data= distdata, family = Gamma, control=glmerControl(optimizer="bobyqa",optCtrl=list(maxfun=2e5)))
simulationOutput <- simulateResiduals(fittedModel = dist.glmm, use.u = T)
plot(simulationOutput)
```

![](GammaProblems_files/figure-html/unnamed-chunk-10-1.png)<!-- -->

Changing the link is a big step, so I guess one would usually choose option 2, although the fact that NAs occur does show that the data is not 100% in line with the model assumptions. If you get lots of NAs, I would think about whether a normally distributed random effect as in lme4 is still suitable for your data, or whether you should either transform the data, or move towards gamma-distributed random effects in another software (e.g. JAGS). 

### Underdispersion

That being solved, let's look closer at the residuals. As one can see above, the residuals still look underdispersed, quite strongly actually. Looking at Pearson Residuals 


```r
plot(dist.glmm )
```

![](GammaProblems_files/figure-html/unnamed-chunk-11-1.png)<!-- -->

```r
hist(residuals(dist.glmm , type = "pearson" ), breaks = 100)
```

![](GammaProblems_files/figure-html/unnamed-chunk-11-2.png)<!-- -->

Looks also underdispersed. I was considering for a while that lme4 might do something wrong while estimating the dispersion of the gamma, but couldn't find an indication of this. So I thought it would be useful to find out if such a result is possible at all. 

Consider this case based on the first example with artificially created data. The only thing I changed is replacing the rgamma by rnorm, i.e. we fit data created with a normal error with a gamma GLMM


```r
# random effect
u <- rep(rnorm(50, 0, .1), each = 25)
# predictor
x <- rnorm(50 * 25, 0, .1)
# outcome
y <- rnorm(50 * 25, 7 + u + x)
# id
id <- factor(rep(1:50, each = 25))

## load lme4 and fit model ##
require(lme4)
m <- glmer(y ~ x + (1 | id), family = Gamma, control=glmerControl(optimizer="bobyqa",optCtrl=list(maxfun=2e5)))
```

```
## Warning in checkConv(attr(opt, "derivs"), opt$par, ctrl = control$checkConv, : Model is nearly unidentifiable: very large eigenvalue
##  - Rescale variables?
```

Residuals


```r
res = simulateResiduals(m)
plot(res)
```

![](GammaProblems_files/figure-html/unnamed-chunk-13-1.png)<!-- -->

Voila, looks like the residuals we get with the real data above. That doesn't prove that the distribution should be normal, but it does seem such residuals can easily arise with the Gamma if the model is misspecified. 

Let's maybe roll back to a LMM and see what we can see


```r
dist.lmm <- lmer(distance ~ pop * season + (1|id), data= distdata)

plot(dist.lmm, resid(., scaled=TRUE) ~ fitted(.) | pop , abline = 0)
```

![](GammaProblems_files/figure-html/unnamed-chunk-14-1.png)<!-- -->

```r
plot(dist.lmm, resid(., scaled=TRUE) ~ fitted(.) | season , abline = 0)
```

![](GammaProblems_files/figure-html/unnamed-chunk-14-2.png)<!-- -->

```r
hist(residuals(dist.lmm))
```

![](GammaProblems_files/figure-html/unnamed-chunk-14-3.png)<!-- -->

```r
qqnorm(residuals(dist.lmm))
```

![](GammaProblems_files/figure-html/unnamed-chunk-14-4.png)<!-- -->

Doesn't look too bad. A bit of assymetry, some heteroskedasticity with SA, and something weird going on in NU. Checking NU data 



```r
barplot(table(distdata$season, distdata$pop))
```

![](GammaProblems_files/figure-html/unnamed-chunk-15-1.png)<!-- -->

Very unbalanced for NU .. also



```r
barplot(table(distdata$id, distdata$season))
```

![](GammaProblems_files/figure-html/unnamed-chunk-16-1.png)<!-- -->

```r
barplot(table(distdata$id, distdata$pop))
```

![](GammaProblems_files/figure-html/unnamed-chunk-16-2.png)<!-- -->

NU has very small sample size per group in the RE - this is probably pushing them to the same main value, while the other REs are strong enough.

Overall, it seems to me the Gamma just doesn't fit so well, although I'm not sure how much difference this makes. The intuition though is that a wrong dispersion could affect p-balues and CIs - maybe better go with a standard LMM, play around with some variable transformations, and could include heteroskedasticity with nlme.









