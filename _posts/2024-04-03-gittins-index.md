---
layout: post
title:  "Gittin's index"
description: Job finish time prediction for scheduling
mathjax: true
img:
date: 2024-04-03  +1050
---

The context is on job finish time prediction for scheduling.

Suppose there are incoming jobs, whose completion time is not known. The scheduling problem is which job should be scheduled at what time. The objective is to reduce the mean delay (sojourn time), other versions say mean Job Completion Time is reduced. 

The best method of scheduling with minimum delay is to schedule the Shortest Remaining Time job First (SRFT). But, this assumes knowledge of how much total time each job would take.

[1] classifies the scheduling task as anticipating scheduling and non-anticipating scheduling based on whether the knowledge of job time is assumed. Anticipating assumes total service time (job completion time) is known while non-anticipating assumes it is not known, therefore only using "serviced" time or time for which jobs in the queue have already run.

For anticipating jobs, as mentioned, SRPT (Shortest Remaining Processing Time) is the known best method. Non-anticipating jobs is a bit more complex, wherein the best method depends on the distribution of the job service time.

[1] again classifies the service time distribution as Decreasing Hazard Rate class or New Better than Used in Expectation class.

For Decreasing Hazard Rate class, the optimal approach is Foreground-Background method also know as the Least Attained Service (LAS) method.

For the New Better than Used in Expectation (NBUE) class, First-Come-First-Serve or any other non-preemptive approach is optimal.

Both of these, use the knowledge that a job has been executing until time t to predict how much more time it will take. At a high level, the Decreasing Hazard says that chances of job completion reduce for a longer running job. Hence a job that has run for a while could run for a longer while. The NBUE is the opposite approach, where a younger job will last longer whereas a older job will die sooner.

### Aside - Understand NBUE
From [2], NBUE is a concept on reliability, survival and demographic modelling. Basically a model to capture human life distribution, bit morbid if you think emotionally about it, instead of a cold and calculated approach.

Lifetime (T) is random variable, with a survival function and a hazard function. The hazard function at a timestamp t (H(t)) is the rate of dying at time t. Specifically, H(t).$$\delta$$t is the conditional probability of dying (or a job completing) in a small interval around t, given survival until t. For demographics, the hazard function is the force of mortality.

Expected life is E(T). Residual life $$T_x$$ is the remaining life given survival until age x. $$T_x = T - x$$. Expected residual life is basically the chance that you survive (the hazard function) every year.

The SO answer gives an example of an exponential distribution for lifetime, which has a constant hazard function (Note: How is this derived?), resulting in a constant expected residual life. Entities with exponential lifetimes do not die of old age, they die by accident. (Again: any real world example to understand this better?)

[3] talks about a similar thing in terms of risk and reliability. Exponential is a special case of Weibull when failure rate is constant over time. A constant failure rate means that the model/system has no memory of previous failures.

NBUE literally means new better than used expectation. Plugging into this equation: $E(0) \geq E(x)


# References
[1] https://www.irit.fr/~Urtzi.Ayesta/argi/gittins_questa-v7.pdf
[2] https://stats.stackexchange.com/questions/283683/what-are-nbue-new-better-than-used-in-expectation-random-variables
[3] https://extapps.ksc.nasa.gov/Reliability/Documents/170505_Risk_Failure_Probability_and_Failure_Rate.pdf
