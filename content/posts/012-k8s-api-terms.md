---
title: "Kubernetes API Terms"
date: 2021-10-15T12:23:30+05:30
categories: ["Kubernetes"]
tags: ["K8s", "DevOps", "KubernetesAPI"]
highlightjslanguages: [ "powershell", "bash"]
---

## Overview

My notes on the Kubernetes API terminology.

## API-Server

The K8s core component api-server is at the heart of the interactions made by Client and other K8s components.
It is the only component that can interact with the the etcd (key-value) store.

Reponsibilities:
- Serve the RESTful HTTP API
    - Reading state - fetch single object, list of objects and stream changes (via events)
    - Modify state - Create, Update & Delete objects
- Proxy cluster components e.g. Dashboard, kubectl exec sessions etc.

API server HTTP interface handles HTTP requests to query and manipulate K8s resources using the standard HTTP verbs:
- HTTP GET - retrieve data with a specific resource or collection or list of resources
- HTTP POST - create a resource
- HTTP PUT - Updating an existing resource
- HTTP PATCH - Partial updates to resources
- HTTP DELETE - Destroy a resource (non-recoverable manner)

Above means that when we use a client like Kubectl to interact with the API-Server it is actually issuing
HTTP requests against a certain endpoint.

### API terminology

Below are some of the most common terms thrown around when referring to the REST HTTP API.

#### Kind

To simply put this is the data type of the underlying object.

Broadly there are 3 categories of types:
- Objects - this category represents individual object types like pod, deployment etc.
- List - this data type is basically a collection of objects defined above.
- Special kind of types - this collections refers to types which define action on objects or non-persistent entities.

Note - In K8s the `kind` directly corresponds with a Golang type.

#### API Group

When logically related kind(s) are grouped together it is referred to as API group.
For example - Kind `Job` or `ScheduledJob` are placed under the Batch API group.

#### Version

Each API group can exist in multiple version e.g. an API group first appears as v1alpha1 and is then promoted to v1beta1 and finally graduates to v1.

K8s supports multiple API versions for making it easy to extend, these are served under different API paths e.g. /api/v1 or /apis/extensions/v1beta1.

Different API verions imply diff level of stability and support:

- Alpha level (e.g. v1alpha1) - disabled by default, support for a feature may be dropped at any time, use only in short lived testing clusters.
- Beta level (e.g. v2beta3) enabled by default, code well tested. However object semantics might get a breaking change in next release.
- Stable (e.g. v1) will appear in software for many subsequent versions.

#### Resource

This is usually a plural word and uniquely identifies the HTTP endpoint exposing CRUD semantics for an object type e.g. /pods, /pods/busybox. Common path patterns are:
- The root e.g. /pods which lists all the instances of that type
- Individual named resources e.g /pods/busybox

> Resources correspond to HTTP endpoint paths

> Kinds are the type of objects returned and recieved by these endpoints, which are then persisted into the datastore.

### Points to note

- Resources are always part of an API group and version, also referred to as GroupVersionResource (GVR)
- GVR basically uniquely defines an HTTP endpoint path
- Each kind also is part of an API group and versio, referred to as GroupVersionKind (GVK)
- Relation between GVK & GVR is - GVKs are served under the HTTP Path identified by GVR.
- Above process of mapping GVK to GVR is also termed as REST Mapping in the K8s SDK.
