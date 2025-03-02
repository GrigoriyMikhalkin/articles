Tags: #k8s #pod #scheduling
# Contents

1. Inside kube-scheduler
    1. What is scheduling
    2. Scheduler's queueing mechanism
    3. Scheduling process
2. Quick note on preemption and eviction
3. Let's schedule some Pods
    1. First steps
    2. Dangers of specifying NodeName field
    3. Setting resource requirements
    4. Affinity rules
        1. Node affinity
        2. Pod affinity
        3. Pod anti-affinity
    5. Pod Topology Spread
4. Qs
    1. What happens to unschedulable Pods? ([[Queueing hints]], preemption)
    2. Is Pod overhead accounted for during scheduling?
    3. What if Node on which Pod is running is no longer satisfies defined constraints?



**Important note:** *as almost everything in Kubernetes(further K8s), scheduling, as a process can be customized/extended by user to their demands. In this guide we will talk about [kube-scheduler | Kubernetes](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/) with [default plugins enabled]([Scheduler Configuration | Kubernetes](https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins)).
# 1. Inside kube-scheduler

## What is scheduling

In Kubernetes, _scheduling_ refers to *making sure that [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) are matched to [Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/) so that [Kubelet](https://kubernetes.io/docs/reference/generated/kubelet) can run them.* The central component responsible for scheduling decisions is **kube-scheduler**.

In a bird's eye view *kube-scheduler* would look like this:
![birdsview.svg](images%2Fbirdsview.svg)

*kube-scheduler* watches for newly created/updated Pods and adds them to the *scheduling queue*. The top priority item from the queue then goes through the scheduling process, which consists of *scheduling and binding cycles*. At the end, the Pod is either scheduled or going back to the *scheduler's queuing mechanism* to wait until it will be considered schedulable again.
## Scheduler's queueing mechanism

The *scheduling cycle* is synchronous, therefore Pods have to wait for their turn to be scheduled. During scheduling, if conditions specified by Pod not yet met(existence of a persistent volume/compliance with affinity rules/etc.), the Pod needs to be moved back to the waiting line. For that reason *kube-scheduler* has queueing mechanism that consists of multiple data structures serving different purposes [^]([kubernetes/pkg/scheduler/backend/queue/scheduling_queue.go at v1.32.2 · kubernetes/kubernetes](https://github.com/kubernetes/kubernetes/blob/v1.32.2/pkg/scheduler/backend/queue/scheduling_queue.go#L145)) .
![scheduling_queue.svg](images%2Fscheduling_queue.svg)

These data structures are:
* Active queue(*ActiveQ*) -- providing pods for immediate scheduling. The Pods here are either newly created or ones that are ready to be retried for scheduling. Implemented as a heap which orders Pods using plugins that implement *QueueSort* extension point. By default, `PrioritySort` plugin is used --  as the name suggests, it sorts Pods by highest priority.
* *Unschedulable* pods map -- when scheduling for Pod fails, either during scheduling or binding cycle, the Pod is considered as unschedulable and placed in this map(or straight in *BackoffQ* if move request was received, see details below). The Pods are held here until some change in a cluster happens(new node added, PV created, Pod affinity rules satisfied, etc.) that could make those Pods schedulable.
* Backoff queue(*BackoffQ*) -- holds previously unschedulable pods for a backoff period, before they are back to *ActiveQ*. The backoff period raises exponentially, depending on the number of unsuccessful scheduling attempts for that Pod.

Pods from *ActiveQ* are popped by scheduler, when it's ready to process them. Pods in *BackoffQ* and in *Unschedulable* map are waiting for certain condition(s) to happen.

Pods, that failed to be scheduled, first are placed in *Unschedulable pods map* -- from which they can move either to *BackoffQ* or to *ActiveQ* directly. Pods are moved from *Unschedulable* map on few occasions:
* *flushUnschedulablePodsLeftover* -- is the routine which is running every 30 seconds(hard-coded value [^]([kubernetes/pkg/scheduler/backend/queue/scheduling_queue.go at v1.32.2 · kubernetes/kubernetes](https://github.com/kubernetes/kubernetes/blob/v1.32.2/pkg/scheduler/backend/queue/scheduling_queue.go#L364))). It selects Pods which stay in the map longer than required amount of time(set by `PodMaxInUnschedulablePodsDuration`) and by using queueing hint [^]([Kubernetes v1.32: QueueingHint Brings a New Possibility to Optimize Pod Scheduling | Kubernetes](https://kubernetes.io/blog/2024/12/12/scheduler-queueinghint/)) determines if Pod could be schedulable again -- if so, moves it either to *BackoffQ* or to *ActiveQ*(if backoff period for Pod is ended already).
* *Move request* which can be triggered either by changes to nodes, PVs, etc.[^]([kubernetes/pkg/scheduler/eventhandlers.go at v1.32.2 · kubernetes/kubernetes](https://github.com/kubernetes/kubernetes/blob/v1.32.2/pkg/scheduler/eventhandlers.go#L341)) or by plugins[^]([kubernetes/pkg/scheduler/backend/queue/scheduling_queue.go at v1.32.2 · kubernetes/kubernetes](https://github.com/kubernetes/kubernetes/blob/v1.32.2/pkg/scheduler/backend/queue/scheduling_queue.go#L613)). When triggered it's using the same logic as *flushUnschedulablePodsLeftover*.

Pods placed in *BackoffQ* are waiting for a backoff period to end. *flushBackoffQCompleted* routine is running every second(hard-coded value [^]([kubernetes/pkg/scheduler/backend/queue/scheduling_queue.go at v1.32.2 · kubernetes/kubernetes](https://github.com/kubernetes/kubernetes/blob/v1.32.2/pkg/scheduler/backend/queue/scheduling_queue.go#L361))) which simply moves all pods that completed backoff to *activeQ*.
## Scheduling process
![scheduling_cycle.svg](images%2Fscheduling_cycle.svg)
When top priority Pod from *ActiveQ* is popped by scheduler it will go through *scheduling and binding cycles*.

First is **scheduling cycle**[^]([kubernetes/pkg/scheduler/schedule_one.go at v1.32.2 · kubernetes/kubernetes](https://github.com/kubernetes/kubernetes/blob/v1.32.2/pkg/scheduler/schedule_one.go#L138)), which is synchronous(meaning that only one Pod at the time is going through the cycle) and consists of 2 stages:
1. *Filtering* nodes on which the Pod can be deployed, based, for example, on node labels, resource utilization and so on.
2. *Scoring* the nodes returned by *filtering* stage based on preferences and optimization rules such as topology spread constraints -- to select the best option.

After decision is made in scheduling cycle, it's time for a **binding cycle**[^]([kubernetes/pkg/scheduler/schedule_one.go at v1.32.2 · kubernetes/kubernetes](https://github.com/kubernetes/kubernetes/blob/v1.32.2/pkg/scheduler/schedule_one.go#L266)) -- which running asynchronously(and allowing another Pod to go through the scheduling cycle) is responsible for notifying API server about the decision.

Fundamental part of each cycle are *extension points*, which are implemented by plugins. Basically, *kube-scheduler* implements the glue between calls to plugins, which are responsible for the actual scheduling decisions. For example, there's `NodeName` plugin which implements *Filter* extension point -- it checks if there's node name in a Pod spec and matches it to the actual node -- if this plugin is disabled, users will not be able to assign Pods to specific nodes.

The list of default plugins can be found in Kubernetes docs [^]([Scheduler Configuration | Kubernetes](https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins)).

# 2. Quick note on preemption and evictions

I prefer to formulate the concepts of *preemption* and *eviction* slightly different than in official docs[^]([Scheduling, Preemption and Eviction | Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/)).

*Preemption* is the process of freeing node from the Pods with lower priority(look into priority classes[^]([Pod Priority and Preemption | Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/))), to make space for the Pod with higher priority.

*Eviction*(which comes in different forms) is the removal of the Pod from the node. Therefore, *eviction* can be a part of *preemption* process.

Concerning the scheduler. If Pod fails to be scheduled during the scheduling cycle the `PostFilter` plugins are called[^]([kubernetes/pkg/scheduler/schedule_one.go at v1.32.2 · kubernetes/kubernetes](https://github.com/kubernetes/kubernetes/blob/v1.32.2/pkg/scheduler/schedule_one.go#L164)). By default, it's only the `DefaultPreemption` plugin.

`DefaultPreemption` goes through the nodes and checks if node preemption will allow to schedule the Pod to this node. If so, it will evict the lower-priority pods and send the currently processed Pod to be rescheduled.

The *eviction* of Pod can be done in multiple ways. For example, API-initiated eviction(for example, by calling `kubectl drain`) will use `Eviction` API[^]([Kubernetes API Reference Docs](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#create-eviction-pod-v1-core)) which will respect Pod Disruption Budget(PDB).

The eviction during node preemption works by removing `nominatedNodeName` field from evicted pods statuses, without respecting PDBs or QoS[^]([Pod Quality of Service Classes | Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)). The Scheduler's preemption process will try to respect PDBs, when selecting pods for eviction, but if Pod with PDB is the only option, it will select it.
# 3. Let's schedule some Pods

Examples in this section can be ran on local cluster provided by [kind]([kind](https://kind.sigs.k8s.io/)). This command will create the cluster: `kind create cluster --config=kind.conf`.
>[!kind]- kind.conf
>```
>kind: Cluster
>apiVersion: kind.x-k8s.io/v1alpha4
>nodes:
>   - role: control-plane
>   - role: worker
>     labels:
>       zone: west
>       disktype: ssd
>   - role: worker
>     labels:
>       zone: west
>   - role: worker
>     labels:
>       zone: east
>   - role: worker
>     labels:
>       zone: east
>```

If you want to dive deeper into runtime operations of kube-scheduler -- it's possible to increase log level by getting into control plane node(for example using `node-shell` plugin for `kubectl`: `kubectl node-shell kind-control-plane`) and then modifying `kube-scheduler` manifest -- `/etc/kubernetes/manifests/kube-scheduler.yaml`, for example with command:
`sed -i '19i \ \ \ \ - --v=10' /etc/kubernetes/manifests/kube-scheduler.yaml`

The `kube-scheduler` will be restarted automatically by control plane, after changes to manifests are applied. Although, it's important to note that `kube-scheduler` doesn't log calls to filter plugins and relies on logging on the plugin side. Which most default plugins don't do that well.
## First steps

As we discussed earlier, standard `kube-scheduler` setup has a number of default plugins activated. A bunch of those don't need any additional config in Pod spec to affect the scheduling.

Let's apply the simple manifest, without any additional rules for scheduler:
>[!simple]- Simple deployment
>```
>apiVersion: apps/v1
>kind: Deployment
>metadata:
>  name: simple
>spec:
>  selector:
>    matchLabels:
>      app: simple
>  replicas: 1
>  template:
>    metadata:
>      labels:
>        app: simple
>    spec:
>      containers:
>      - name: nginx
>        image: nginx
>```

A bunch of plugins will do some work during scheduling this Pod:
![simple_scheduling_clean.svg](images%2Fsimple_scheduling_clean.svg)
* `PrioritySort` -- is, basically, a `Less` function applied when Pod is added to queue(heap).
* `NodeUnschedulable` -- at `Filter` endpoint will filter out nodes with `.spec.unshedulable` set to `true`.
* `DefaultPreemption` -- the `PostFilter` is called only if `Filter` phase didn't found any feasible nodes for the Pod. `DefaultPreemption`, as the name suggests, tries to remove lower priority Pods to make scheduling, for Pod in processing, possible.
* During scoring, all plugins that implement extension point will be called to score feasible nodes. Shown in this example are: `ImageLocality` -- favoring nodes that already have the container image that Pod runs; `NodeResourceFit` -- by default using "least allocated"(max available resources) strategy to score nodes; `NodeResourcesBalancedAllocation` -- favors nodes with more balanced resource usage if Pod is scheduled there.
* `DefaultBinder` -- when feasible node is found, scheduler updates `nodeName` in Pod's spec.

## Dangers of specifying NodeName field
plugins: `NodeName`, `NodeUnschedulable`

The most straight-forward way of dealing with scheduling is setting the node in a Pod spec in `nodeName` field. However the behavior is somewhat unintuitive in this case.

Set the worker node to be unschedulable: `kubectl cordon kind-worker`. Then deploy the `nginx` manifest below, which specifies the `kind-worker` node in the spec. You'll see that, despite the node being unschedulable, the Pod was still deployed and running on it.

>[!specific-node]- Manifest with node name
>```
>apiVersion: apps/v1
>kind: Deployment
>metadata:
>  name: nginx
>spec:
>  selector:
>    matchLabels:
>      app: nginx
>  template:
>    metadata:
>      labels:
>        app: nginx
>    spec:
>     nodeName: kind-worker
>      containers:
>      - name: nginx
>        image: nginx

*`nodeName` is intended for use by custom schedulers or advanced use cases where you need to bypass any configured schedulers. Bypassing schedulers might lead to failed Pods if the assigned Nodes get oversubscribed. You can use [node affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity) or the [`nodeSelector` field](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector) to assign a Pod to a specific Node without bypassing the schedulers.*
##  Setting resource requirements
plugins: `NodeResourcesFit`, `NodeResourcesBalancedAllocation`
![resource_scheduling.svg](images%2Fresource_scheduling.svg)
When deploying a Pod you can request and limit the usage of following resources: *CPU*, *Memory* and *Ephemeral storage*. Resource constraints play a significant role in scheduling but also in how they will be treated when there's not enough resources on a node(known as Quality of Service [^]([Pod Quality of Service Classes | Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)))

>[!kind]- kind config
Test 2

There's 2 plugins that play the main role during scheduling here: `NodeResourcesFit`, `NodeResourcesBalancedAllocation`.

`NodeResourcesFit` filters the nodes that have all the resources that the Pod is requesting. Then during the scoring

`NodeResourcesBalancedAllocation` does additional scoring in favor of nodes that would obtain a more balanced resource usage if Pod is scheduled there.

Apply this manifest:
>[!manifest]- nginx1.26 deployment

If you search *kube-scheduler* logs(if set to appropriate level) for the Pod name, you will see something like this:
```
pod="nginx1.26" plugin="NodeResourcesFit" node="kind-worker" score=91
pod="nginx1.26" plugin="NodeResourcesBalancedAllocation" node="kind-worker" score=93
...
pod="nginx1.26" plugin="NodeResourcesFit" node="kind-worker2" score=91
pod="nginx1.26" plugin="NodeResourcesBalancedAllocation" node="kind-worker2" score=93
```

If you used the *kind config* provided above, you will see the scores for 2 worker nodes(`kind-worker` and `kind-worker2`). As there was no other pods on these nodes, which means the identical allocatable resources on each, we see that `NodeResourcesBalancedAllocation` scores both nodes the same.

Given that the nodes final score is the same, the one is chosen randomly[^]([kubernetes/pkg/scheduler/schedule_one.go at v1.32.2 · kubernetes/kubernetes](https://github.com/kubernetes/kubernetes/blob/v1.32.2/pkg/scheduler/schedule_one.go#L873)). In our case `kind-worker2` node was selected.

Now try to apply this manifest additionally:
>[!manifest]- nginx1.27 manifest

You will see different scoring for new Pod in the logs:
```
pod="nginx1.27" plugin="NodeResourcesFit" node="kind-worker" score=91
pod="nginx1.27" plugin="NodeResourcesBalancedAllocation" node="kind-worker" score=93
...
pod="nginx1.27" plugin="NodeResourcesFit" node="kind-worker2" score=83
pod="nginx1.27" plugin="NodeResourcesBalancedAllocation" node="kind-worker2" score=87
```

As we already have a pod deployed on the `kind-worker2` node -- we see that the scores given by `NodeResourcesFit` and `NodeResourcesBalancedAllocation` plugins are lower. Therefore, final score for `kind-worker` node is higher and Pod is scheduled to it.
## Affinity rules

The dictionary definition of affinity would say that it's "attractions or connection between things/ideas". So when we define affinity/anti-affinity rules in K8s, it's helpful to think about them as rules of attraction either to nodes or to pods that have certain characteristics.
### Node affinity
plugins: `NodeAffinity`

The simplest example of affinity rule for nodes is `nodeSelector` field. Let's say we have nodes in different regions, and these nodes have label `topology.kubernetes.io/zone` that carry that info:
![ssdcluster.svg](images%2Fssdcluster.svg)
```
nodeSelector:
	zone: east
```
When Pod `nodeSelector` set, scheduler will only schedule the Pod to one of the nodes that has all the specified labels.

But what if Pods should be deployed in the certain zone and it's preferred that pods are scheduled on nodes that have SSD storage. If `nodeSelector` set to:
```
nodeSelector:
	zone: east
	disktype: ssd
```
the Pod will end up unschedulable, as there's no node with SSD in the *east* zone. `nodeSelector` isn't expressive enough for selection logic with optional conditions.

For such cases *affinity rules* come to help:
>[!manifest]- SSD app manifest
>```
>apiVersion: apps/v1
>kind: Deployment
>metadata:
>  name: east-ssd
>spec:
>  selector:
>    matchLabels:
>      app: east-ssd
>  replicas: 1
>  template:
>    metadata:
>      labels:
>        app: east-ssd
>    spec:
>      affinity:
>        nodeAffinity:
>          requiredDuringSchedulingIgnoredDuringExecution:
>            nodeSelectorTerms:
>            - matchExpressions:
>              - key: zone
>                operator: In
>                values:
>                - east
>          preferredDuringSchedulingIgnoredDuringExecution:
>          - weight:1
>            preference:
>              matchExpressions:
>              - key: disktype
>                operator: In
>                values:
>                - ssd
>      containers:
>      - name: nginx
>        image: nginx
>```

In manifest above the following affinity rules are set:
```
	  affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: zone
                operator: In
                values:
                - east
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
```

There's 2 types of affinity rules:
- `requiredDuringSchedulingIgnoredDuringExecution` -- filter out nodes based on provided rules. Important note: *rules in `nodeSelectorTerms` are ORed, rules in `matchExpressions` are ANDed.* Besides `In`, other operators are available: `NotIn, Exists, DoesNotExist. Gt, and Lt` [^]([Assigning Pods to Nodes | Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#operators))
- `preferredDuringSchedulingIgnoredDuringExecution` -- scores nodes based on provided weighted rules. As was mentioned before, each plugin that implements *score* extension point will return a score for each feasible(the ones that passed *filter stage*) node. Affinity plugins(both *Node* and *InterPod*) return score based on weights provided in manifests. Therefore, in given example, it's not guaranteed, even if node with SSD is present, that it will be chosen for Pod scheduling. It will be scored and compared with other feasible nodes.

Important note: *both rules have suffix `IgnoredDuringExecution` meaning that if conditions change after Pod was already scheduled, for example the node label value will be changed, the Pod will not be rescheduled.*

### Pod affinity
plugins: `InterPodAffinity`

There's scenarios when co-location of different services is desired. For example, there could be interdependent microservices, that constantly communicate. Placing such workloads close to each other would improve performance(minimizing latency).

*Pod affinity* allows to define such constraints of the form "this Pod should (or, in the case of anti-affinity, should not) run in an X if that X is already running one or more Pods that meet rule Y, where X is a *topology domain* like node, rack, cloud provider zone or region, or similar"[^]([Assigning Pods to Nodes | Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity)).

*Pod affinity* rules are similar to *node affinity*. The names are same: `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution`. But rules have additional required field `topologyKey` -- which should point to node's label, based on which co-location is defined.
>[!app1]- Co-located app1
>```
>apiVersion: apps/v1
>kind: Deployment
>metadata:
>  name: colocated-app1
>spec:
>  selector:
>    matchLabels:
>      app: colocated-app1
>  replicas: 1
>  template:
>    metadata:
>      labels:
>        app: colocated-app1
>    spec:
>      containers:
>      - name: nginx
>        image: nginx

>[!app2]- Co-located app2
>```
>apiVersion: apps/v1
>kind: Deployment
>metadata:
>  name: colocated-app2
>spec:
>  selector:
>    matchLabels:
>      app: colocated-app2
>  replicas: 1
>  template:
>    metadata:
>      labels:
>        app: colocated-app2
>    spec:
>      affinity:
>        podAffinity:
>          requiredDuringSchedulingIgnoredDuringExecution:
>          - labelSelector:
>              matchExpressions:
>              - key: app
>                operator: In
>                values:
>                - colocated-app1
>            topologyKey: zone
>      containers:
>      - name: nginx
>        image: nginx

In these examples *Pod affinity* is defined for `app2` as:
```
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
      matchExpressions:
      - key: app
        operator: In
        values:
        - colocated-app1
          topologyKey: zone
```
Based on this, the `app2` will be scheduled to the node that has the same value in `zone` label(*topology domain*) as the one on which `app1` is deployed.
### Pod anti-affinity
plugins: `InterPodAffinity`

Another scenario is when it's preferred that Pods are deployed in different *topology domains*(for example for availability) -- so there's anti-affinity between Pods.

*Pod anti-affinity* rules are the same ones that are used for *Pod affinity*:
>[!app1]- Anti-affinity app
>```
>apiVersion: apps/v1
>kind: Deployment
>metadata:
>  name: aaapp
>spec:
>  selector:
>    matchLabels:
>      app: aaapp
>  replicas: 2
>  template:
>    metadata:
>      labels:
>        app: aaapp
>    spec:
>      affinity:
>        podAntiAffinity:
>          requiredDuringSchedulingIgnoredDuringExecution:
>          - labelSelector:
>              matchExpressions:
>              - key: app
>                operator: In
>                values:
>                - aaapp
>            topologyKey: zone
>      containers:
>      - name: nginx
>        image: nginx

```
     affinity:
       podAntiAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:
         - labelSelector:
             matchExpressions:
             - key: app
               operator: In
               values:
               - aaapp
           topologyKey: zone
```
The rule is identical to how it would be defined in *Pod affinity*, with exception that it's part of `podAntiAffinity` kind. Therefore, 2 replicas of Anti-affinity app will be scheduled to the nodes in different *topology domains*.
## Pod Topology Spread
plugins: `PodTopologySpread`
![pts_scheduling.svg](images%2Fpts_scheduling.svg)

*Pod Topology Spread(PTS)*, on the first glance, can be similar to *affinity rules*. But, in fact, it's a very different concept. *Affinity rules* are concerned with *attraction* between Pods and nodes -- or in simple terms, keeping Pods close or at a distance from each other. *PTS* is about controlling evenness of distribution of Pods across different *topology domains*.

Let's utilize the cluster from *Affinity* examples:
![ssdcluster.svg](images%2Fssdcluster.svg)
Assume we want an even distribution across *east* and *west* zones -- so that when we scale an app from 2 to 4 replicas, or 4 to 8, the same number of replicas will be running in each zone.
>[!app]- Distributed App
>```
>apiVersion: apps/v1
>kind: Deployment
>metadata:
> name: dapp
>spec:
> selector:
>   matchLabels:
>     app: dapp
> replicas: 6
> template:
>   metadata:
>     labels:
>       app: dapp
>   spec:
>     topologySpreadConstraints:
>     - maxSkew: 1
>       topologyKey: zone
>       whenUnsatisfiable: DoNotSchedule
>       labelSelector:
>         matchLabels:
>           app: dapp
>     containers:
>     - name: nginx
>       image: nginx

```
    topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: dapp
```

`maxSkew` field defines the degree of unevenness of distribution of Pods between *domains*. `whenUnsatisfiable` defines either scheduler use *skew* for filtering nodes out(`DoNotSchedule`) or for prioritizing nodes which cause min skew(`ScheduleAnyway`). For example, with `maxSkew: 1` and `DoNotSchedule` strategy, assume we already have 3 replicas distributed between *east* and *west* zones -- 2 pods in *west* and 1 in the *east*, so the skew between domains is 1. If we add another replica -- it can't be placed in *west* zone, as skew will be greater than `maxSkew`, so it has to be placed in *east* zone.

*PTS* can be useful for achieving:
- *resilience* -- if one zone is down, workload is still available, by being deployed in another zone.
- *balanced resource utilization* -- ensuring that no single *topology domain* becomes a bottleneck and keeping app close to consumers, optimizing network latency.
