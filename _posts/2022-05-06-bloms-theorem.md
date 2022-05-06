---
layout: post
title:  "Blom's Equation on Order Statistics"
description: Estimating the Gaussian's parameters of a distribution using its top K values
mathjax: true
img:
date: 2022-05-06  +1050
---

This is a problem we had while working on our paper, Privacy-Preserving Decentralized Exchange Marketplaces, which recently got accepted at IEEE ICBC 2022.

## Problem
In the context of the paper, the problem is the following. We have a private set of orders **O**, whose order rates are hidden and are from an unknown Gaussian G, whose parameters $$\mu$$ and $$\sigma$$ are unknown. Now given the top K of those values (which are revealed for price discovery functionality of a marketplace - to allow future traders to know what bid/ask price they should use), can we (simulating an attacker or malicious entity) theoretically estimate the parameters for the underlying Gaussian G.

Removing the marketplace related terminology, what we have is a set of values generated from a Gaussian distribution G, with unknown mean and standard deviation ($$\mu$$, $$\sigma$$) and the top K of those values are known to us, in order. How closely can we estimate $$\mu$$ and $$\sigma$$?

## Order Statistics 
Order statistics is the official terminology for the top-K values mentioned before. In a set of random variables, the *r* th order statistic is the *r* th smallest value after sorting the values. The maximum, minimum, median, quartile etc. are special order statistics.
There are some papers and methods to estimate the Guassian parameters when special order statistics are known. However, in our problem, we do not have any special order statistics, only the top (or bottom) K.

## Blom's Equation
There is a Bio Medical research paper titled "Estimating the sample mean and standard deviation from the sample size, median, range and/or interquartile range" which solves a similar problem of estimating the mean and standard deviation of a Gaussian distribution using some values of the data.

They try different methods, but the method that is of interest to us, is using Blom's equation on order statistics. 
The equation in from an old paper/Ph.D disassertation (and by old, I mean very old - 1958) by Blom.

Blom's equation estimates the *r* th order statistic given r, n (the total number of values in the distribution) and the parameters of the distribution. What we want is the inverse, given the *r* th order statistic and r, we want to estimate the parameters of the distribution.

Blom's equation, where $$\Phi^{-1}$$ is the quantile function of the standard Gaussian distribution is:

$$Blom(r, n) = \mu_E - \Phi^{-1}\left(\frac{r - \alpha}{n - 2\alpha + 2}\right).\sigma_E$$

$$\alpha$$ is a constant that changes with the value of n. Blom proposed that for practical purposed we could use the value of 0.375 or as given elsewhere, $$\frac{\pi}{8}$$.

### Quantile Function
Quantile function is the inverse Cumulative Distribution Function (CDF). The CDF of a value x, with respect to some distribution on X, is the probability that X $$\leq$$ x.

$$CDF(x) = Pr(X \leq x)$$

The inverse CDF is the inverse of this function. The quantile function takes a probability p and returns x such that $$ Pr(X \leq x) = p $$.

It sounds complicated when converted to English, so I will leave it at the above equation.

## Measuring estimate
### Estimating distribution - calculating estimates for $$\mu$$ and $$\sigma$$
Coming back to the problem at hand, how do we estimate the parameters of the distribution given K *r* th order statistics of the distribution and an estimate function for the *r* th order statistic.

What we have done is create equations for each *r* th order statistic, using the known *r* th value. This gives us K equations with $$\mu$$ and $$\sigma$$. We then eliminate $$\mu$$ to get K-1 equations solving for $$\sigma$$ which we average to get the estimate for $$\sigma$$ and finally $$\mu$$. There might be some more intelligent way here, for instance, are some of the order statistic estimates closer to the true values etc., but we stick to this for now.

The next step, is that given our estimates for the distribution, how can we get a measure of how close this estimated distribution is to the original distribution.

### Entropy and KL-divergence
Entropy, from Claude Shannon, is a measure of information. It measures how many bits are needed to encode a particular information.

While the standard entropy formula of $$ -\sum{P(X_i).log(P(X_i)}$$ is defined for a discrete variable, for a continuous variable/distribution G, it is:

$$ H(G) = - \int{f(x).log(f(x)) dx $$

KL-divergence $$D_{KL}(P || Q)$$, known as relative entropy, is one of many measures of how a distribution P deviates from another distribution Q.
In information theory, we can define it as the number of bits of error needed to encode a distribution P over an encoding for a distribution Q.

A KL-divergence of 0 means that the 2 distributions have the same information.

We use KL divergence to measure how the estimated distribution differs from the true distribution. This gives us an output in bits, which don't specifically mean anything. In order to normalize the numbers, we divide it further by the entropy of the true distribution.

This gives us a score of the estimate. A score of 0, shows no deviation, which is actually bad in our case, since the malicious entity is able to fully guess the trade order values. However, this score **is not** capped at 1, since KL-divergence is not capped at the entropy of distribution Q. 

One way to think about it, is that if P and Q distributions deviate so much that it takes more bits to communicate the error bits than to encode either P or Q themselves.

Thus, this score is not a O to 1 score. In fact, it is just normalized to make it unitless and easier to compare.

## Scripts
Now that we have the necessary equations, we wanted to run some theoretical numbers. All of this was done in R.

### The pieces
First I will share the individual pieces of code.

#### Blom's equation
Blom's equation translates very easily to R

```R
A <- pi/8
blom <- function(r, n) {
	qnorm((r - A)/(n - 2*A + 1))
}
```

We note that the meat of Blom's equation, i.e. the part that is independent of $$\mu$$ and $$\sigma$$. It can also be described as Blom's equation for a standard normal distribution, i.e. a distribution with mean 0 and standard deviation 1. 
So, this is how we have defined the function `blom`.

#### Entropy and KL-divergence

```R
Entropy <- function() {
	log(2*pi*exp(1)*sigma^2)/(2*log(2))
}

# KL divergence of f(x) = estimated Gaussian from g(x) = True Gaussian
KL <- function(emu, esigma) {
	log(sigma/esigma) + (esigma^2 + (mu - emu)^2)/(2*sigma^2) -1/2
}
```

Here `mu` and `sigma` are the $$\mu$$ and $$\sigma$$ respectively of the original distribution, whereas the `emu` and `esigma` are those of the estimated distribution.

What I have done is solve for the entropy and the KL-divergence[^!] for Gaussians and between Gaussians respectively. R also makes it quite easy to perform integrations, so the original equation could also have been solved here.

TODO: Sources here for the formula

#### Estimate Function
```R
Estimate <- function(K, N){
	Values <- sort(rnorm(N, mu, sigma)) # We generate N numbers in a normal distribution with mu and sigma
	Bloms <- sapply(c(1:K), function(x){blom(x, N)}) # We pre-calculate the Blom value for a standard normal distribution 
	sigmas <- mapply(function(x,y) {(x - Values[1])/(y - Bloms[1])}, x=Values[2:K], y=Bloms[2:K]) # We calculate sigma estimates using each of the K order statistics
	esigma <- mean(sigmas) # Calculate sigma as the mean of all sigmas
	c(Values[1] - Bloms[1]*esigma, esigma) # Calculate mean by putting it back in Blom's equation and return both mean and sigma
}
```

The function `estimate` generates a random set of N numbers and calculates the estimated value of mean and standard devaiation. 
First, it generates random numbers in a Gaussian distribution using global mean and standard deviation values using the `rnorm` function.
Then, we pre-calculate Blom values as per the description in the Blom code section. This is simply to make calculations neater in the next step and also because I really like the R style of working with arrays, such as how the random number generator also takes in the number of values to be generated and the map/apply functions.

Notice that in Blom's equation, the mean is simply added to the product of quantile and standard deviation. Thus, if we subtract 2 order estimates/values (in our case the first one and every other one), we are left with only `sigma` to solve for.

$$v_2 - v_1 = (Blom(2) - Blom(1)).\sigma$$

A simple re-arrangement, results in an estimate for $$\sigma$$.
Using `mapply`, we do this over all K - 1 values and estimate sigma as the mean of all of those values. Finally, we calculate mean by plugging this sigmal back into Blom's equation.

### Full code

Since I have already detailed the entropy, KL divergence and estimate calculator functions of the code, I will quickly go over the experiment runner functions.

The `Runner` function runs `Estimate` however many times as needed. Finally, it calculates the `KL` divergence for each run and returns the average KL divergence across runs.

Finally, the runner runs the `Runner` function over all values of K and N as needed. Here K is the number of order statistics revealed and N is the total number of values in the distribution.


{% highlight R linenos %}
A <- pi/8
blom <- function(r, n) {
	qnorm((r - A)/(n - 2*A + 1))
}

Entropy <- function() {
	log(2*pi*exp(1)*sigma^2)/(2*log(2))
}

# KL divergence of f(x) = estimated Gaussian from g(x) = True Gaussian
KL <- function(emu, esigma) {
	log(sigma/esigma) + (esigma^2 + (mu - emu)^2)/(2*sigma^2) -1/2
}

# Set values for mean and standard deviation
mu <- 300
sigma <- 10 

Estimate <- function(K, N){
	Values <- sort(rnorm(N, mu, sigma)) # We generate N numbers in a normal distribution with mu and sigma
	Bloms <- sapply(c(1:K), function(x){blom(x, N)}) # We pre-calculate the Blom value for a standard normal distribution 
	sigmas <- mapply(function(x,y) {(x - Values[1])/(y - Bloms[1])}, x=Values[2:K], y=Bloms[2:K]) # We calculate sigma estimates using each of the K order statistics
	esigma <- mean(sigmas) # Calculate sigma as the mean of all sigmas
	c(Values[1] - Bloms[1]*esigma, esigma) # Calculate mean by putting it back in Blom's equation and return both mean and sigma
}

Runner <- function(K, N){
  estimates <- sapply(c(1:100), function(x) Estimate(K, N)) # Run the estimates 100 times
  KLs <- mapply(function(x,y) {KL(x,y)/log(2)}, x=estimates[1,], y=estimates[2,]) # Calculate KL divergence for each run
  mean(na.omit(KLs))
}

for(K in seq(5, 20, by=5)) {
  for(N in c(128, 256, 512, 1024)){
    runs <- sapply(c(1:10), function(x) Runner(K,N))
	cat(K, ", ", N, ",", mean(runs)/Entropy(), "\n")
  }
}
{% endhighlight %}


## Results
When we run the above code, we notice ... that it runs really fast! Actually, I was surprised that even for around 1000 * 4 * 4 runs and some non-trivial array looping in the code, the code runs in less than a second or so. R rocks!

But related to the experiment at hand, when we run the above code, we notice 2 trends, both expected and seen. The first is on increasing K. As K, the number of order statistics, or more precisely, the number of values known increases, we expect that we are able to estimate better. We also see scores (KL-divergence over entropy) decreasing . A lesser score implies lesser deviation from the true distribution which means a more precise estimate. This matches our expectation.

The second trend is on increasing N. As N, the number of values in the set increases, and K remains constant, the number of unknown values increases, so we expect our estimate to get worse. We see scores increasing, implying a more imprecise estimate.

To summarize this post, we first defined the problem of estimating a Gaussian's parameters given only some values from the distribution, albeit values whose rank we know. This as we saw, was defined as an **Order Statistic**. Next, we see Blom's magic equation which can estimate the *r* th order statistic given the parameters of the Gaussian and discuss how we can use that to do the inverse - estimate the Gaussian parameters using the *r* th order statistic. Then, we define a measure of this estimate using KL-divergence. Finally, we see the code for doing this in R and run some simulations to see if the simulations match our expectations.

## References
[^1]: KL divergence between 2 gaussians - https://stats.stackexchange.com/questions/7440/kl-divergence-between-two-univariate-gaussians
