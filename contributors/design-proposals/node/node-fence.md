# Node remedy
Status: Pending

Version: Alpha

Implementation Owners: Yaniv Bronhaim (ybronhei@redhat.com), Jan Chaloupka (jchaloup@redhat.com)

Current Repository: https://github.com/rootfs/node-fencing

## Motivation
When a cluster size reaches a certain order of magnitude, the maintenance of each
node becomes a considerable issue. It is no longer feasible to pet each node and
investigate every situation in which the node starts to misbehave. The time spent
on the problem solving becomes a precious resource which if consumed improperly
can increase overall maintenance cost. Thus, it is desirable for the cluster
to have ability to heal itself.

Based on the same self-healing principle of applications running in a container,
the cluster should be able to detect which nodes are misbehaving and carry proper
actions that will either cure each affected node or replace it with a new instance.

The goal of the proposal is to design and provide a solution implementing self-healing mechanism that is general enough to react on various aspects of a node behavior
and at the same time simple enough to allow to run various remedy operations (either
generic or specific for a cloud provider environment).

### Known limitations

When a node stop responding (for more than a certain period of time),
the Kubernetes initializes an eviction process that will
cause pods assumed to be running on the node to enter a termination state.
However, the Apiserver and the node controller have no way of knowing if pods
on the node are still running and serving or if they are down by that time.
If the issue is caused by a network partition state between nodes and the apiserver
running on the master node (e.g. network device failure), the remedy could be to
switch to a different network without performing the eviction.

A node can show signs of misbehavior even if it is reporting a Ready status:
* A pod may need to access an AWS volume which claims it is not attached on a node
even if it actually is. The remedy here can be to restart the node so the volume
is properly detached and re-attached again (not necessarily on the same node).
* Node resources consumed by a node daemon or a container runtime environment may
grow over time without any visible reason (e.g. faulty garbage collector in Go
runtime environments or a container runtime environment loosing memory/not releasing
i-nodes) which can eventually exhaust available resource causing a node
to be inoperable. Rebooting the node can help to re-claim the resources.

When k8s is deployed on a cloud providers such as AWS or GCE, the autoscaler uses
the Cloud Provider APIs to recognize unhealthy nodes and effectively bounding
the amount of time the node will stay in the “not ready” state until the node-controller
removes the node (if possible, kernel panic can't be handled that way), and how long
the scheduler will need to wait before it can safely start the pod elsewhere.
This treatment is not immediate and not covered in bare metal deployments.
Also, there is no mechanism to recover quicker, but only to downscale the partitioned
nodes. This leads to admin intervention which we can avoid using automated mechanism.
This is particularly problematic because all StatefulSet scaling events (up and down)
block until existing failures have been recovered. From some perspectives,
it could be considered that each node hosting a StatefulSet member has become
a single point of failure for that set.

### Restrictions
Each higher-level construct built on top of pods has a different
requirements and expectations of guarantees around its life-cycle:
* If the pod belongs to a ReplicaSet, once the termination state is set, the ReplicaSet
controller will immediately start another instance of that pod (without any side-effects
as the ReplicaSet is assumed to be stateless).
* When the pod is part of a StatefulSet, the StatefulSet controller won’t start
new instance of that pod until the kubelet responds with the pod’s status.
This is because there is a permanent association between a StatefulSet member and
its storage. Not waiting would potentially result in multiple copies of a member,
all writing to the same volume leading to corruption or worse (see [pod-safety](https://github.com/kubernetes/community/blob/16f88595883a7461010b6708fb0e0bf1b046cf33/contributors/design-proposals/pod-safety.md)).
* Pet pod could also represent a VM that needs to be removed (with all its resources freed)
before a new instance is created.

The self-healing mechanism must take all the guarantees into account so the remedy operation
is performed properly.

## User experience

If possible, any node problem occurring in a cluster should be transparent to user applications.

1. As a user running a database as a StatefulSets, I would like to be able to scale
   up/down normally no matter if a node running my workload is malfunctioning
1. As a user running a production backend, I would like to have the cluster to repair
   a node in case it starts to show signs of services degradation without any manual intervention
1. Pods on a node providing HA services can suffer from a network malfunction caused be
   an incorrectly functioning network utilities on a node. Rebooting a node allows to re-schedule the pods safely.
1. In a bare metal Kubernetes deployment, StatefulSets should not require admin intervention
   in order to restore capacity when a member Pod was active on a lost worker

## Admin experience

1. As an SRE I would like to set automatic self-healing mechanism so I minimize
   the number of manual interventions.
1. As an admin/cloud operator I would like to see the current state of the remedy
   operations in a compound manner so I can monitor ability of the cloud to self-heal
   and react properly on distinctive situation (e.g. be able to force stop of fencing operation and fix problems manually).
1. As an admin/cloud provider I would like to be able to set various remedy operations
   and conditions under which the operations get performed (i.e. easy-to-configuration).
1. As an architect of a new source of node behavior notifications I would like to be able
   to extend the list of resource the controller consumes to make a better remedy decision.


In addition, problem events counter will be integrated. This allows to monitor issues
and their handling in operated cluster. Alarms can be set when issues repeated in certain period
that the admin configures (e.g. if one pod causes hardware crashes, this monitoring
view will provide overview of the cause that can be fixed manually).

## Solution proposal

Depending on cloud provider environment, each remedy operation can have different
implementation requirements:
* Cloud providers such as AWS or GCE expose a cloud API that allows to perform various machine related actions.
In most cases only credentials and an instance name are required to access the cloud provider API.
* Bare-metal cloud providers may require knowledge of hardware specific parameters such as
Eaton PDU parameters to perform power management actions or switch brocade parameters
to disconnect a node from the network.
* There may be new emerging providers that are not yet known, required parameters to be set are not yet identified
* Any combination of cloud providers as hybrid cloud providers where each subset of nodes run on a different provider

All those information increase the complexity of the self-healing mechanism.
In order to keep the complexity sane, the remedy mechanism is broken down into two pieces:
* Controller that is responsible for monitoring nodes (and/or other sources providing information about each node behavior) and initiating actions responsible for changing node states
* Implementation of individual actions as pod specifications (i.e. remedy pod) that are responsible for converging a node into its expected state

All the cloud provider specific knowledge and implementation are delegated into pods.
This way, each operator that has a specific knowledge and expertise of affected environment can provide his/her own solutions.
Given a specific remedy solution is shipped in a container image, it is also easier to update and test individual pieces
without a need to update the controller itself.
Also, the open-source communities provide a lot of functioning solutions that can be used.

At the same time the approach makes the controller transparent to a way a given remedy operation is performed.
Thus, the controller implementation can concentrate purely on problems that are related to:
- Monitoring and detection of events that a misbehaving node generates (either as a node status reporting `Unready` or a problem node detector reporting application crashes)
- Generating metrics about the self-healing process itself (e.g. how many times a node was repaired, how many times a node was terminated due to network failures)
- Decision making logic that controls when and which remedy action gets triggered

and can ignore:
- Credentials management needed to perform cloud specific operations (delegating the responsibility to an operator for figuring out the way to provide credentials)
- SSH keys management (the keys can be read from a specific URL that is available only on a specific subnet)
- Distribution of specific configuration tailored for each node (bare-metal providers require more parameters to perform power-management operations)

Some remedy operations require to be modeled as a state automaton.
Given the controller is designed to react on node misbehavior by creating a pod,
it does not keep state of any of the remedy operations internally.
This decision simplifies the overall design of the controller.
On the other hand, it shifts responsibility of making sure an operation is performed
properly to remedy pod implementation. Including retry mechanisms in case the pod fails,
is recreated and needs to start from a non-initial state.

If a remedy pod fails and a node does not converges to its expected state, the pod is recreated again.
The controller checks if there is any remedy pod running before recreated it so only one instance of a remedy pod per each node is running.

Reaction of the controller (which operation gets triggered) on a node misbehavior are stored in a `ConfigMap` (most likely as a blob that gets interpreted within the controller).
The configuration can be similar to the following one (just for illustrative means, the final form may be different):

```yaml
apiVersion: node-repair/v1alpha
kind: Configuration
# optional node selector (type=compute, type=accelerated, type=storage-optimized)
nodeSelector: type=compute
# list of thresholds tracked, and the remedy to execute if triggered
# note: you could track many node conditions other than ready
thresholdConfigs:
- nodeConditionType: Ready
  nodeConditionStatus: false
  nodeConditionPeriod: 10m
  remedy: repair
# users may create remedies for any number of things that could happen to a node
# the tool only cares if the configured job runs to success or failure
remedyConfigs:
# the repair remedy that in this case says run a job that executes a command to clean node
# the pod template should assume standard input via downward api that describes what node is being remedied
- name: repair
  podTemplate:
   spec:
    containers:
    - name: repair
      image: IMAGE:VERSION
      command: [ "binary", "clean node command" ]
    restartPolicy: Never
# the terminate remedy that in this case says run a job that executes a command to terminate node
- name: terminate
  podTemplate:
   spec:
    containers:
    - name: terminate
      image: IMAGE:VERSION
      command: [ "binary", "terminate node command" ]
    restartPolicy: Never
```

When deciding which remedy operation to perform, the controller should try to collect
more information before actually triggering the operation (e.g. based on remedy policies).
For example:

1. Switch failure - a switch used to connect a few nodes to the environment is failing,
and there is no redundancy in the network environment. In such a case,
the nodes will be reported as unresponsive, while they are still alive and kicking,
and perhaps providing service through other networks they are attached to.
If we fence those nodes, a few services will be offline until restarted,
while it might not have been necessary.

1. Management system network failure - if the management system has connectivity issues,
it might cause it to identify that nodes are unresponsive,
while the issue is with the management system itself.

The design and implementation also need to acknowledge that other entities,
such as the autoscaler, are likely to be present and performing similar monitoring and recovery actions.
Therefore it is critical that the remedy controller does not interfere or is to be susceptible to race conditions.

Other sources informing about a node behavior can be utilized:
* **Node problem detector**: Daemon which runs on each node, detects node problems and reports them to apiserver.
The project lives under [https://github.com/kubernetes/node-problem-detector/](https://github.com/kubernetes/node-problem-detector/) repository.
The node problem reports can be used to detect different misbehavior than just node going into "Unready" status.
* **Kdump integration**: When kdump enabled is the controller can check for kdump notifications once node
becomes not ready. Once dumping is recognized, the controller can use the notification
to make a better conclusion on which remedy operation needs to be run.


### Remedy policies
The policies allows to regulate remedy operations performed throughout the cluster such as:

* How many times a remedy operation gets performed over a single node per a time period (e.g. backoff policy)
* Upper bound on a number of nodes that can be remedied simultaneously (e.g. a node isolation performed over a set of nodes can lead to a "fencing storms")
* Maximum number of retries a remedy pod gets recreated before the controller gives up (can be set to "never give up")
* Limit the amount of resources (e.g. nodes) removed per a time period
* Limit the set of nodes to be self-healed (e.g. master nodes should not be restarted without an admin intervention)
* Set a waiting period before a node is processed (in case a node manages to fix itself on its own)

In some cases it is required to perform only a single node remedy at a given time
to prevent a cluster from draining and recreating a lot of pods at the same time.

NOTE: Fencing storm is a situation in which fencing for a few nodes in the cluster
is triggered at the same time, due to an environmental issue.
Some ways to prevent fencing storms:
* Skip fencing if select % of hosts in cluster is non-responsive (will help for issue #2 above)
* Skip fencing if detected the host cannot connect to storage.

## Remedy pods

A remedy pod is the smallest operational unit of work that gets created as a reaction
to a node misbehavior. Its purpose is to provide a solution that encompasses all the logic
needed to remedy a node.

Depending on the characteristics of each misbehavior, the remedy pod can perform combination of:
* Power management fencing: powering off\on, rebooting a node
* Storage fencing: disconnection from specific storage to prevent multiple writers, and unfence when connectivity is restored
* Cluster fencing: Cordon node, removing node workload
* Network isolation
* Other environment specific actions (e.g. running diagnostics of a node)

The remedy operations can be broken into two categories:
- Stateless operations: the operation is restarted from the beginning no matter where the previous run failed
- Stateful operations: the operation needs to resume in the last state in which the previous operation failed

Currently, there is no plan to make the controller aware of the stateful operations.
Though, some logic may be added later to support the case.

### Cloud provider remedy pods

Currently, the following cloud providers are identified:
* AWS
* GCE
* OpenStack
* Azure
* WmWare

NOTE: the list is not meant to be a complete enumeration of all available providers

Each remedy pod needs to know how to access a specific cloud provider API and
what parameters are needed to perform selected action.
There are multiple ways how to provide such parameters to each remedy pod:

* Per node configuration (e.g. read from a ConfigMap)
* Per set of nodes configuration (e.g. read from a ConfigMap)
* Explicit assumed configuration common for all nodes
* Centralized place that can by queried from within a running pod (e.g. URL metadata link, cloud database)

In any case we need to minimize the remedy pod configuration complexity.
E.g. the node configuration fallback mechanism can be used (e.g. the per set of nodes configuration overrides the default one).

#### Bare-metal cloud providers

As the bare-metal provider requires deeper knowledge of the environment,
some operations require to be implemented as a sequence of steps.
Each step corresponding to a state that needs to be kept in the process in case a failure
occurs and the process needs to be resumed from the last state.

**Example**: Some bare-metal operations can be modeled as a state automaton.
The automaton can be defined as an n-tuple (S, T, r, new, F), where `S` denotes a set of states,
`T` a set of fence steps, `r` stands for a transition function, ``new`` is its initial state
and ``F`` a set of final states. The set `S` consists of:
* ``new``: a node gets into the state in a moment it starts misbehaving (e.g. node reporting `Unready` status)
* ``running``: a node is isolated and ready for power management actions
* ``done``: a node is ready for recovery
* ``final``:  node is repaired and joined back to the cluster
* ``error``: either of the fence steps failed

The transition function ``r`` is defined as:
* When a node is in the ``new`` state for a specified number of times units,
the ``isolation`` step is triggered. Once the step is completed,
the node transitions to the ``running`` state. In case the step is not completed,
the node transitions to the ``error`` state.

* When a node is in the ``running`` state, check if the node status is ``Ready``.
If it is, node transitions to the ``done`` state.
Otherwise the ``power-management`` step is triggered.
If the step fails, the node transitions to the ``error`` state. Otherwise
the node transitions to the ``done`` state.

* When a node is in the ``done`` state, check the node status. If the status is
``Ready``, run the ``recovery`` step. Otherwise, transition to the ``error`` state.
If the ``recovery`` step is completed, move the the ``final`` state. Otherwise,
transition to the ``error`` state.

If any of the steps fail, it can be retried a configurable number of times before
the node transitions into the ``error`` state.
If a node gets into the ``error`` state, entire execution is restarted and the
node is moved to the ``new`` state. The number of restarts is configurable as well.

## Roadmap

Underway:

* Implement the remedy controller monitoring node status
* Specify a remedy configuration covering the most common use cases
* Implement a remedy pod self-healing a node in AWS/GCE environment
* Implement the fence flow mechanism for bare-metal support (focus on 1 or 2 PM devices)

Would like to get soon:

* Demonstrate the remedy mechanism on AWS and GCE (optionally OpenStack)
* Demonstrate the mechanism on a bare metal provider (demonstrating how to work with node specific parameters)
* Generate remedy metrics (how many times a network related remedy operation was run, how many times a node was restarted, how many times a node was removed)
* Cluster fence policies applied

Other possibilities:

* Add mechanics to ease running of stateful remedy operations (allow to store last known state)
* Consume events from the [Node problem detector](https://github.com/kubernetes/node-problem-detector)
* Consume events/data from other source providing information about node behavior
