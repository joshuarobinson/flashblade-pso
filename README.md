# Stateful Kubernetes Applications Made Easy: PSO and FlashBlade

Running stateful services using Kubernetes and Pure Storage Orchestrator to
automate and productionize those services with FlashBlade as shared storage. I
will focus on three applications: a simplistic shared scratch space as well as
file-sharing services [NextCloud](https://nextcloud.com/) and [OwnCloud](https://owncloud.org/).

I assume the reader has only a basic understanding of Kubernetes and is
interested in examples showing how to use FlashBlade with Kubernetes.

For a more general introduction and guide to PSO for both FlashArray and
FlashBlade, see this [explanation of Kubernetes
Storage](https://blog.purestorage.com/the-kubernetes-storage-eco-system-explained/),
[Kubernetes
documentation](https://support.purestorage.com/Solutions/Kubernetes/Kubernetes%2C_Persistent_Volumes%2C_and_Pure_Storage),
and [Containers-as-a-service
architecture](https://www.purestorage.com/content/dam/purestorage/pdf/datasheets/ps-containers-as-a-service-ri.pdf).

## Introduction

The [Pure Storage Orchestrator](https://hub.docker.com/r/purestorage/k8s/)
(PSO) automates the process of creating filesystems on FlashBlade and attaching
them to the running applications in Kubernetes.

But first, why run applications in Kubernetes? There is a generic set of
problems that almost all applications need to solve: recovering from hardware
failures, scaling up or down resources, orchestrating connectivity between
inter-linked services, securing and isolating environments, and provisioning
resources for running applications. These are the types of problems that
Kubernetes solves for the application engineers.

## Why Use PSO and FlashBlade

In Kubernetes, applications run inside a set of
[pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/), which are
ephemeral and not tied to physical compute or storage resources. If the pod
needs to access persistent storage for any reason (and there are many!),
[PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
are needed. But manual administration is complicated: creating volumes, attaching
them to running pods regardless of protocol, and finally deleting volumes. In a
fully automated system, none of these steps need to involve a human!

PSO automates the process of provisioning storage and hides the details of
storage creation and attachment to each pod. The result is [self-service
storage](https://blog.purestorage.com/container-storage-as-a-service/) that
matches cloud-native workflows and applications. 

FlashBlade provides shared filesystems for Read-Write-Many (RWX) volumes. These
are critical for scale-out applications that spawn multiple pods; each
additional pod is able to automatically share access to a common data store.
FlashBlade can also support Read-Write-Once (RWO) volumes.

While I focus here on FlashBlade filesystems as volumes, PSO also serves
FlashArray and block devices. The details of storage administration are
different for block (FlashArray) and file (FlashBlade) volumes, and PSO hides
all of these differences from the end-user. For example, a developer creating
an application that uses a ReadWriteOnce volume can choose between latency
sensitive or bandwidth sensitive performance simply by changing the text
between "pure-file" and "pure-block." All of the annoying differences between
iSCSI, Fibre Channel and NFS mounts are hidden automatically by PSO.

## How to Use PSO with FlashBlade

Compared to traditional storage workflows, PSO automates most of the requests
and interactions between users and administrators for file system creation and
mounting. The result is a self-service storage infrastructure; no direct
interaction is necessary between administrators and users. 

Administrators install and occasionally upgrade the PSO software and users
(developers) interact with standard Kubernetes PersistentVolume mechanisms.

### For the Kubernetes Administrator

PSO is a software layer for Kubernetes that utilizes the public REST API
provided by FlashBlade to automate storage-related tasks and is [installed via
helm](https://github.com/purestorage/helm-charts/tree/master/pure-k8s-plugin#how-to-install).

The one-time PSO configuration requires a FlashBlade management IP, data VIP,
and the corresponding API token. These API tokens prove that PSO has permission
to create and delete volumes or filesystems so safeguard them appropriately.

To get the API token, use the following command from the FlashBlade CLI:
```
pureuser@flashblade-ch1-fm2> pureadmin list --api-token --expose
Name      API Token                               Created                  Expires
pureuser  T-c4925090-c9bf-4033-8537-d24ee5669135  2017-09-12 13:26:31 PDT  -
```

Then, add the following to the PSO values.yaml
([example](example_pso_config.yaml)) file to add a FlashBlade under management:
```
    FlashBlades:
    	- MgmtEndPoint: "10.62.64.20"
    	  NFSEndPoint: "10.62.64.200",
    	  APIToken: "T-ab0ed3e0-5438-4485-9503-863e6a9c1434"
```

The PSO works by fulfulling any PVC claims with a StorageClass with
corresponding name: “pure-file.” Update the default storageClass to use the
“pure-file” class if you want FlashBlade to be the default for PVCs that do not
specify StorageClass.

```
> kubectl patch storageclass pure-file -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

After this change, view the configured StorageClasses and their associated provisioner with the following command:
```
> kubectl get storageclass
NAME                  PROVISIONER        AGE
pure-block            pure-provisioner   24d
pure-file (default)   pure-provisioner   24d
```

Finally, to view the PersistentVolumes in the system which were automatically created by PSO:
```
> kubectl get persistentvolume
...
```

The PersistentVolumes reported here should match the filesystems created on the
FlashBlade, as seen with the following CLI command in the FlashBlade:
```
> purefs list --filter "name = 'k8s-*'"
Name                                          Size  Used     Hard Limit  Created                  Protocols
k8s-pvc-98bb10ca-c037-11e8-9ada-525402501103  1T    0.00     False       2018-09-24 13:22:46 PDT  nfs
k8s-pvc-98c51436-c037-11e8-9ada-525402501103  800G  52.50K   False       2018-09-24 13:22:46 PDT  nfs
k8s-pvc-98d6147c-c037-11e8-9ada-525402501103  20T   450.00K  False       2018-09-24 13:22:48 PDT  nfs
k8s-pvc-9fcfdf49-beb3-11e8-9ada-525402501103  5T    0.00     False       2018-09-22 15:05:33 PDT  nfs
```

Adding a
[StorageQuota](https://kubernetes.io/docs/tasks/administer-cluster/limit-storage-consumption/)
([example](storage-quota.yaml)) allows the administrator
to limit the number of claims or the total amount of storage requested. These
limits cause graceful failures in the case of buggy jobs that consume too much
storage.

### For the Kubernetes User

The following examples create a
[PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)
that will be satisfied by the PSO by using a
[storageClassName](https://kubernetes.io/docs/concepts/storage/storage-classes/)
of “pure-file”.  The storageClass signals to Kubernetes how to satisfy the
claim request.  Switching to use a FlashArray block device instead can be done
by using “pure-block” as StorageClass.

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: file-claim
spec:
  storageClassName: pure-file  # This line is all that is required to use PSO
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 15Ti
```

A PersistentVolumeClaim is a request for a PersistentVolume of certain type and
size. If there is a plugin for the specified storageClass, then it will create
a matching PersistentVolume. To attach this storage to a container, refer back
to the claim as follows:

```
volumes:
      - name: myvol
        persistentVolumeClaim:
          claimName: file-claim
      containers:
	...container config...
        volumeMounts:
        - name: file-claim
          mountPath: /mountpoint 
```

What did you NOT need to do?
 * Create a filesystem via a UI of any kind (GUI or CLI)
 * Log in to the node and issue the commands to mount the filesystem
 * Think about what happens if the physical node fails and you need to move the app
 * Track who uses which filesystem so you know when to cleanup

And what you **do** get is that the volume automatically follows your container if
Kubernetes restarts or moves the container to a different physical host.

## Example 1: Shared Scratch Space

The first example to illustrate PSO is a simple Deployment and
PersistentVolumeClaim to create a shared scratch workspace. In other words, the
application is a standard Linux shell and command line tools.

A shared scratch workspace is useful when a small team needs to work together,
for example for forensics investigation of log files. The goal is to
automatically create a shared scratch space to download necessary data to work
with and produce derivative datasets. PSO automates the creation, connection,
and cleanup of the workspace on FlashBlade.

[scratch-space.yaml](scratch-space.yaml).

The yaml file creates two pods that both mount a shared “/scratch” directory
for collaborative work. This illustrates the core steps necessary to use a
persistentVolumeClaim to attach a volume to pods in a Deployment.

To use this shared scratch space, each user connects to a pod and spawns a
shell by running ‘exec’ on one of the pod instances:

```
> kubectl exec -it scratchspace--7877bc68f9-s8nxw /bin/bash
```

Once this shell has been started, the user can collaborate using the ‘/scratch’
directory and all results will be visible to users in other pods.

What if a third user wants to also collaborate? Adding additional workspaces
that share the same filesystem is as simple as:

```
> kubectl scale deployment scratchspace --replicas 3
```

Each additional replica pod is automatically connected to the filesystem, i.e.,
the mounting and unmounting of the filesystem is automatically handled by PSO.

There are many directions to make this example even more useful: 1) add a
unique name to the resources based on a ticket number, 2) automate cleanup of
resources when finished, and 3) use a prebuilt container image with already
installed tools.

## Example 2: NextCloud

[NextCloud](nextcloud-pure.yaml)

NextCloud and OwnCloud are both open-source file sharing applications providing
an alternative to cloud services like DropBox. These applications provide
similar file sharing services across multiple client platforms while keeping
the data in-house instead of an offsite 3rd party cloud; either performance,
cost, or data governance may drive this desire for an in-house service instead
of external. Both applications rely upon persistent storage for the user's data
and metadata.

The NextCloud configuration borrows heavily from existing documentation for
NextCloud.  This configuration is kept simple in order to highlight the usage
of persistent storage.

The config file combines three major elements:
  * Service ‘nextcloud’ that exposes the application externally
  * Deployment ‘nextcloud’ that starts the application server and mounts a filesystem
  * PersistentVolumeClaim that PSO will use to create a matching filesystem on the FlashBlade

Create the nextcloud service as follows:

```
> kubectl apply -f nextcloud-pure.yaml
```

To make this application accessible externally, I used a NodePort service,
which is not recommended in production. To access the NextCloud application,
find the port assigned by Kubernetes:

```
> kubectl get svc nextcloud | grep NodePort
```

Connect to the NextCloud server by using that port number and the ip address of
any Kubernetes node. Production environments should configure load balancers
instead.

### Using InitContainers for Setup

The NextCloud example utilizes an
[initContainer](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
to solve a common problem: configuring the volume to meet the conditions
expected by the application. An initContainer is a container that is run to
completion before the application container is started and must complete
successfully for the application to begin. InitContainers encode the steps
necessary to satisfy pre-conditions or dependencies that the user does not want
to or cannot build into the main application container.  As an example, an
initContainer can download a dataset to the persistent volume for the primary
application to use. 

The NextCloud server expects the "data/" directory to be owned by the www-data
user and to have permissions of 770, otherwise the application fails. To
accomplish this, an initContainer performs two simple commands to achieve this
precondition:

```
     initContainers:
      - name: install
        image: busybox
        command:
        - sh
        - '-c'
        - 'chmod 770 /var/www/html/data && chown www-data /var/www/html/data'
        volumeMounts:
        - name: nc-files
          mountPath: /var/www/html/data
```

Many applications, especially legacy applications not originally built for
containers, expect preconditions on the volumes and initContainers are the
easiest way to achieve this. Beyond the simple example here with busybox,
different container images can be used to place data or install necessary
software on the volume.


## Example 3: OwnCloud

[OwnCloud](owncloud-pure.yaml)

The OwnCloud deployment involves multiple applications working together:
mariadb and redis alongside the owncloud server. These applications use a mix
of block devices and filesystems. For example, MariaDB is an RDBMS service that
requires a block device backed volume whereas OwnCloud stores user data on a
filesystem-backed volume. Derived from docker documentation for
[OwnCloud](https://github.com/owncloud-docker/server).

With PSO, multiple applications and volume types can be mixed and modified
easily. For example, switching the Redis application between “pure-file” and
“pure-block” is as simple as changing the value in the yaml config before
deploying.

Running multiple different applications on FlashBlade works well because the
system is 1) built to support high concurrency and many simultaneous
applications, 2) scales-out seamlessly and non-disruptively, and 3) is
architected natively to leverage the mixed IO performance of NAND flash.

Deploy this set of applications and volumes together with the following
command:

```
> kubectl apply -f owncloud-pure.yaml
```

Note that there is a helm chart for OwnCloud
[here](https://github.com/helm/charts/tree/master/stable/owncloud) but it is
currently not working due to a failed connection between OwnCloud and mariadb.
With the default storageClass set to “pure-file,” this helm chart would
automatically use FlashBlade as the backing store.

### Quick Tips for Usability

Below are some quick-tips that I found useful in running the OwnCloud application.

Run OwnCloud reporting commands internally:
```
kubectl exec oc-owncloud-0 occ user:report
```

Log in to the server and look around at the contents of the filesystem.
```
kubectl exec -it oc-owncloud-0 /bin/bash
```

## Summary

A good software engineer always looks to automate tedious and error-prone tasks
in order to increase the team's productivity. Running Kubernetes and the Pure
Storage Orchestrator automates almost all storage related tasks: filesystem
creation, mounting, and deletion. The result is simple and agile infrastructure
that *just works*.
