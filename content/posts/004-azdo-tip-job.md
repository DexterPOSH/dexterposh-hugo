---
title: "Azure DevOps Tip - Job re-use within a Stage"
date: 2020-04-26T09:53:50+05:30
tags: ["azdo", "tip", "yaml", "jobs"]
---

## Background

Azure DevOps introduced multi-stage yaml pipelines a while ago. It allows us
to define our entire Build/Release landscape inside these yaml definitions.

To re-iterate of some terms used in this post:

- A pipeline comprises one or more stages
- Stage is collection of jobs
- Job runs on an agent/ agentless
- Job contains steps (task/script)
- Steps are the atomic unit to perform a task

## Challenge

Recently, working our multi-stage yaml pipelines, we hit an interesting
behavior with our job template re-use.

We had quite few common steps we require to take in each stage (multiple times)
to hit an external API. So, to re-use we extracted them out as a job
[template](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops).

### jobtemplate.yml

For this post, creating a sample job and taking an
[agentless delay task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/utility/delay?view=azure-devops) as an example but this job template can include
series of steps.

Below is my extracted out job template definition (indues a delay) which I want to re-use within stages in my main **azure-pipelines.yml** pipeline definition later.

```yaml
parameters:
  name: 'windows'
jobs:
  - job: commonJob
    displayName: Common Job
    pool: server
    steps:
    - task: Delay@1
      displayName: 'Delay by 1 minutes for ${{ parameters.name }}'
      inputs:
        delayForMinutes: 1
```

### azure-pipelines.yml

Now, I want to call my Job template from above inside two stages Build & Deploy
in the multi-stage yaml pipeline as below.

```yaml
stages:
- stage: Build
  jobs:
  - template: ./jobtemplate.yml
  - job: Build1

- stage: Deploy
  jobs:
    - template: ./jobtemplate.yml
    - job: Deploy1
```

Kick off a build for this and you would see it in the AzDO portal, that it works
out well.

![Same Job template used with diffn stages](/static/004/jobWithinDiffStages.png)

But, if I try to use the same Job template multiple times within the same stage
that is when things get interesting. Let's modify our **azure-pipelines.yml** definition now.

```yaml
stages:
- stage: Build
  jobs:
  - template: ./jobtemplate.yml
    parameters:
        name: Deepak
  - template: ./jobtemplate.yml
    parameters:
        name: Dhami
  - job: Build1

- stage: Deploy
  jobs:
    - template: ./jobtemplate.yml
    - job: Deploy1
```

Try running this new modified pipeline and you would be greeted with the below
in the portal.

![Job name not unique](/static/004/nameCollision.png)

This is quite obvious but the job names in a stage should be unique.

## Solution

To get to a solution we looked at generating a unique name for our jobs but
it turns out there is a very simple solution to this problem.

One can remove the job identifier in the job template completely while
extracting out jobs to re-use.

What this means is we had to modify our Job templates like below:

```yaml
parameters:
  name: 'windows'
jobs:
  - job: # Identifier here is removed
    displayName: Common Job
    pool: server
    steps:
    - task: Delay@1
      displayName: 'Delay by 1 minutes for ${{ parameters.name }}'
      inputs:
        delayForMinutes: 1
```

Now, it works.

![Job re-use within same stage](/static/004/jobReuse.png)

## Conclusion

As a general practice, when I start extracting out steps as independent job
templates I tend to not populate the Job identifier if it's scope is to be
re-used multiple times in the same stage.

But if there is a job which you want to ensure that it is allowed to run only
once, you could stick with populating the field.

So, as every engineer building solution says:

> It depends on the use-case
