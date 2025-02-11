---
id: manager
title: Manager
---

It plays the role of manager in the multi-P2P cluster deployment process.
Used to manage the dynamic configuration that each module depends on,
and provide keepalive and metrics functions.

## Features {#features}

- Stores dynamic configuration for consumption by cdn cluster, scheduler cluster and dfdaemon
- Maintain the relationship between cdn cluster and scheduler cluster
- Provide async task management features for image preheat combined with harbor
- Keepalive with scheduler instance and cdn instance
- Filter the optimal scheduler cluster for dfdaemon
- Provides a visual console, which is helpful for users to manage the P2P cluster

## Relationship {#relationship}

- CDN cluster and Scheduler cluster have a `1:N` relationship
- CDN cluster and CDN instance have a `1:N` relationship
- Scheduler cluster and Scheduler instance have a `1:N` relationship

![manager-relationship](../../resource/architecture/manager-relationship.jpg)
