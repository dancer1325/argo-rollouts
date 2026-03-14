# Argo Rollouts - Progressive Delivery for Kubernetes

[![Artifact HUB](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/argo-rollouts)](https://artifacthub.io/packages/helm/argo/argo-rollouts)

## What is Argo Rollouts?

* Argo Rollouts
  * == Kubernetes controller + set of CRDs 
  * provide
    * advanced deployment capabilities | Kubernetes
      * blue-green
      * canary
      * canary analysis
      * experimentation
      * progressive delivery features
  * 👀can (== OPTIONAL) integrate -- with --👀
    * traffic routing tools
      * _Examples:_ ingress controllers & service meshes
      * Reason:🧠gather information -- to -- gradually shift traffic🧠
    * metric provider tools

* [video demo](https://youtu.be/hIL0E2gLkf8)
  * TODO:

## Why Argo Rollouts?

TODO:
Kubernetes Deployments provides the `RollingUpdate` strategy which provide a basic set of safety guarantees (readiness probes) during an update
* However the rolling update strategy faces many limitations:

* Few controls over the speed of the rollout
* Inability to control traffic flow to the new version
* Readiness probes are unsuitable for deeper, stress, or one-time checks
* No ability to query external metrics to verify an update
* Can halt the progression, but unable to automatically abort and rollback the update

For these reasons, in large scale high-volume production environments, a rolling update is often considered too risky of an update procedure since it provides no control over the blast radius, may rollout too aggressively, and provides no automated rollback upon failures.

## Features

* Blue-Green update strategy
* Canary update strategy
* Fine-grained, weighted traffic shifting
* Automated rollbacks and promotions
* Manual judgement
* Customizable metric queries and analysis of business KPIs
* Ingress controller integration: NGINX, ALB, Apache APISIX
* Service Mesh integration: Istio, Linkerd, SMI
* Metric provider integration: Prometheus, Wavefront, Kayenta, Web, Kubernetes Jobs, Datadog, New Relic, InfluxDB

## Supported Traffic Shaping Integrations
| Traffic Shaping Integration       | SetWeight                    | SetWeightExperiments        | SetMirror                  | SetHeader                  | Implemented As Plugin       |
|-----------------------------------|------------------------------|-----------------------------|----------------------------|----------------------------|-----------------------------|
| ALB Ingress Controller            | :white_check_mark: (stable)  | :white_check_mark: (stable) | :x:                        | :white_check_mark: (alpha) |                             |
| Ambassador                        | :white_check_mark: (stable)  | :x:                         | :x:                        | :x:                        |                             |
| Apache APISIX Ingress Controller  | :white_check_mark: (alpha)   | :x:                         | :x:                        | :white_check_mark: (alpha) |                             |
| Istio                             | :white_check_mark: (stable)  | :white_check_mark: (stable) | :white_check_mark: (alpha) | :white_check_mark: (alpha) |                             |
| Nginx Ingress Controller          | :white_check_mark: (stable)  | :x:                         | :x:                        | :x:                        |                             |
| SMI                               | :white_check_mark: (stable)  | :white_check_mark: (stable) | :x:                        | :x:                        |                             |
| Traefik                           | :white_check_mark: (stable)  | :x:                         | :x:                        | :x:                        |                             |
| Contour                           | :white_check_mark: (beta)    | :x:                         | :x:                        | :x:                        | :heavy_check_mark:          |
| Gateway API                       | :white_check_mark: (alpha)   | :x:                         | :x:                        | :x:                        | :heavy_check_mark:          |

:white_check_mark: = Supported

:x: = Not Supported

:heavy_check_mark: = Yes

## Documentation

* [here](docs)

## Community

You can reach the Argo Rollouts community and developers via the following channels:

* Q & A: [Github Discussions](https://github.com/argoproj/argo-rollouts/discussions)
* Chat: [The #argo-rollouts Slack channel](https://argoproj.github.io/community/join-slack)
* Contributors Office Hours: [Every Thursday](https://calendar.google.com/calendar/u/0/embed?src=argoproj@gmail.com) | [Agenda](https://docs.google.com/document/d/1xkoFkVviB70YBzSEa4bDnu-rUZ1sIFtwKKG1Uw8XsY8)
* User Community meeting: [First Wednesday of each month](https://calendar.google.com/calendar/u/0/embed?src=argoproj@gmail.com) | [Agenda](https://docs.google.com/document/d/1ttgw98MO45Dq7ZUHpIiOIEfbyeitKHNfMjbY5dLLMKQ)

## How does it work?
Similar to the [deployment object](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/), the Argo Rollouts controller will manage the creation, scaling, and deletion of [ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/). These ReplicaSets are defined by the `spec.template` field inside the Rollout resource, which uses the same pod template as the deployment object.

When the `spec.template` is changed, that signals to the Argo Rollouts controller that a new ReplicaSet will be introduced. The controller will use the strategy set within the `spec.strategy` field in order to determine how the rollout will progress from the old ReplicaSet to the new ReplicaSet. Once that new ReplicaSet is scaled up (and optionally passes an [Analysis](features/analysis/)), the controller will mark it as "stable".

If another change occurs in the `spec.template` during a transition from a stable ReplicaSet to a new ReplicaSet (i.e. you change the application version in the middle of a rollout), then the previously new ReplicaSet will be scaled down, and the controller will try to progress the ReplicasSet that reflects the updated `spec.template` field. There is more information on the behaviors of each strategy in the [spec](features/specification/) section.

## Use cases of Argo Rollouts

- A user wants to run last-minute functional tests on the new version before it starts to serve production traffic. With the BlueGreen strategy, Argo Rollouts allows users to specify a preview service and an active service. The Rollout will configure the preview service to send traffic to the new version while the active service continues to receive production traffic. Once a user is satisfied, they can promote the preview service to be the new active service. ([example](https://github.com/argoproj/argo-rollouts/blob/master/examples/rollout-bluegreen.yaml))

- Before a new version starts receiving live traffic, a generic set of steps need to be executed beforehand. With the BlueGreen Strategy, the user can bring up the new version without it receiving traffic from the active service. Once those steps finish executing, the rollout can cut over traffic to the new version.

- A user wants to give a small percentage of the production traffic to a new version of their application for a couple of hours. Afterward, they want to scale down the new version and look at some metrics to determine if the new version is performant compared to the old version. Then they will decide if they want to roll out the new version for all of the production traffic or stick with the current version. With the canary strategy, the rollout can scale up a ReplicaSet with the new version to receive a specified percentage of traffic, wait for a specified amount of time, set the percentage back to 0, and then wait to rollout out to service all of the traffic once the user is satisfied. ([example](https://github.com/argoproj/argo-rollouts/blob/master/examples/rollout-analysis-step.yaml))

- A user wants to slowly give the new version more production traffic. They start by giving it a small percentage of the live traffic and wait a while before giving the new version more traffic. Eventually, the new version will receive all the production traffic. With the canary strategy, the user specifies the percentages they want the new version to receive and the amount of time to wait between percentages. ([example](https://github.com/argoproj/argo-rollouts/blob/master/examples/rollout-canary.yaml))

- A user wants to use the normal Rolling Update strategy from the deployment. If a user uses the canary strategy with no steps, the rollout will use the max surge and max unavailable values to roll to the new version. ([example](https://github.com/argoproj/argo-rollouts/blob/master/examples/rollout-rolling-update.yaml))

## Who uses Argo Rollouts?

[Official Argo Rollouts User List](https://github.com/argoproj/argo-rollouts/blob/master/USERS.md)

## Community Blogs and Presentations

* [Awesome-Argo: A Curated List of Awesome Projects and Resources Related to Argo](https://github.com/terrytangyuan/awesome-argo)
* [Automation of Everything - How To Combine Argo Events, Workflows & Pipelines, CD, and Rollouts](https://youtu.be/XNXJtxkUKeY)
* [Argo Rollouts - Canary Deployments Made Easy In Kubernetes](https://youtu.be/84Ky0aPbHvY)
* [How Intuit Does Canary and Blue Green Deployments](https://www.youtube.com/watch?v=yeVkTTO9nOA)
* [Leveling Up Your CD: Unlocking Progressive Delivery on Kubernetes](https://www.youtube.com/watch?v=Nv0PPwbIEkY)
* [Minimize failed deployments with Argo Rollouts and Smoke tests](https://codefresh.io/continuous-deployment/minimize-failed-deployments-argo-rollouts-smoke-tests/)
* [Recover automatically from failed deployments with Argo Rollouts and Prometheus metrics](https://codefresh.io/continuous-deployment/recover-automatically-from-failed-deployments/)
* [Kubernetes Blue-Green deployments with Argo Rollouts](https://www.youtube.com/watch?v=krDxDz4V4Tg)
* [Kubernetes canary deployments with Argo Rollouts](https://www.youtube.com/watch?v=fviYWA2mcF8)
* [GitOps with Argo CD and an Argo Rollouts canary release](https://www.youtube.com/watch?v=35Qimb_AZ8U)
* [Multi-Stage Delivery with Keptn and Argo Rollouts](https://www.youtube.com/watch?v=w-E8FzTbN3g&t=1s)
* [Gradual Code Releases Using an In-House Kubernetes Canary Controller on top of Argo Rollouts](https://doordash.engineering/2021/04/14/gradual-code-releases-using-an-in-house-kubernetes-canary-controller/)
* [How Scalable is Argo-Rollouts: A Cloud Operator’s Perspective](https://www.youtube.com/watch?v=rCEhxJ2NSTI)
* [Minimize Impact in Kubernetes Using Argo Rollouts](https://medium.com/@arielsimhon/minimize-impact-in-kubernetes-using-argo-rollouts-992fb9519969)
* [Progressive Application Delivery with GitOps on Red Hat OpenShift](https://www.youtube.com/watch?v=DfeL7cdTx4c)
* [Progressive delivery for Kubernetes Config Maps using Argo Rollouts](https://codefresh.io/blog/progressive-delivery-for-kubernetes-config-maps-using-argo-rollouts/)
* [Multi-Service Progressive Delivery with Argo Rollouts](https://codefresh.io/blog/multi-service-progressive-delivery-with-argo-rollouts/)
* [Progressive Delivery for Stateful Services Using Argo Rollouts](https://codefresh.io/blog/progressive-delivery-for-stateful-services-using-argo-rollouts/)
* [Argo Rollouts Demo application](https://github.com/argoproj/rollouts-demo)
