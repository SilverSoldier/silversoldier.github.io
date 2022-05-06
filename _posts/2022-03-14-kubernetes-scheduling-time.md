---
layout: post
title:  "Kubernetes Pod Scheduling Time"
description: Measuring the pod creation to scheduling time in Kubernetes
img:
date: 2022-03-21  +1430
---
Requirement: Need to measure the time Kubernetes takes to schedule a pod, from the time it was submitted to Kubernetes for creation.

On searching, there might be a straightforward way of getting this number, which I was not able to use and one slightly more scripty way in case the first does not work.

## Method 1 - Prometheus Scheduler Metrics
According to a [Github Issue](https://github.com/kubernetes/kubernetes/issues/97111), the scheduler exposes a `pod_scheduling_duration` metric which is the time from when the scheduler first saw the pod to when it was finally scheduled. This seems like exactly what we need. 

[Another link](https://docs.splunk.com/Observability/gdi/kubernetes-scheduler/kubernetes-scheduler.html) showed that there are a larger set of scheduler metrics such as `scheduler_scheduling_algorithm_duration_seconds_sum` which tracks the time spent on the scheduling algorithm etc.

However, the Openshift cluster I was running these experiments on, was not exposing these metrics. I am not sure whether it is a version issue or some config needs to be enabled for this.

## Method 2 - Using Pod Spec data
In the second method, we examine the pod spec of the created pod which contains time related fields as populated by the kube api-server.

First set of timestamps are under `metadata`. Of this, `creationTimestamp` refers to when the kubernetes object was created.

Second set of timestamps are under `status`. You can observe that `status` looks like this:

```json
"conditions": [
   {
     "lastProbeTime": null, 
     "lastTransitionTime": "2022-03-11T08:51:48Z",
     "status": "True",     
     "type": "Initialized"
   },
   {                       
     "lastProbeTime": null,
     "lastTransitionTime": "2022-03-11T08:51:56Z",
     "status": "True",    
     "type": "Ready"                
   },
   {                                           
     "lastProbeTime": null,
     "lastTransitionTime": "2022-03-11T08:51:56Z",
     "status": "True",
     "type": "ContainersReady"
   },
   {
     "lastProbeTime": null,
     "lastTransitionTime": "2022-03-11T08:51:48Z",
     "status": "True",
     "type": "PodScheduled"
   }
],
...
```

We are interested in the `conditions` field, which contains multiple timestamps for events such as pod being scheduled, containers getting ready etc.

Specifically, the `PodScheduled` type's `lastTransitionTime`, represents the pod schedule time.

Now, we can get the time taken from pod creation to schedule, i.e. the pod scheduling time.

We can use `jq` to pull these values from the yaml/json of the pod.

For the creation timestamp,
```
kubectl describe pod <podname> -o json | jq '.metadata.creationTimestamp'
```

For the podScheduled timestamp,
```
kubectl describe pod <podname> -o json | jq '.status.conditions | map(select(.type | contains ("PodScheduled")) | .lastTransitionTime) | .[0]'
```

Edit: Not a week after writing this post, I needed to use `jq` elsewhere and was able to form the right command in 2 minutes using the filtering logic above.

### Script this
In case we need this number for a number of pods, say matching a label, it would be better if the finding the time difference can be scripted.

There are a couple of obstacles (aren't there always) when fixing up a bash script for the required functionality. 

The first is when converting the time output from the `jq` output, so that the 2 times can be subtracted. As always there is a [perfect SO answer](https://stackoverflow.com/questions/41793634/subtracting-two-timestamps-in-bash-script) for this, using the `date` command.
We don't need the `%N` which converts into nanoseconds, since the timestamps are only in seconds.

```
$ date '+%s' --date="2022-03-11T09:20:27Z"
1646990427
```

However, when we directly pipe the `jq` date string output to the date command as shown above, we get an error:

```
$ date '+%s' --date "$(kubectl get pod <podname> -o json | jq '.metadata.creationTimestamp')"

date: invalid date ‘"2022-03-11T09:20:27Z"’
```

It seems to be a quotes issue, so we run the output through `tr` to remove the quotes and use the output of this with the above date command.

```bash
kubectl get pod <pod_name> -o json | jq '.metadata.creationTimestamp'  | tr '\"' ' '
```

Finally, we run this over all the pods matching the label:

``` bash
kubectl get pods -l app=abc | awk '{print $1}' | tail -n +2
```

The `awk` command will get the pod names from the list of pods. When we use `get pods` with a label matcher, the first line of the output is `NAME`. We remove that using the `tail` command.

In the `tail` man page, we can see that the `+` sign indicates from the beginning of the input.

``` bash
Numbers having a leading plus (`+') sign are relative to the beginning of the input
```

Thus, the `+2` indicates that we want from the second line of the input onwards.

The final script is shown below. 

{% highlight bash linenos %}
timestamp() {
    date '+%s' --date $1
}
for pod in $(kubectl get pods -l app=abc | awk '{print $1}' | tail -n +2); do
        create_time=$(kubectl get pod $pod -o json | jq '.metadata.creationTimestamp'  | tr '\"' ' ')
        schedule_time=$(kubectl get pod $pod -o json | jq '.status.conditions | map(select(.type | contains ("PodScheduled")) | .lastTransitionTime) | .[0]' | tr '\"' ' ')
		
        diff=$(( $(timestamp $schedule_time) - $(timestamp $create_time) ))
        echo $diff
done
{% endhighlight %}

The script can further be modified to report the average time over all the pods.

NOTE: The data used in Method 2 can be pulled from Prometheus as well. Specifically, `kube-state-metrics` exposes the creation time as `kube_pod_created` and the scheduling time as `kube_pod_status_scheduled_time`. While those metrics might be easier to query (without requiring some `jq`-fu), many clusters might not have prometheus, making `kubectl` commands much more feasible than Prometheus queries.

## Problem - Seconds

However, there is one problem with this entire approach. It only gives us the time in seconds, but unless we are testing at scale, the scheduling time would probably be in milliseconds, showing up as 0.

As per the [previously mentioned link](https://docs.splunk.com/Observability/gdi/kubernetes-scheduler/kubernetes-scheduler.html), there are some metrics that show the scheduling latency in microseconds, such as `scheduler_scheduling_algorithm_latency_microseconds` that are now deprecated.

A quick check led to [kubernetes v1.18 change logs](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.18.md) which show that from v1.14 onwards the microseconds metrics are deprecated and to use the seconds ones instead.

So, as of this moment, I am still not sure how to get the pod scheduling time in milliseconds. Will update here if I find anything.
