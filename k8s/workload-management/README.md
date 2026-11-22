# Kubernetes Workload Management

## Table of Contents

- [How they relate](#how-they-relate)
- [Side-by-side comparison](#side-by-side-comparison)
- [What each layer does](#what-each-layer-does)
- [Which one should I use?](#which-one-should-i-use)
- [Official references](#official-references)

A **Pod** is the unit that actually runs containers. A **ReplicationController**, **ReplicaSet**, or **Deployment** is a controller that works to keep the requested Pods available and, depending on the controller, manage updates.

## How they relate

```text
Pod created directly:
  Pod

ReplicationController (legacy):
  ReplicationController -> Pods

ReplicaSet:
  ReplicaSet -> Pods

Deployment:
  Deployment -> ReplicaSet(s) -> Pods
```

A ReplicaSet is the newer successor to a ReplicationController; it does not manage or wrap one. A Deployment manages ReplicaSets, and each ReplicaSet manages its Pods. During a Deployment rollout, the Deployment may keep both the new and previous ReplicaSets for a time.

## Side-by-side comparison

| Feature | Pod | ReplicationController | ReplicaSet | Deployment |
| --- | --- | --- | --- | --- |
| **API version** | `v1` | `v1` | `apps/v1` | `apps/v1` |
| **Core purpose** | Runs one or more related containers together. | Maintains a target count of matching Pods. | Maintains a target count of matching Pods. | Manages ReplicaSets, Pod updates, and rollout history. |
| **Self-healing** | No replica controller by itself. The kubelet can restart a container in the same Pod, but a deleted or lost Pod is not replaced. | Yes. Creates a replacement when its matching Pod count is below the target. | Yes. Creates a replacement when its matching Pod count is below the target. | Yes, through its active ReplicaSet. |
| **Scaling** | No `replicas` field; create/manage separate Pod objects yourself. | Set the desired replica count. | Set the desired replica count. | Set replicas on the Deployment or let an autoscaler manage them. |
| **Label selectors** | Has labels but does not select or manage other Pods. | Equality-based selectors, such as `app=web`. | Equality and set-based selectors, such as `matchExpressions` with `In` or `NotIn`. | Uses an `apps/v1` label selector to identify Pods through its ReplicaSets. |
| **Updates and rollback** | No controller-driven rollout. You manage the Pod directly; changing its image can restart its container, while many other spec changes require recreating the Pod. | No Deployment-style declarative rollout or revision history. Updates historically require manual coordination. | A changed template is used for future replacements; the ReplicaSet does not roll existing Pods through versions or keep rollout history. | Built-in rollout strategies and revision history, including rollback. Rolling updates can preserve availability when the strategy, readiness checks, and capacity allow; zero downtime is not guaranteed. |
| **Typical use** | Learning, debugging, or a one-off workload. | Legacy manifests; avoid for new workloads. | Usually created and managed by a Deployment, not directly. | Standard choice for stateless application workloads. |

The word **self-healing** needs a distinction: a kubelet may restart a failed container inside a Pod, but only a higher-level controller such as a ReplicaSet can create a replacement Pod when the Pod itself is missing. A Deployment delegates that replica repair to its ReplicaSet.

## What each layer does

1. **Pod — the workload instance.** Holds one or more containers, their shared Pod network, and any declared volumes. A Pod is scheduled as one unit. A standalone Pod has no replica controller watching it.
2. **ReplicationController — the legacy supervisor.** Watches matching Pod labels and creates or deletes Pods to move toward its desired count. It supports equality-based selectors; use it only when maintaining older workloads.
3. **ReplicaSet — the current replica supervisor.** Also reconciles a desired Pod count, and adds set-based selector support. It creates Pods from its template, but changing that template does not perform a rollout across existing Pods. Use it directly only when you need custom orchestration or do not need rollout behavior.
4. **Deployment — the release manager.** Owns ReplicaSets and coordinates application updates. When its Pod template changes, it creates a new ReplicaSet and adjusts old and new replica counts according to the configured strategy. The default strategy is `RollingUpdate`; `Recreate` is also available. Rollback uses retained Deployment revision history.

The controller's selector identifies the Pods it manages, and its Pod template describes new Pods. The template's labels must satisfy the selector. Avoid overlapping selectors between controllers: multiple controllers can act on the same Pods and conflict.

## Which one should I use?

- Use a **Deployment** for most stateless applications such as APIs, web frontends, and microservices.
- Use a **ReplicaSet** directly only for a specialized controller workflow that does not need Deployment rollouts.
- Use a **ReplicationController** only to maintain a legacy workload.
- Create a **standalone Pod** for learning, diagnosis, or a one-off run; use a controller when the Pod should be replaced or scaled automatically.

For stateful applications that need stable identities or storage behavior, consider a StatefulSet instead of assuming a Deployment is always the right controller.

## Official references

- [Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)
- [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
