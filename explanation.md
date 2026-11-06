# How this was built

Basically, I sent this around to a couple people - I'm seeing some opens which is hype - but in case anyone is interested, this is my summary of the project
I'm lowkey just gonna freestyle this and write off the top based on what I can remember
I'm kind of a rambly guy so throw this in Claude - it's honestly more for myself to see how well I can remember the system design 
I'll os the github sometime soon, wanted to finish the project first, also, i built out a pipeline for it to auto update, but for some reason my github student acct is taking hella time to give me the perks / give me ability to run code in cloud - so, that will happen in next few days - 
also - tbh - i have a big feeling that if u sent this full thing into claude - it can one shot whatever i do - but i feel like if gpt described it - it wouldn't be able to one shot it and would add too much noise

Generally, been interested in macro risk model for a while
I like applying what I learned in different markets - 
a while back I thought to reimplement the barra risk model after realizing how easy it was to scrape SEC data, that went relatively well, I was happy with the results (r2=50 and good vol predictions), felt I had a good understanding of how to build covariance matrices 
I then tried it with D/EM currencies and was pretty disappointed
I think I had like 9 factors and 30 currencies
If you know how the cross-sectional risk models work - 
every day you run a regression - 
assume that relationship between factor and returns is fixed
so take the returns for the 30 currencies on that day - regress that against the 9 factors
obviously with 30 sample sizes per day, it makes it very easy to overfit / not robust at all
kind of expected it to be awful for currencies, but made sense

completely separately a while back - I reimplemented some macro risk model from QuantInsight - I found one of their old papers where they described their process pretty deeply - so had a easy time doing so - I was really interested in their paper because they applied a time series risk model 
every day - do like a 250d regression for each asset separately against the macro risk factors
did a decent job reimpleemnting it, it was before the coding agents got really good, so had some kinks, but for most part was happy 

I've always kind of had this idea that - if I was a discretionary PM - I'd create some sort of automated portfolio construction thing - and all I'd care about is entry / exit - I feel most PMs have little edge in portfolio management so it's just wasting their mental bandwidth - focus on macro, focus on stock picking, etc (tbh, I've changed my mind on this and am actually pretty bearish on the idea now) - but that's besides the point - I thought to reimplement the macro risk model (recently, been trading G10 currencies) but by combining the QuantInsight time series approach to currencies - but using the Barra Risk model for advice on covariance matrix / other stuff

Basically the whole process: 
scrape 7 tickers (macro factors - ) from yahoo finance and 9 currencies (EUR, JPY, NOK, SEK, GBP, CAD, AUD, NZD, CHF all against USD)
cleaned the data - it was pretty basic cleaning IIRC - I think just like - >4std outlier, remove - also, I remember later on, I found 2 covid data points for NZD that changed its r2 from 11% to 26%, removed those 2 also, but yeah, nothing special

I'd say the main focus can be split in a few segments:
Figuring out the Factor Exposures:
So, in XS risk model, can usually think of as - we have factor loadings (each stock will have a score) and are trying to back out the factor returns and idio returns
in TS risk model, we have factor returns (the returns of the indices) and are backing out implicit currencies beta to those factors and idio returns
Every day, we run a PLS (QuantInsight did this) with 3 principal components, 250d lookback, I think maybe 90d exponentially weighted - fact check - but basically, this distill's the 7 factors into 3 PC and the regression tells us the beta between a given asset and the 3 PC + error 
from here, we know each currencies beta to the principal components and idio returns - but logically, each regression (we will run 9 TS regressions, one for each CCY) would have its own 3 PC and our goal is to find beta to the 7 macro factors - because PCA is basically just linear combination of the original 7 factors, we can implicitly back out the beta to the original 7 factors
so after doing that step, 
we have each currency's beta to the original 7 macro factors, we started off with the 7 macro factors returns, and idio returns

Before we get to next step - a simple note is - 
The whole idea of a risk mode is - we do
Covariance matrix = XFX' + d
basically, X is factor loadings, F is the factor covaraince matrix, and d is the idio vol forecast - so we care about getting factor cov matrix and idio vol forecast

Next big step is - factor covariance matrix:
there were a couple big steps
so first - estimate factor correlation matrix using EWMA - 252d half-life 
then, clean correlation matrix with PCA blending (basically corr matrix is noisy, so retrieve top PC's and then set rest as noise) 
then, estimate each factor’s variance separately - 84d half-life, apply newey west to the variances 
Combine cleaned correlation matrix and factor variances
VRA - vol regime adjustment -  if predicted factor variance has been under/over actual, then scale up/down accordingly - doesn’t matter if u do it pre or post the combining of factor variances/correlation matrix step
So far working with daily data, so now we annualize 
PSD eigenvalue repair - this was cool idea - didn't know abt this and i think it was one of the things I missed when I reimplemented quant insight long time ago - tbh, im too lazy to explain it, just search it up - it's just like - some of the numbers get skewed because of all the changes we made from, so we have to normalize them 

then, the idio vol forecast was pretty chill - pretty similar to above but simplified
estimate the residual variance w/84d EWMA, apply newey west, vol regime adjustment.

then, obv apply the XFX' + d equation to get the Cov matrix
i ran them all
results were alright, i got like 30% r2, was honestly really expecting like 60%
then, i had a brilliant idea, usually, a risk team would combine multiple risk models - 
so like, XS risk model + statistical risk model (uninterpretable) + TS risk model
usually the XS risk model is best, stat is pretty good, TS is worse - i have a special bias towards TS based strategies though - cus macro it's so common to run a regression for like rates vs some macro variable - and ngl, most of the time, there's def some bad quant practices going on, but i have enough confidence that it works so I think equities people should use time series stuff more - it also just unlocks so much data - also as i said at start, in my experience XS risk model really doesn't work for ccys / any space where you have few assets

anyways, 
so we run a PCA - i think it was just basic 252 days over the idio returns 
first, i ran with 3PC - it explained the residual returns really well
so i think PC1 was like 57%, then PC2 was like 10%, PC3 was 8%
like wow, that's better than my whole project, crazy turn of events - but not really cus the interpretability is a problem
after playing around with the PC1 stuff - i realized it was truly just US beta - 
my end goal in this is, I start out with 9 XXX-USD stuff, and then u can back out the implicit all 45 bilatteral crosses - and when I would do some of that and see the exposures - the PC1 exposure would fall to 0 - so i got a pretty good indication
this is why i turned it into 8 factors - PC2/3 I still run and it shows that the residual returns isn't entirely independent - there is def some correlation - but - wow, did not think US beta would be that extreme

then, after running all this - i think i was explaining like 70% of the variance or smth, the vol forecasts were good, everything seemed pretty good, but i feel overall, it's important to like just as a person with some background in the subject matter - just read through the results urself - see if it makes any sense beyond the headline stats
i went through them - was very surprised at how little oil was impacting the ccys - there were 2 reasons to this - a) norway had good positive beta to oil prices and b) currencies are relative, but was still a bit weaker than i would have expected
i had an interesting idea to test correlation between the 7 macro factors and the PC1 (USD market beta) - and there seemed to be pretty high correlation
so the final step i did is - we see what part of PC1 can be explained by the 7 macro factors, and then add that back to the original macro factors - so the final PC1/USD beta you see - is like PC1 stripped of correlation to the original 7 macro factors

the other thing i noticed - very weird exposures between european ccys and european equities macro factor - i think the super high correlation between the european equities ticker and the global equities ticker messed up the regression a bit
lowkey should have tested for that
but will finish it up this weekend
just wanted to write this up in case anyone cares 
but yeah, ill os the github and comment it at the top sometime in future so people can rinse the system design 




## The idea

Notes coming soon.

## Model & methodology

Notes coming soon.

## System design

Notes coming soon.

## Validation & trade-offs

Notes coming soon.
