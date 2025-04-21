---
title: "Recovering S3 Data from a Ceph RGW Pool after Placement‑Group Loss: A Case Study"
date: 2025-04-21
tags: ["ceph", "data-recovery", "rados", "case-study"]
slug: "how-to-recover-files-from-broken-s3-bucket-on-ceph-rgw"
---

![](attachments/poster.webp)


In this case study I want to describe how I was able to recover (partially) objects from a Ceph RADOS Gateway (RGW) pool after several placement groups (PGs) became irrecoverably corrupted. Unable to use the standard S3 API, I demonstrate how to leverage the `rados` CLI, craft custom scripts to list and download objects directly from the underlying RADOS pool, and reconcile partial data loss while restoring service availability.

# Introduction!

Ceph's RGW provides an S3-compatible storage interface backed by an underlaying RADOS pool. RGW maintains three types of information: metadata, bucket indexes, and (payload) data.

## Metadata

In RGW, metadata is categorized into three sections:

1. **User Metadata** - this section holds information related to users who interact with the Rados Gateway
2. **Bucket Metadata** - The bucket metadata stores information mapping a bucket's name to its corresponding bucket instance ID. This section allows RGW to track the creation and configuration of different buckets within the system.
3. **Bucket Instance Metadata** - This section holds detailed information about the specific instance of a bucket, such as its unique instance ID and related attributes. A bucket might have multiple instances, particularly when versioning is enabled or when dealing with replication across different regions.

## Bucket index

**Bucket Index** in RGW is a specialized type of metadata used to manage and organize the objects within a bucket. Unlike regular metadata (such as user, bucket, or bucket instance metadata), the bucket index focuses on efficiently indexing the objects stored in a bucket and maintaining critical information for fast object lookup and management.

It is a crucial component of RGW's internal data structure, allowing it to efficiently track, organize, and retrieve object metadata within a bucket. The bucket index uses RADOS objects to store the index and can be sharded for scalability. The index is structured as a key-value map, where the **key** is the object name, and the **value** contains basic metadata about the object. Additional information, like bucket accounting data (e.g., object count, total size), logs, and versioning metadata, is also maintained within the bucket index. This enables RGW to provide efficient object listing, fast lookups, and proper management of large-scale object storage.

## Data

**Data** in RGW refers to the actual content of the objects stored in the system. Each object is stored in one or more RADOS objects, depending on the size and

# Background