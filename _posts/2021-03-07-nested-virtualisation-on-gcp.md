---
title: "How to provisions a Google Cloud compute instance with support for nested virtualization"
date: 2021-03-07 09:00:00 +0100
categories: [devops]
tags: [gcp, bash] ## always lowercase !!
image:
  path: /4.webp
  alt: "Provisions a GCP compute instance with support for nested virtualization"
---

## Introduction
At times it is required to configure a Compute instances to support [nested virtualization](https://cloud.google.com/compute/docs/instances/nested-virtualization/overview). This tutorial aims to provide the instructions on how to achieve that on Google Cloud Platform.

## Problem
Nested virtualization allows you to run virtual machines (VM) inside other VMs. This can be useful when you want to run a VM, and your machine is already a VM, for example in a Cloud environment. Another use case can be cost reduction, since you can share an environment with multiple nested VMs. In my specific case I wanted to create a Kubernetes cluster from scratch.

## Plan
In this tutorial I will not cover any ground on what problem I am trying to solve (I will do that in another post), but I will rather focus on HOW I approached the problem of programmatically create Terraform resources with Go. This means, regardless of your use-case, you should have good pointers on how to programmatically write Terraform files after reading this post. See this [Stack Overflow discussion](https://stackoverflow.com/a/59052599)

## Prerequisites
The following are the requirements for the successful execution of the script

1. [Create a Google Cloud account](https://cloud.google.com/free)
2. [Install Google Cloud SDK](https://cloud.google.com/sdk/docs/install)
3. Execute from a Unix-based Operating System (MacOS or Linux)

### Step 1: Authenticate to gcloud, create project and configure defaults
Export environment variables with default values
```bash
export GCP_PROJECT=nested-virtualization
# see https://cloud.google.com/compute/docs/regions-zones
export GCP_REGION=europe-west1
export GCP_ZONE=europe-west1-b
export NAME=nv-test
export MACHINE_TYPE=n1-standard-4
``` 

Authenticate to Google Cloud
```bash
gcloud auth login --no-launch-browser
```

Create project
```bash
gcloud projects create $GCP_PROJECT
```

> After creating a new project you will have to enable billing for the project, unless you are on a Free account. See [Google Cloud doc for reference](https://cloud.google.com/billing/docs/how-to/modify-project)
{: .prompt-warning }

Configure default values
```bash
gcloud config set project $GCP_PROJECT
gcloud config set compute/region $GCP_REGION
gcloud config set compute/zone $GCP_ZONE
```

### Step 2: Create VM instance
Enable compute API
```bash
gcloud services enable compute.googleapis.com
```

Create a disk from the `debian-9` image family with 200GB pd storage you can ignore the warning `".. You might need to resize the root repartition manually .."` as the operating system supports automatic resizing.
```bash
gcloud compute disks create "$NAME-disk" \
	--image-project debian-cloud \
	--image-family debian-12 \
	--zone $GCP_ZONE \
	--size 200 \
	--type pd-standard
```

Create custom image with special license key for nested virtualization
```bash
gcloud compute images create "$NAME-nested-vm-image" \
	--source-disk "$NAME-disk" \
	--source-disk-zone $GCP_ZONE \
	--licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx"
```

Create a VM instance using the new custom image with the license.
```bash
gcloud compute instances create "$NAME-nested-vm" \
	--zone $GCP_ZONE \
	--min-cpu-platform "Intel Haswell" \
	--image "$NAME-nested-vm-image" \
	--machine-type=$MACHINE_TYPE
```

### Step 3: Instance operations
Connect to the VM instance using `gcloud`
```bash
gcloud compute ssh "$NAME-nested-vm" \
	--zone $GCP_ZONE
```

Start or re-start the instance
```bash
gcloud compute instances start "$NAME-nested-vm" \
	--zone $GCP_ZONE -q
```

Stop the instance
```bash
gcloud compute instances stop "$NAME-nested-vm" \
	--zone $GCP_ZONE -q
```

### Step 4: Clean up
Destroy the instance
```bash
gcloud compute instances delete "$NAME-nested-vm" \
	--zone $GCP_ZONE -q
```

Delete the image
```bash
gcloud compute images delete "$NAME-nested-vm-image" -q
```

Delete the disk
```bash
gcloud compute disks delete "$NAME-disk" \
	--zone $GCP_ZONE -q
```

Disable compute API service 
```bash
gcloud services disable compute.googleapis.com
```

## Validation
To check that nested virtualization is enabled run `grep -cw vmx /proc/cpuinfo`. A nonzero response confirms that nested virtualization is enabled.

```bash
simone@nv-test-nested-vm:~$ grep -cw vmx /proc/cpuinfo
8
```

## Conclusions
This short tutorial demonstrate the simple process to enable nested virtualization on Google Cloud. Your next step could be deploying a Kubernetes cluster from scratch on this VM!

## References
* [Google Cloud Docs: nested virtualization](https://cloud.google.com/compute/docs/instances/nested-virtualization/overview)
* [Stack Overflow: Practical use of Nested virtualization](https://stackoverflow.com/a/59052599)
* [GH repo](https://github.com/SRodi/calico-certified-operator-infra-gcloud)