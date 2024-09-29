---
layout: post
title: Deploying Snowplow
---

In this article we are going to explain our deployment of of Snowplow open source, on GCP plaform. Our deployment collects the events and passes them to PubSub on GCP and then stores them in BigQuery. Our deployment is made possible
primarily because of a great (but a little older document) [Highly available Snowplow pipeline on kubernetes and GCP](https://www.datascienceengineer.com/blog/post-ha-snowplow-on-k8s). What differentiates our deployment compared to that
document are a few things:

1. We use [terraform](https://www.terraform.io/) to provision our cloud infrastructure. This makes the deployment
   cleaner, more manageable, and easier to inspect and replicate

1. We use Google Cloud's [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation) to authenticate the workloads, this greatly simplifies the configuration files as we will see below.

1. We also simplify our deployment by using only one service account for different service types. We have combined all the required the permissions and gave it to one service account for simplicity. You can of course define multiple service accounts. For example our [reference document](https://www.datascienceengineer.com/blog/post-ha-snowplow-on-k8s#service-accounts) uses three separate service accounts for better security.

Before we continue, we should also mention that very recently, in January 8, 2024, Snowplow has changed their license agreement from Apache 2, to their own agreement called [Introducing the Snowplow Limited Use License Agreement](https://snowplow.io/blog/introducing-snowplow-limited-use-license) which limits how you can use Snowplow's open source. In particular, it states:

> You cannot run this software in production in a highly available manner. For current users of Snowplow Open Source Software who want high availability capabilities in production, you must contact Snowplow for a commercial license. However, SLULA does allow you to run highly available development and QA pipelines that are not for production use

Therefore, in order to use Snowplow in production, we need to use images that are released before January 8, 2024. That is why we are not using the latest versions of the docker images, as you will see later in this post.

We divide this document to three parts:

1. [Architecture Overview](#architecture-overview)
2. [Infrastructure Deployment](#infrastructure)
3. [Service Deployment](#services)

In the first part we briefly go over the architecture. Explain types of services that we need to deploy for Snowplow to function properly. In the second part, we explain how to deploy the needed infrastructure, such as PubSub topics, subscriptions, the PostGRE databases that is used by Iglu server, etc. And finally in the last part, we explain how to deplow the servcies in kubernetes.

# Architecture Overview

Perhaps it's the easier to start with the reference architecture for Snowplow deployment on GCP. Below is the diagram originally from Snowplow:

![Snowplow Pipline Architecture](/public/images/Snowplow_Pipeline_architecture.png)
