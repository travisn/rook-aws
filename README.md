# rook-aws

----

**Contents:**
* [Prelude](README.md#prelude)
* [Rational](README.md#rational)
    + [Setup](README.md#setup)
    + [Try it out](README.md#try-it)
* [Instance store vs. EBS](README.md#instance-store-vs-ebs)
* [s3 vs Rook Object Store](README.md#s3-vs-rook-object-store)
* [Conclusions](README.md#conclusions)

-----

## Prelude

[Rook][1] is an evolutionary step in storage management, empowering users to go beyond storage administration routines to orchestration of distributed storage. It is a young open source project, with a lot of potential, that runs on top of [Kubernetes][2] (or standalone) exposing file, block and object storage as k8s primitives.

We'll evaluate several scenarios of running kubernets cluster with rook distributed storage in AWS, and how Rook provides significant advantages of managing storage in teh public cloud. Criteria for comparison is:

- Resiliency
- Compatibility
- Cost
- Performance

## Rational

There're still significant challenges running Kubernetes workloads that are beyond _vanilla_ in a public cloud. This is mostly due to the continuing rapid evolution of _Cloud Native_ standards, and the pace of its adaptations. As it stands now, infrastructure orchestration tools are in conflict with cloud providers, particularly in the area of management of network and storage resources.

This project explores different approaches in providing storage in AWS utilizing Rook and points out things to consider. Does it make sense to use k8s distributed Rook storage or just EBS for block storage? Is there a benefit to have in-cluster object store as an alternative to s3?

Compatibility perhaps is the most compelling reason to start considering using Rook now. Rook acts as an abstraction layer for your storage, and instead of managing specific primitives for different provides, manifests describe Rook primitives resulting in portable infrastructure code. Infrastructure folks would configure Rook cluster to use storage for any given environment, and applications would need no change. Often such compatibility is useful for testing, multi cloud setups or harassing your cloud provider under a threat of migrating out in an hour ;).

### Setup

[Kops][4] project was used to generate initial kubernetes cluster setup as a _[Terraform][5] project_. AWS instances with ephemeral storage (instance store) on k8s nodes, rook cluster setup, prometheus monitoring, grafana to represent metrics and tooling for debugging and load generation.

The terraform project provisions AWS resources needed for a Kubernetes cluster, including VPC, network topology, policies, security groups and ASGs for nodes and master. [Nodeup][7] from Kops is used to bootstrap Kubernetes based on the configuration generated by kops/terraform.

> > &nbsp; :warning: &nbsp; AWS resources described by terraform are tracked by it and their lifecycle is fully managed by terraform. Kubernetes created resources (ELBs for ingress, EBS etc.) are managed by Kubernetes. When destroying your cluster via `terraform destroy` ensure you either delete Kube provisioned resources by `kubectl` or manually to avoid unexpected charges by the cloud provider.

Kubernetes nodes are run on `c3.large`, by default in terraform, with ephemeral storage (2 * 16 GB SSD) and `Moderate` networking performance, which will affect performance of distributed storage. It isn't a goal to compare different instance types, but this project can easily be used for it. I will focus on Rook on Instance store vs. EBS and Rook Object Store vs. AWS's s3. Kubernetes is installed on Ubuntu 16.04 LTS with three nodes in the k8s cluster plus the master.

Rook cluster is configured to deliver block storage using disks attached to instances. The cluster configuration looks like this:

```yaml
apiVersion: rook.io/v1alpha1
kind: Cluster
metadata:
  name: rook-eval
  namespace: rook
spec:
  versionTag: v0.5.1
  dataDirHostPath: "/var/lib/docker/rook-eval"
  storage:
    useAllNodes: true
    useAllDevices: false
    deviceFilter: xvdc
    metadataDevice:
    location:
    storeConfig:
      storeType: bluestore
      databaseSizeMB: 1024
      journalSizeMB: 1024
```

Tests are run within a Pod provisioned with Rook PVS mapped as a volume.

Persistent Volume Claim configuration:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rookeval-claim
  labels:
    app: eval
spec:
  storageClassName: rook-block
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 12Gi
```

Deployment for the test pod:

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: rookeval
  labels:
    app: eval
spec:
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: eval
    spec:
      containers:
      - image: ubuntu
        name: rookeval
        command: [ "/bin/bash", "-c", "--" ]
        args: [ "while true; do sleep 600; done;" ]
        volumeMounts:
        - name: eval-block-storage
          mountPath: /eval
      volumes:
      - name: eval-block-storage
        persistentVolumeClaim:
          claimName: rookeval-claim
```

There're very, very many factors, that might affect the performance of a store in a public cloud, system hardware, networking, hypervisor, instance type, local storage, kernel, filesystem to name a few. AWS offers many options, the blog however focuses on some general patterns, please share your experience, and if similar considerations are part of your evaluation of a public provider feel free to use and improve this setup.

:exclamation: As of rook v0.5.x ceph replication can't be set via rook configuration. The default is 1 replica, witch provides no resiliency. The default was changed to have 3 replicas, which will impact the performance particularly writes. I deployed [rook-tools][8] and ran `ceph osd pool set replicapool size 3` from it.

### Try it

If you are designing your own Kubernetes architecture for AWS, this project might be helpful to try different options and run comparison tests.

#### Requirements

- AWS account with AWS CLI setup and configured.
- Install [terraform][5]
- Install [kubectl][6]
- In the `./terraform` directory create `terraform.tfvars` with your customizations. At a minimum two variables have to be set - *key_name* and *zone*.

#### Provisioning

1. Deploy terraform project: `cd terraform && terrafrom init && terrafrom apply`
1.

## Instance store vs. EBS

Basic `dd` small writes speed tests can be found bellow. Ec2 instance store provisioned via rook within a test container shows more the double the speed for these type of writes.

```bash
root@rookeval-456649855-nlldd:/eval# df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay         9.7G  4.4G  5.3G  46% /
tmpfs           1.9G     0  1.9G   0% /dev
tmpfs           1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/rbd0       9.8G  183M  9.1G   2% /eval
/dev/xvda1      9.7G  4.4G  5.3G  46% /etc/hosts
shm              64M     0   64M   0% /dev/shm
tmpfs           1.9G   12K  1.9G   1% /run/secrets/kubernetes.io/serviceaccount

root@rookeval-456649855-nlldd:/eval# dd if=/dev/zero of=/tmp/output bs=8k count=20k
20480+0 records in
20480+0 records out
167772160 bytes (168 MB, 160 MiB) copied, 0.582112 s, 288 MB/s

root@rookeval-456649855-nlldd:/eval# dd if=/dev/zero of=/eval/output bs=8k count=20k
20480+0 records in
20480+0 records out
167772160 bytes (168 MB, 160 MiB) copied, 0.264301 s, 635 MB/s
```

### Resiliency

EBS

### Compatibility

Major Public Cloud provides are likely to resist adaptation of common Cloud standards. Few of them currently have a competitive advantage, and wide compatibility can potentially hurt their bottom line, _capitalizm_ baby... During this period abstracting different Cloud providers primitives gives flexibility and compatibility for your workloads. Rook can act as such an abstraction layer for storage, it can abstract block, file and object storage so that the same manifests can be used to deploy applications to any cloud provider or on premises.


### Cost

### Performance

Since Instance store would have less latency, it should be superior to EBS for frequent small size read/write operations. And indeed, comparative tests demonstrate that ---

-----

## s3 vs. Rook Object store

### Resiliency

### Compatibility

### Cost

Using in-cluster object store provided by Rook completely removes the cost of object store, since already provisioned cloud resources on which your k8s is running are used.

### Performance

For in-cluster use of the object store gains significant improvements.

-----

## Conclusions

Rook is...

Notes:

Public cloud storage resource offerings aren't always optimal to run Cloud Native applications. This talk explores several storage options comparing costs, performance, resilience, features and interfaces of file, block and object storage for Cloud Native applications in AWS. EBS vs Instance store for Kubernetes nodes are compared for different scenarios. Pros and cons of leveraging object store using resources already provisioned as oppose s3.


-----

[1]: https://rook.io
[2]: https://kubernetes.io
[3]: http://ceph.com
[4]: https://github.com/kubernetes/kops
[5]: https://www.terraform.io
[6]: https://kubernetes.io/docs/tasks/tools/install-kubectl
[7]: https://github.com/kubernetes/kops/blob/master/pkg/model/resources/nodeup.go
[8]: https://github.com/rook/rook/blob/master/cluster/examples/kubernetes/rook-tools.yaml