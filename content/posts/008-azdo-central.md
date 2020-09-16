---
title: "Azure DevOps Pipeline as a Service - Part 1"
date: 2020-07-04T15:18:35+05:30
draft: true
image: "/static/008/AzDOPaaS.png"

---
![Pipeline as a Service Cover](/static/008/AzDOPaaS.png)

## Problem Statement 🤔

If you are working in a team supporting Application/ Infrastructure and authoring or supporting build/release pipelines in Azure DevOps, you'd have faced some of the below pain points.

* Rolling out updates to different build/release pipelines e.g. task extension version update etc.
* Governance around pipelines e.g. allowing changes via a Pull request and review.
* Time to deliver pipeline to a new application.

## Research 🤖

Azure devops has this hidden gem of a feature which a teamcan leverage to maintain a centralized repository of build/release templates under and Azure DevOps tenant.

It is called [**Use other repositories**](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#use-other-repositories) in the official docs for templates.

> Thanks to [Mark Kraus](https://twitter.com/markekraus) for pointing out the more marketed term for this feature **Pipeline as a Service**.

What this allows to do is:

* Maintain a single repository housing all the multi-stage yaml based templates under an Azure DevOps tenant.
* Developers can create a repository and choose which remote template to use.

Limitations:

* Centralized build templates repository can only contain yaml files, no other file type is supported. This means you can't place your helper scripts side by side in the repository.
* No marketplace/feed or gallery to advice the supported pipeline templates, at most this can be documented on a wiki.


## Solution - Pipeline as a Service ⚛

For the sake of showing this feature, I am going to keep it a very simple scenario.

* Create a centralized repository
* Author a minimalistic pipeline to perform ARM template deployment.
* Create an end-user repository and consume above pipeline template.

### Centralize Repository

Let's start by creating 2 projects in Azure DevOps to show that this works very well within the Azure DevOps tenant boundary.

Two projects & repositories created:

* **Build** project
  * **templates** repository
* **AppDev** project
  * **ARMTemplates** repository

### Author ARM template pipeline

Let's keep this pipeline simple and take something already out there.
Idea is to add this centralized

### Consume the Pipeline Service

## Conclusion

## Resources
