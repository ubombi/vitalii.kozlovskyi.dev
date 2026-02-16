---
title: "Intrinsic Lag Debugging"
date: 2025-09-15T11:57:18+01:00
summary: "The story about Beta versions, K8S resources, CPU pinning and scheduler itself.
"
tags:
- K8S
- Latency
- Valkey
- Linux
- CFS
- EEVDF
ShowToc: true
TocOpen: false
---

The story about Beta versions, K8S resources, CPU pinning and scheduler itself.

**TL;DR:** Guaranteed QoS isn't necessarily guaranteed. Burst workloads stall neighbours for 100ms+.
Static CPU policy and/or newer kernel (EEVDF scheduler) fix it.

## Problem & First steps

During [Valkey](https://valkey.io/) cluster deployment, I've got reports about some failed requests where latency matters.
Can't post much details, but Valkey V8 was 
[not stable](https://github.com/valkey-io/valkey/issues/2017)
and V9 was still in beta, thus was first target to blame and investigate. And K8S cluster runs on baremetal, allowing whole class of additional problems.

It's hard to believe that in-memory database, with crazy things like RDMA for cluster speedup can do it, but it's a starting point nevertheless

### Maybe network. Network always fails

Well, not at this time. Quick test would be `mtr` from app servers to valkey servers for 20minutes,
to cover few latency spike intervals. By default `mtr` uses tiny packets,
so I increased it to maximum that can pass VXLAN based CNI, 1450b.

```bash
# mtr accepts hosts list and can do multiple reports, but it does them sequentially. 
# so I just run them like so, 
mtr -s 1450 -r -c 1200 valkey1.local > mtr_1.txt &
mtr -s 1450 -r -c 1200 valkey2.local > mtr_2.txt &
mtr -s 1450 -r -c 1200 valkey3.local > mtr_3.txt &
wait
```

**reports were fine, it's not network.**


### Is it Valkey? It can't be...

Similarly to Redis, Valkey has great documentation and similar tooling. 
Here we are interested in:
* **[latency monitoring](https://valkey.io/topics/latency-monitor/)** - Req timing, typical metrics.
* **[latency](https://valkey.io/topics/latency/)** page, to debug external problems.

After following those suggestions, disabling hugepages, completely disabling `aof` _(persistency)_, I enabled monitoring, 

```
CONFIG SET save ""
CONFIG SET appendonly no
CONFIG SET latency-monitor-threshold 50
```

And sure enough, latency was real. Not just real, it `LATENCY DOCTOR` showed >100ms spikes.
It's bad, but at least problem is reproducible. 


### But is it Valkey? 

**TL;DR: No**
Built in test for latency caused by external causes, showed 250ms spikes rather quickly. **Thanks Valkey.**
```
./valkey-cli --intrinsic-latency 1200
```

**But what does it mean?**

These tools spin-wait threads, measuring scheduling jitter. So, in some cases we have process paused by OS scheduler for 250ms.


## Intrinsic Latency

We need to find why we have 250ms intrinsic lag spikes, that stall Valkey processes.

Quick check for "bad neighbours" containers didn't show anything suspicious. Clean nodes with basic daemonset containers.

### Kubernetes node (host)

With no better ideas at a time, I jumped into nodes where valkey was running.
```
kubectl debug node/foo1 -it --image=ubuntu --profile=sysadmin
```

The plan was to inspect host processes, CPU throttling, and hardware logs.


To my big surprise, I immediately saw CPU burning with work.

{{< figure
	src="/media/latency/high-cpu-illustration.png"
	alt="Illustration showing high CPU usage. "
  caption="Illustration"
>}}

While still in denial, looking at my laptop screen, it dropped back to normal

{{< figure
	src="/media/latency/normal-cpu-illustration.png"
	alt="Illustration showing normal CPU usage. "
  caption="Illustration"
>}}



### Noisy neighbour. Classic.

It didn't take much time to find that problem was caused by CI workers with some CPU intensive work. But catching an offender isn't enough.  
In what world of holy containers, would it cause 1/4 of second lags. Is cgroups resource quotas a joke?

## Little lie of K8S 

So, when I deployed valkey, I configured it with 4-8 CPU cores without much thinking about QoS. 
```yaml
---
#  Burstabl QoS: requets < limits
resources:
  requests:
    memory: "4G"
    cpu: "4"
  limits:
    memory: "8G"
    cpu: "8"
---
# Guaranteed QoS: requests == limits
resources:
  requests:
    memory: "8G"
    cpu: "8"
  limits:
    memory: "8G"
    cpu: "8"
```

K8S has `Guaranteed` > `Burstable` > `BestEffort` QoS classes

> Pods that are Guaranteed have the strictest resource limits and are least likely to face eviction.
> They are guaranteed not to be killed until they exceed their limits or there are no lower-priority
> Pods that can be preempted from the Node. They may not acquire resources beyond their specified limits.
> These <mark>Pods can also make use of exclusive CPUs using the static CPU management policy.</mark>  
> -- K8S [docs](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/#guaranteed)

### Thundering Herd

While scheduled CI workloads may run many minutes or longer, problematic lags were happening only for a very short period.
Monitoring CPU utilization live, with intrinsic latency tool on second monitor, I realized that it was happening only when 
"noisy neighbour" workload goes from idle to full throttle on all CPU cores.

This is a thundering herd -- dozens of threads spawn at once, saturating the runqueue before cgroups quota kicks in.
While the scheduler gives new processes their initial share of timeslices, for a brief moment guaranteed workloads get starved.

Any burstable workload can trigger this by simply spawning threads that burn 100% of CPU cycles.


### "Guaranteed" QoS does not guarantee

After switching Valkey pods to Guaranteed QoS, I quickly went back to intrinsic-latency checker.  
And no success. When next heavy container was scheduled, it showed 100-200ms lags.

Let's check what that static policy is about.

> There are two supported policies:
>
> * none: the default policy.
> * static: allows pods with certain resource characteristics to be granted increased CPU affinity and exclusivity on the node.  
> -- K8S docs

Let me rephrase it:

By default `none`, Guaranteed QoS is more of a quota suggestion passed to cgroups.  
With `static`, Guaranteed workloads are pinned to dedicated cores, like `taskset -c`. Everything else runs on the remaining cores with no interference (allegedly).


### The fix

**TL;DR: It solved a problem, in PoC tests**
If you work with PvP Games, RTB, VoIP... You may benefit from it.

But I couldn't redeploy K8S clusters, since teammates were afraid of this "satanic" unproven tech, from official K8S repo :).


## The experiment

To pin point a problem and find a solution, I needed simpler test setup.
I needed to compare `none` and `static` cpu policies.

And since I recalled year old articles about EEVDF scheduler quickly replacing CFS,
decided to compare those too. So: `none + CFS`, `static + CFS`, `none + EEVDF`, `static + EEVDF`.


### Test setup:

I needed controlled K8S with and without static CPU policy.

#### Latency test
cyclictest with 1s polling interval. Latency bug is most occurring in such setup. Idle workload and spiky noise.  Tried both testing all threads and guaranteed threads.
```bash
cyclictest -q -D 10m -t 4 -i 1000000
```

#### Load Generator
stress-ng with partial CPU load 5-50% and 5s slice. `cpu-load` is like PWM duty cycle.
```bash
# All CPU cores per VM
stress-ng --cpu 8 --cpu-load 50 --cpu-load-slice 5000 -t 600s
```
Those parameters define the spiky load, exactly like one caused latency spikes.

{{< figure
	src="/media/latency/spiky_load.png"
	alt="Example of stress-ng cpu-load duty cycle working. Graph shows periodic 100% load of specific cores with idle in between"
  caption="stress-ng command result. CPU is periodically 100% loaded on host"
>}}
{{< figure
	src="/media/latency/spiky_load.gif"
	alt="Example of stress-ng cpu-load duty cycle working. The htop animation"
  caption="stress-ng command result. HTOP within VM"
>}}

The goal is to create those `idle->overload` transitions, that bypass quota and cause our lag.


#### Kubernetes

To make it easier to reproduce, I went with minikube VMs. 4 Nodes each, just because I can

The `vanilla` clusters -- with default settings, including `none` CPU policy.
```bash
minikube start -n 4 --cpus 8 --memory='32G' --vm-driver kvm2 -p vanilla --kubernetes-version=v1.34.0
```

The `static` clusters -- with `static` CPU policy.
```bash
minikube start -n 4 --cpus 8 --memory='32G' --vm-driver kvm2 -p static --kubernetes-version=v1.34.0 \
    --extra-config=kubelet.cpu-manager-policy=static \
    --extra-config=kubelet.kube-reserved='cpu=1,memory=500Mi,ephemeral-storage=1Gi' 
```

CFS scheduler tested with K8S v1.31.3 and kernel v5.10.207  
EEVDF scheduler tested with K8S 1.34.0 and kernel v6.6  

In later tests CPU cores were statically pinned to host physical cores with virsh.


To run my commands within K8S, i used `quay.io/container-perf-tools/`. They have multiple containers, including:
```
quay.io/container-perf-tools/stress-ng
quay.io/container-perf-tools/cyclictest
```

### Tests

This section contains deployment files and results, in case my conclusions are wrong.
The goal was to find robust protection against intrinsic lag, not to prove anything.


#### 01 & 02 - Failed attempts

Test 01 and 02 didn't work. QoS works per pod, thus all containers must have `resources.requests==resources.limits` to make it work.
Without Guaranteed QoS, results are meaningless.


#### 03 - CFS - Does core-pinning help?

Results summary: `none` worst max **13ms**, `static` worst max **9.8ms**.
Pinning works, but difference is smaller than expected.

{{% toggle 
	title="This confirms that `static` scheduler policy does CPU pinning." 
%}}
{{< figure
	src="/media/latency/taskset_works.png"
	alt="htop screenshot with some cores 100% loaded, but dedicated cores idle. Taskset works"
  caption="regardless of load our 4 guaranteed cores remain idle"
>}}
{{% /toggle %}}
Yet, somehow no drastic difference. Perhaps that 1min delay is all it takes to ruin results 


{{% toggle title="deployment.yaml" %}}
```yaml
---
# this splits deployment into separate pods, as static does not work on a container level
# also adds antiaffinity and affinity
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpulag-stress
  labels:
    app: cpulag-stress
spec:
  replicas: 5
  selector:
    matchLabels:
      app: cpulag-stress
  template:
    metadata:
      labels:
        app: cpulag-stress
    spec:
      containers:
        - name: stress-ng
          image: quay.io/container-perf-tools/stress-ng
          imagePullPolicy: IfNotPresent
          resources:
            requests:
              memory: "1G"
              cpu: "2"
            limits:
              memory: "1G"
              cpu: "8"
          env:
          - name: tool
            value: "stress-ng"
          - name: CMDLINE
            value: "--cpu 8 --cpu-load 50 --cpu-load-slice 5000 -t 600s"
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cpulag-stress
            topologyKey: topology.kubernetes.io/hostname
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpulag-test
  labels:
    app: cpulag-test
spec:
  replicas: 5
  selector:
    matchLabels:
      app: cpulag-test
  template:
    metadata:
      labels:
        app: cpulag-test
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cpulag-test
            topologyKey: topology.kubernetes.io/hostname
      containers:
        - name: cyclictest
          image: quay.io/container-perf-tools/cyclictest
          imagePullPolicy: IfNotPresent
          resources:
            requests:
              memory: "200Mi"
              cpu: "4"
            limits:
              memory: "200Mi"
              cpu: "4"
          command: ["/bin/sh"]
          args:
            - "-c"
            - 'cyclictest -q -D 10m -t 4 -i 1000000 && sleep infinity'
          securityContext:
            # Required for access to /dev/cpu_dma_latency and /sys/kernel/debug on the host
            privileged: true
```
{{% /toggle %}}


{{% toggle title="`none` results" %}}
```
# /dev/cpu_dma_latency set to 0us
T: 0 (    8) P: 0 I:1000000 C:    600 Min:     26 Act:   60 Avg:   85 Max:     420
T: 1 (    9) P: 0 I:1000500 C:    599 Min:     22 Act:   65 Avg:   82 Max:     171
T: 2 (   10) P: 0 I:1001000 C:    599 Min:     17 Act:   59 Avg:   82 Max:     218
T: 3 (   11) P: 0 I:1001500 C:    599 Min:     18 Act:   59 Avg:   85 Max:    2572
# /dev/cpu_dma_latency set to 0us
T: 0 (    8) P: 0 I:1000000 C:    600 Min:     48 Act:   63 Avg:   87 Max:     212
T: 1 (    9) P: 0 I:1000500 C:    599 Min:     53 Act:   63 Avg:   92 Max:     322
T: 2 (   10) P: 0 I:1001000 C:    599 Min:     56 Act:   60 Avg:  100 Max:    6121
T: 3 (   11) P: 0 I:1001500 C:    599 Min:     27 Act:   59 Avg:   84 Max:     230
# /dev/cpu_dma_latency set to 0us
T: 0 (    8) P: 0 I:1000000 C:    600 Min:     56 Act:   60 Avg:   91 Max:     346
T: 1 (    9) P: 0 I:1000500 C:    599 Min:     53 Act:   61 Avg:   87 Max:     214
T: 2 (   10) P: 0 I:1001000 C:    599 Min:     55 Act:   61 Avg:   94 Max:     739
T: 3 (   11) P: 0 I:1001500 C:    599 Min:     51 Act:   59 Avg:   86 Max:     638
# /dev/cpu_dma_latency set to 0us
T: 0 (    8) P: 0 I:1000000 C:    600 Min:     26 Act:   66 Avg:  105 Max:    4231
T: 1 (    9) P: 0 I:1000500 C:    599 Min:     43 Act:   49 Avg:   85 Max:     321
T: 2 (   10) P: 0 I:1001000 C:    599 Min:     41 Act:   58 Avg:  104 Max:    8567
T: 3 (   11) P: 0 I:1001500 C:    599 Min:     12 Act:   59 Avg:  110 Max:   13413
```
{{% /toggle %}}

{{% toggle title="`static` results" %}}
```
T: 0 (    8) P: 0 I:1000000 C:    600 Min:     48 Act:  103 Avg:   98 Max:     940
T: 1 (    9) P: 0 I:1000500 C:    599 Min:     15 Act:   95 Avg:   96 Max:     721
T: 2 (   10) P: 0 I:1001000 C:    599 Min:     13 Act:   70 Avg:   98 Max:     885
T: 3 (   11) P: 0 I:1001500 C:    599 Min:     30 Act:   96 Avg:   99 Max:     728
# /dev/cpu_dma_latency set to 0us
T: 0 (    8) P: 0 I:1000000 C:    600 Min:     25 Act:   91 Avg:  102 Max:     861
T: 1 (    9) P: 0 I:1000500 C:    599 Min:     25 Act:  120 Avg:  100 Max:     733
T: 2 (   10) P: 0 I:1001000 C:    599 Min:     23 Act:  107 Avg:  103 Max:     785
T: 3 (   11) P: 0 I:1001500 C:    599 Min:     21 Act:   86 Avg:  100 Max:     689
# /dev/cpu_dma_latency set to 0us
T: 0 (    8) P: 0 I:1000000 C:    600 Min:     19 Act:  177 Avg:  107 Max:     859
T: 1 (    9) P: 0 I:1000500 C:    599 Min:     61 Act:  101 Avg:  106 Max:     993
T: 2 (   10) P: 0 I:1001000 C:    599 Min:     20 Act:   73 Avg:  130 Max:    9813
T: 3 (   11) P: 0 I:1001500 C:    599 Min:     23 Act:  143 Avg:  106 Max:     816
# /dev/cpu_dma_latency set to 0us
T: 0 (    8) P: 0 I:1000000 C:    600 Min:     16 Act:  103 Avg:   92 Max:     571
T: 1 (    9) P: 0 I:1000500 C:    599 Min:      8 Act:  101 Avg:   93 Max:     137
T: 2 (   10) P: 0 I:1001000 C:    599 Min:     20 Act:  102 Avg:   96 Max:     177
T: 3 (   11) P: 0 I:1001500 C:    599 Min:     23 Act:   83 Avg:   95 Max:     482
```
{{% /toggle %}}


#### 04 - CFS with delay

In order to ignore potential delay for CPU core pinning, test starts 2m after deployment.
Giving kubelet enough time to configure it.

Results summary: `none` worst max **14ms**, `static` worst max **1.2ms**.
Delay helped -- static pinning now shows clear advantage.

changes: `sleep 120s && cyclictest ...`
{{% toggle title="deployment.yaml" %}}
```yaml
---
# this splits deployment into separate pods, as static does not work on a container level
# also adds antiaffinity and affinity
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpulag-stress
  labels:
    app: cpulag-stress
spec:
  replicas: 5
  selector:
    matchLabels:
      app: cpulag-stress
  template:
    metadata:
      labels:
        app: cpulag-stress
    spec:
      containers:
        - name: stress-ng
          image: quay.io/container-perf-tools/stress-ng
          imagePullPolicy: IfNotPresent
          resources:
            requests:
              memory: "1G"
              cpu: "2"
            limits:
              memory: "1G"
              cpu: "8"
          env:
          - name: tool
            value: "stress-ng"
          - name: CMDLINE
            value: "--cpu 8 --cpu-load 50 --cpu-load-slice 5000 -t 600s"
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cpulag-stress
            topologyKey: topology.kubernetes.io/hostname
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpulag-test
  labels:
    app: cpulag-test
spec:
  replicas: 5
  selector:
    matchLabels:
      app: cpulag-test
  template:
    metadata:
      labels:
        app: cpulag-test
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cpulag-test
            topologyKey: topology.kubernetes.io/hostname
      containers:
        - name: cyclictest
          image: quay.io/container-perf-tools/cyclictest
          imagePullPolicy: IfNotPresent
          resources:
            requests:
              memory: "200Mi"
              cpu: "4"
            limits:
              memory: "200Mi"
              cpu: "4"
          command: ["/bin/sh"]
          args:
            - "-c"
            - 'sleep 120s && cyclictest -q -D 10m -t 4 -i 1000000 && sleep infinity'
          securityContext:
            # Required for access to /dev/cpu_dma_latency and /sys/kernel/debug on the host
            privileged: true

```
{{% /toggle %}}



{{% toggle title="`none` results" %}}
```
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    600 Min:     27 Act:  157 Avg:   90 Max:     228
T: 1 (   10) P: 0 I:1000500 C:    599 Min:     43 Act:  144 Avg:   94 Max:    3156
T: 2 (   11) P: 0 I:1001000 C:    599 Min:     34 Act:  159 Avg:   90 Max:     243
T: 3 (   12) P: 0 I:1001500 C:    599 Min:     16 Act:   93 Avg:   91 Max:     984
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    600 Min:     45 Act:  108 Avg:   91 Max:     300
T: 1 (   10) P: 0 I:1000500 C:    599 Min:     49 Act:  149 Avg:   93 Max:     512
T: 2 (   11) P: 0 I:1001000 C:    599 Min:     55 Act:  154 Avg:   90 Max:     217
T: 3 (   12) P: 0 I:1001500 C:    599 Min:     55 Act:  149 Avg:   93 Max:    2767
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    600 Min:     52 Act:  155 Avg:   91 Max:     297
T: 1 (   10) P: 0 I:1000500 C:    599 Min:     12 Act:  114 Avg:   91 Max:     295
T: 2 (   11) P: 0 I:1001000 C:    599 Min:     57 Act:  165 Avg:  134 Max:   11658
T: 3 (   12) P: 0 I:1001500 C:    599 Min:     32 Act:  153 Avg:   88 Max:     218
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    600 Min:     60 Act:  107 Avg:   95 Max:     511
T: 1 (   10) P: 0 I:1000500 C:    599 Min:     35 Act:  158 Avg:   92 Max:     550
T: 2 (   11) P: 0 I:1001000 C:    599 Min:     55 Act:  157 Avg:  194 Max:   14156
T: 3 (   12) P: 0 I:1001500 C:    599 Min:     58 Act:  152 Avg:   96 Max:     321
```
{{% /toggle %}}

{{% toggle title="`static` results" %}}
```
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    600 Min:     47 Act:  139 Avg:  130 Max:     912
T: 1 (   10) P: 0 I:1000500 C:    599 Min:     16 Act:  147 Avg:  107 Max:    1032
T: 2 (   11) P: 0 I:1001000 C:    599 Min:     54 Act:  141 Avg:  107 Max:     942
T: 3 (   12) P: 0 I:1001500 C:    599 Min:     16 Act:  152 Avg:  113 Max:    1023
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    600 Min:     42 Act:  178 Avg:  110 Max:     596
T: 1 (   10) P: 0 I:1000500 C:    599 Min:     17 Act:  145 Avg:  104 Max:     737
T: 2 (   11) P: 0 I:1001000 C:    599 Min:     27 Act:  152 Avg:  115 Max:    1179
T: 3 (   12) P: 0 I:1001500 C:    599 Min:     29 Act:  113 Avg:  107 Max:     800
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    600 Min:     65 Act:  103 Avg:  118 Max:    1042
T: 1 (   10) P: 0 I:1000500 C:    599 Min:     29 Act:  113 Avg:  115 Max:    1042
T: 2 (   11) P: 0 I:1001000 C:    599 Min:     32 Act:  143 Avg:  103 Max:    1048
T: 3 (   12) P: 0 I:1001500 C:    599 Min:     21 Act:  154 Avg:  120 Max:     846
```
{{% /toggle %}}


Clear difference. Could be a fluke -- so test 05 adds more threads.


#### 05 - CFS with delay and more threads

To make experiment load more CI-like, here we run 40 threads with 5% duty cycle. 
stress-ng will still run 100% load, but 5% of time.
This test bombards scheduler with more threads starting and stopping at once.

Each VM has 8 cores. All cores available in vanilla/none, only 4 cores available
with static (since other 4 are guaranteed to test). 5% will use ~50% of CPU time
on static and ~25% on vanilla. Just the way, how it works.


Results summary: `none` worst max **36ms**, `static` worst max **27ms**.
More threads overwhelm even pinned cores -- single 27ms spike leaked through.

changes: `stress-ng --cpu 40 --cpu-load 5`
{{% toggle title="deployment.yaml" %}}
```yaml
---
# this splits deployment into separate pods, as static does not work on a container level
# also adds antiaffinity and affinity
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpulag-stress
  labels:
    app: cpulag-stress
spec:
  replicas: 5
  selector:
    matchLabels:
      app: cpulag-stress
  template:
    metadata:
      labels:
        app: cpulag-stress
    spec:
      containers:
        - name: stress-ng
          image: quay.io/container-perf-tools/stress-ng
          imagePullPolicy: IfNotPresent
          resources:
            requests:
              memory: "1G"
              cpu: "2"
          env:
          - name: tool
            value: "stress-ng"
          - name: CMDLINE
            value: "--cpu 40 --cpu-load 5 --cpu-load-slice 5000 -t 600s"
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cpulag-stress
            topologyKey: topology.kubernetes.io/hostname
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpulag-test
  labels:
    app: cpulag-test
spec:
  replicas: 5
  selector:
    matchLabels:
      app: cpulag-test
  template:
    metadata:
      labels:
        app: cpulag-test
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - cpulag-test
            topologyKey: topology.kubernetes.io/hostname
      containers:
        - name: cyclictest
          image: quay.io/container-perf-tools/cyclictest
          imagePullPolicy: IfNotPresent
          resources:
            requests:
              memory: "200Mi"
              cpu: "4"
            limits:
              memory: "200Mi"
              cpu: "4"
          command: ["/bin/sh"]
          args:
            - "-c"
            - 'sleep 120s && cyclictest -q -D 8m -t 4 -i 1000000 && sleep infinity'
          securityContext:
            # Required for access to /dev/cpu_dma_latency and /sys/kernel/debug on the host
            privileged: true
```
{{% /toggle %}}

{{% toggle title="`none` results" %}}
```
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     62 Act:  162 Avg:  124 Max:     282
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     45 Act:  167 Avg:  251 Max:   27976
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     20 Act:  159 Avg:  233 Max:   35931
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     56 Act:  163 Avg:  240 Max:   36065
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     36 Act:  138 Avg:  121 Max:     241
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     17 Act:  146 Avg:  113 Max:     246
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     48 Act:  189 Avg:  114 Max:     346
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     58 Act:   94 Avg:  117 Max:    1121
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     35 Act:   35 Avg:  115 Max:     253
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     45 Act:  158 Avg:  120 Max:     360
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     56 Act:  142 Avg:  116 Max:     239
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     51 Act:  137 Avg:  115 Max:     620
```
{{% /toggle %}}

{{% toggle title="`static` results" %}}
```
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     56 Act:  153 Avg:  121 Max:     205
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     42 Act:  141 Avg:  118 Max:     301
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     25 Act:  102 Avg:  122 Max:     706
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     58 Act:  144 Avg:  115 Max:     564
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     20 Act:  161 Avg:  129 Max:     611
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     31 Act:  161 Avg:  136 Max:     880
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     36 Act:  152 Avg:  304 Max:   27356
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     43 Act:  150 Avg:  128 Max:     984
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     62 Act:  149 Avg:  125 Max:     275
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     39 Act:  129 Avg:  133 Max:    1211
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     46 Act:  172 Avg:  122 Max:     245
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     30 Act:  152 Avg:  124 Max:     920
```
{{% /toggle %}}




#### 06 - EEVDF with delay and more threads

After recreation of both clusters with newer K8S and OS images.
I run same 05 test with different scheduler.
Same VMs, same physical CPU cores, same 5th test.

changes: New kernel (full OS image)
deployment.yaml - same as before

Results summary across two runs:

| | `none` worst max | `static` worst max |
|---|---|---|
| Run 1 | **2.9ms** | **1.3ms** |
| Run 2 | **4.2ms** | **4ms** _(single outlier, rest <0.5ms)_ |

{{% toggle title="`none` results" %}}
```
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     62 Act:  153 Avg:  127 Max:    2934
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     62 Act:  164 Avg:  121 Max:     437
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     47 Act:  141 Avg:  119 Max:    1972
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     46 Act:  125 Avg:  133 Max:    1579
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     26 Act:  103 Avg:  120 Max:     471
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     24 Act:   92 Avg:  120 Max:    1475
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     25 Act:  149 Avg:  130 Max:    2611
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     24 Act:  221 Avg:  117 Max:    1946
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     61 Act:  143 Avg:  121 Max:     416
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     55 Act:  113 Avg:  131 Max:    2888
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     41 Act:  143 Avg:  128 Max:    1274
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     62 Act:  156 Avg:  129 Max:    1662
```
{{% /toggle %}}
{{% toggle title="`static` results" %}}
```
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     43 Act:  147 Avg:  120 Max:     206
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     29 Act:  218 Avg:  113 Max:     276
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     13 Act:  160 Avg:  116 Max:     339
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     42 Act:  140 Avg:  117 Max:    1271
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     67 Act:  142 Avg:  120 Max:     338
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     23 Act:  163 Avg:  119 Max:     392
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     28 Act:  170 Avg:  125 Max:     477
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     66 Act:  149 Avg:  121 Max:     376
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     23 Act:   99 Avg:  117 Max:     344
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     40 Act:   94 Avg:  123 Max:     335
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     63 Act:  138 Avg:  123 Max:     216
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     35 Act:  145 Avg:  122 Max:     209
```
{{% /toggle %}}

Results surprised me. Too good to be true. So I retried:

{{% toggle title="`none` results 2nd run" %}}
```
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     59 Act:  152 Avg:  114 Max:    1758
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     59 Act:  160 Avg:  128 Max:    1175
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     43 Act:  154 Avg:  126 Max:    1975
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     53 Act:  142 Avg:  124 Max:    1053
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     38 Act:  141 Avg:  119 Max:    1273
T: 1 (   10) P: 0 I:1000499 C:    479 Min:     60 Act:  148 Avg:  122 Max:    1968
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     34 Act:  149 Avg:  135 Max:    4150
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     29 Act:  155 Avg:  133 Max:    3081
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     36 Act:  104 Avg:  111 Max:    1668
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     22 Act:  157 Avg:  117 Max:    1901
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     29 Act:  155 Avg:  118 Max:    1634
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     18 Act:  106 Avg:  122 Max:    1473
```
{{% /toggle %}}
{{% toggle title="`static` results 2nd run" %}}
```
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     28 Act:  115 Avg:  118 Max:     183
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     67 Act:  162 Avg:  116 Max:     334
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     66 Act:  143 Avg:  114 Max:     213
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     16 Act:  157 Avg:  121 Max:     203
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     23 Act:  159 Avg:  126 Max:     333
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     39 Act:  144 Avg:  126 Max:     430
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     63 Act:  157 Avg:  113 Max:     266
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     30 Act:   45 Avg:  122 Max:     504
# /dev/cpu_dma_latency set to 0us
T: 0 (    9) P: 0 I:1000000 C:    480 Min:     49 Act:  116 Avg:  112 Max:     329
T: 1 (   10) P: 0 I:1000500 C:    479 Min:     28 Act:  154 Avg:  123 Max:     278
T: 2 (   11) P: 0 I:1001000 C:    479 Min:     67 Act:  103 Avg:  131 Max:    4015
T: 3 (   12) P: 0 I:1001500 C:    479 Min:     15 Act:  156 Avg:  120 Max:     312
```
{{% /toggle %}}

### Results

| Setup | Worst tail latency (test 05/06) |
|---|---|
| CFS + `none` | ~36ms |
| CFS + `static` | ~27ms _(may be an error)_|
| EEVDF + `none` | ~4ms |
| EEVDF + `static` | ~1.3ms |

36ms -> 4ms with just a kernel upgrade. 4ms -> 1.3ms with static pinning on top.
Though static wastes compute when pod is idle, use where latency is critical.

### Static CPU manager

Mostly helps. Allocates cores to specific tasks. Reduces spikes but doesn't fully prevent them.

Has a reconciliation delay -- pods get pinned a few seconds to a minute after deploy, not instantly.

### EEVDF scheduler

Significant difference in tail latency without effect on average latencies.
Great improvement, stacks well with dedicated cores.

### big.LITTLE (P/E cores)

The strange part -- EEVDF almost solves the latency issue on Zen3 CPU.
But on a laptop with Intel P/E cores, running the same stress and cyclictest commands on a 6.6 kernel,
I reproduced the original huge spikes even with EEVDF.

No idea why. Could be P/E core scheduling, could be Chrome eating my CPU.
Laptop isn't a lab, I didn't dig deeper.

### Takeaways

- K8S `Guaranteed` QoS does **not** isolate CPU cores by default. It's a cgroups quota, not pinning.
- `static` CPU manager policy provides real isolation via `cpuset`. It solves the noisy neighbor problem but wastes cores.
- EEVDF scheduler is a massive improvement over CFS for transient latency spikes -- even without static pinning.
- If you run latency-sensitive workloads on shared K8S nodes, test with `cyclictest` before trusting QoS. 
