- [Argo Rollouts](#argo-rollouts)
  - [Features](#features)
  - [How does it work?](#how-does-it-work)
  - [Installation](#installation)
  - [Concepts](#concepts)
    - [Deployment Strategies](#deployment-strategies)
  - [Architecture](#architecture)
    - [Rollout resource](#rollout-resource)
    - [Replica sets for old and new version](#replica-sets-for-old-and-new-version)
    - [Ingress/Service](#ingressservice)
    - [AnalysisTemplate and AnalysisRun](#analysistemplate-and-analysisrun)
    - [Metric providers](#metric-providers)

# [Argo Rollouts](https://argoproj.github.io/argo-rollouts/)

Argo Rollouts is a Kubernetes controller and set of CRDs which provide advanced deployment capabilities such as blue-green, canary, canary analysis, experimentation, and progressive delivery features to Kubernetes.

Argo Rollouts (optionally) integrates with ingress controllers and service meshes, leveraging their traffic shaping abilities to gradually shift traffic to the new version during an update. Additionally, Rollouts can query and interpret metrics from various providers to verify key KPIs and drive automated promotion or rollback during an update.

## Features

* Blue-Green update strategy
* Canary update strategy
* Fine-grained, weighted traffic shifting
* Automated rollbacks and promotions
* Manual judgement
* Customizable metric queries and analysis of business KPIs
* Ingress controller integration: NGINX, ALB
* Service Mesh integration: Istio, Linkerd, SMI
* Simultaneous usage of multiple providers: SMI + NGINX, Istio + ALB, etc.
* Metric provider integration: Prometheus, Wavefront, Kayenta, Web, Kubernetes Jobs, Datadog, New Relic, Graphite

## How does it work?

Similar to the deployment object, the Argo Rollouts controller will manage the creation, scaling, and deletion of ReplicaSets. These ReplicaSets are defined by the spec.template field inside the Rollout resource, which uses the same pod template as the deployment object.

When the spec.template is changed, that signals to the Argo Rollouts controller that a new ReplicaSet will be introduced. The controller will use the strategy set within the spec.strategy field in order to determine how the rollout will progress from the old ReplicaSet to the new ReplicaSet. Once that new ReplicaSet is scaled up (and optionally passes an Analysis), the controller will mark it as "stable".

If another change occurs in the spec.template during a transition from a stable ReplicaSet to a new ReplicaSet (i.e. you change the application version in the middle of a rollout), then the previously new ReplicaSet will be scaled down, and the controller will try to progress the ReplicasSet that reflects the updated spec.template field. There is more information on the behaviors of each strategy in the spec section.

## Installation

Refer https://argoproj.github.io/argo-rollouts/installation/

## Concepts

### Deployment Strategies

* Rolling Update
* Recreate
* Blue-Green
* Canary

## Architecture
![Architecture](https://argoproj.github.io/argo-rollouts/architecture-assets/argo-rollout-architecture.png)

### Rollout resource

The Rollout resource is a custom Kubernetes resource introduced and managed by Argo Rollouts. It is mostly compatible with the native Kubernetes Deployment resource but with extra fields that control the stages, thresholds and methods of advanced deployment methods such as canaries and blue/green deployments.

Note that the Argo Rollouts controller will only respond to those changes that happen in Rollout sources. It will do nothing for normal deployment resources. This means that you need to migrate your Deployments to Rollouts if you want to manage them with Argo Rollouts.

You can see all possible options of a Rollout in the full specification page.

### Replica sets for old and new version

These are instances of the standard Kubernetes ReplicaSet resources. Argo Rollouts puts some extra metadata on them in order to keep track of the different versions that are part of an application.

Note also that the replica sets that take part in a Rollout are fully managed by the controller in an automatic way. You should not tamper with them with external tools.

### Ingress/Service

This is the mechanism that traffic from live users enters your cluster and is redirected to the appropriate version. Argo Rollouts use the standard Kubernetes service resource, but with some extra metadata needed for management.

Argo Rollouts is very flexible on networking options. First of all you can have different services during a Rollout, that go only to the new version, only to the old version or both. Specifically for Canary deployments, Argo Rollouts supports several service mesh and ingress solutions for splitting traffic with specific percentages instead of simple balancing based on pod counts and it is possible to use multiple routing providers simultaneously.

### AnalysisTemplate and AnalysisRun

Analysis is the capability to connect a Rollout to your metrics provider and define specific thresholds for certain metrics that will decide if an update is successful or not. For each analysis you can define one or more metric queries along with their expected results. A Rollout will progress on its own if metric queries are good, rollback automatically if metrics show failure and pause the rollout if metrics cannot provide a success/failure answer.

For performing an analysis, Argo Rollouts includes two custom Kubernetes resources:  AnalysisTemplate and AnalysisRun.

AnalysisTemplate contains instructions on what metrics to query. The actual result that is attached to a Rollout is the AnalysisRun custom resource. You can define an AnalysisTemplate on a specific Rollout or globally on the cluster to be shared by multiple rollouts as a  ClusterAnalysisTemplate. The AnalysisRun resource is scoped on a specific rollout.

Note that using an analysis and metrics in a Rollout is completely optional. You can manually pause and promote a rollout or use other external methods (e.g. smoke tests) via the API or the CLI. You don't need a metric solution just to use Argo Rollouts. You can also mix both automated (i.e. analysis based) and manual steps in a Rollout.

Apart from metrics, you can also decide the success of a rollout by running a Kubernetes job or running a webhook.

### Metric providers

Argo Rollouts includes native integration for several popular metrics providers that you can use in the Analysis resources to automatically promote or rollback a rollout. See the documentation of each provider for specific setup options.