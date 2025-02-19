---
layout: post
title:  Kale: Elastic GPU Scheduling for Online DL Model Training
description: "Paper review series 2025"
img:
date: 2025-01-15  +1045
---

Link to paper: https://dl.acm.org/doi/pdf/10.1145/3698038.3698532

Summary: The paper proposes an elastic GPU scheduling system to improve performance of online DL model training, using traffic forecasting and resource throughput modelling to estimate resource allocation.

The niche use case that this paper targets is online model training. The usual model training that we work with, reads training data from datasets. This is offline model training. Online model training, on the other hand, reads real-time data that streams from, for ex. a Kafka message platform, so that models keep evolving with the data.

With that background, the problem is that a real-time stream of data has variance in the volume of data, so resource requirement for training also has variance. This makes scaling and elasticity, a non-trivial problem. Reactive systems like threshold based systems act late, therefore they propose a proactive system.

For this, they need to predict the volume of data that will arrive, so as to predict the resource requirements and whether scaling needs to happen. Then, for that data traffic, they need to estimate the number of workers such that data processing throughput matches the traffic rate.

They assume a Parameter-Server architecture (usually used in DL systems).

Their evaluation is top-notch with a lot of interesting metrics, comparison against SOTA in 2 different aspects and finally, traces from a production level implementation.

## Architecture
To understand their solution, it is best to describe their system architecture with 4 components for each of the mini-problems they face. 

### Workload Forecaster
As explained before, this component tries to predict the data volume using past data. They use a time series prediction model (they use FedFormer, a Transformers based time series prediction model) to use past data traffic to predict volume of incoming data traffic.

### Worker Estimator
Given the prediction of data volume, what is the scaling action that should be done? This is the work of the worker estimator which is a resource estimation component. 

It models the number of GPUs versus the throughput. The goal is that the throughput should be larger than the incoming data flow traffic. They do this by having a mathematical model of the training time as a factor of the number of GPUs. They solve an optimization problem to determine the ideal number of workers.

One additional factor they provide for, is stability. Since scaling has associated costs, rapidly increasing and decreasing number of workers will introduce latency. Thus, they ensure that their prediction graph shows stable levels of prediction for a period of time before scaling.

The actual allocation of GPUs is done by an autoscaler component.

### Data shuffling
I feel that this problem is sort of introduced by their implementation itself though there is a small chance it is inherent to the online training strategy. A simple round-robin distribution of input data flows from the streaming data system to a worker could result in imbalanced loads at workers. Load balancing is a well enough studied problem in all fields, so this might be something they actually faced but not exactly a novel contribution on its own. 

They have a "Shuffler" which redirects data to other workers to ensure load balancing between workers.

## Evaluation
On of the interesting metrics they observe is lag: this the queueing time of data until it is trained. A large lag implies a large accuracy loss due to staleness of data. They have an intro graph showing how large lag affects accuracy of a recommendation system.

Other metrics are SLO violation rate, downtime (for doing scaling) and num of GPU hours (lower is better for them)

In eval section, Sec 4.4 is the most relevant (end to end performance of Kale). They compare against baselines of no-scaling (give max as needed), K8s HPA (resource usage based), Autopilot (by Google - uses a sliding window of resource usage) and Madu (workload prediction but no resource-throughput modelling).

The authors are from a company called Kuaishou which apparently runs large recommendation models on large clusters of around 8000 GPUs. In Sec 5, they run Kale on a production level and show traces of its execution. They are able to see 40% higher GPU utilization. Data processing happens 10x faster. 

While faster data processing is understandable, the higher GPU utilization is not completely intuitive to me. I guess the baseline strategy might be an average number of GPUs, which is underutilized when data traffic flow is low. Scaling down in that case, could free up GPUs which other jobs could use to scale up, increasing the overall GPU utilization. Again, it would be useful to know exactly how they are measuring the cluster GPU utilization, is it at the granularity of GPUs (whether idle or allocated) or is it within a GPU as well.

## Key Interesting Points in the paper
1. Online training: I've somehow always considered offline training as the default, so the online scaling aspect is quite interesting. The fact that data is not read from a fixed, pre-loaded dataset file but is continuously and dynamically streaming from some source like Kafka introduces a range of problems that don't occur in the offline training case.
2. Modeling component (Resource estimator): One of the key components is the worker estimator which models how many workers are needed to handle a certain throughput. They achieve this by having a mathematical model of the throughput estimate and solving an optimization problem to get the num workers to manage the incoming traffic throughput.
3. Stability management: A small but thoughtful contribution to reduce too much autoscaling. Their technique of smoothing the prediction curve before estimating workers should be reusable for other problems as well.
4. Evaluation: They had a good eval section, mostly because they compare well against multiple prior art. For accuracy related metrics they compare with 2 similar solutions (one is their past work). For scaling, they compare with other scaling strategies and evaluate over many performance metrics (SLO violations, lag, num GPUs, downtime). They also claim that they have integrated their solution with k8s. Those architecture details would have been useful if shared.
5. I didn't understand the whole deal with the data shuffling. Perhaps if I had a pipeline setup with Kafka pushing data to a model training system, I would appreciate the problems that came with it. I am just noting this down, so that just in case, in the future, if I am working on online model training and come across this problem, I will come back to this paper and maybe understand their solution better.
