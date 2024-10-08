# Multi-Region Active-Active Setup for Camunda 8

Note: This Helm profile requires [version 8.3.2 (or later) of Camunda's Helm chart](https://github.com/camunda/camunda-platform-helm/releases/tag/camunda-platform-8.3.2).

## WARNING
The provided Helm profiles are designed to demonstrate the technical feasibility of running Camunda 8 in a multi-region environment. However, these Helm profiles are not part of Camunda’s standard release process and are considered a community project.

For current limitations and an official guide, please refer to the official documentation: [Camunda Multi-Region Setup Guide](https://docs.camunda.io/docs/self-managed/concepts/multi-region/dual-region/).


Given the complexity of multi-region setups, we strongly recommend reaching out to your customer success manager or Camunda representative for expert guidance and a thorough review of your configuration. If the available options in the official documentation do not meet your multi-region needs, we advise submitting a feature request for further assistance.

If your multi-region requirements are not met with the currently available options described in the official documentation, please raise a feature request.


## Prerequisite: Kubernetes Cross-Cluster Communication

A multi-region setup in Kubernetes really means a multi-cluster setup and that comes with a networking challenge: How to manage connectivity between my pods across different Kubernetes clusters? You should setup proper firewall rules and correctly route traffic among the pods. For that you have many options (in nor particular order):
* ["DNS Chainging" with kube-dns](https://youtu.be/az4BvMfYnLY?si=RmauCqchHwsmCDZZ&t=2004): That's the option we took in this example. We setup kube-dns automatically through a [python script](https://github.com/camunda-community-hub/camunda-8-helm-profiles/blob/main/google/multi-region/active-active/setup-zeebe.py) to route traffic to the distant cluster based on the namespace. This requires to have different namespaces in each cluster.
* [Istio](https://medium.com/@danielepolencic/scaling-kubernetes-to-multiple-clusters-and-regionss-491813c3c8cd) ([video](https://youtu.be/_8FNsvoECPU?si=dUOFwaaUxRroj8MP))
* [Skupper](https://medium.com/@shailendra14k/deploy-the-skupper-networks-89800323925c#:~:text=Skupper%20creates%20a%20service%20network,secure%20communication%20across%20Kubernetes%20clusters.)
* [Linkerd multi-cluster communication](https://linkerd.io/2.14/features/multicluster/)
* [Submariner](https://submariner.io/)
* [KubeStellar](https://kubestellar.io)
* [Google Kubernetes Engine (GKE) Fleet Management](https://cloud.google.com/kubernetes-engine/docs/fleets-overview)
* [Azure Kubernetes Fleet Manager](https://azure.microsoft.com/en-us/products/kubernetes-fleet-manager)
* [SIG Multicluster](https://multicluster.sigs.k8s.io/guides/)
* etc.

## Special Case: Dual-Region Active-Active

We are basing our dual-region active-active setup on standard Kubernetes features that are cloud-provider-independent. The heavy-lifting of the setup is done by `kubectl` and `helm`. Python and Make are just used for scripting combinations of kubectl and Helm. These scripts could be easily ported to Infrastructure as Code languages. You can run `make --dry-run` on any of the Makefile targets mentioned below to see which kubectl and Helm commands are used.

### Initial Setup

### Prepare installation

You should clone this repository as well as the [second one](https://github.com/camunda-consulting/camunda-platform-helm) locally. This repository  references the first one in the makefile of each region : https://github.com/camunda-community-hub/camunda-8-helm-profiles/blob/d9168169ffe368a817e67c8cd70217ace1071285/google/multi-region/active-active/region0/Makefile#L29. So depending on how you clone these repositories you may want to change that line.

The installation configurations are available at the beginning of these makefiles (clustername, region, project, machine type, etc). For this example, we decided to name our namespaces as our regions for an easier readability. You may want to change this. In such a case and if you want to use setup-zeebe.py to configure kube-dns,  this script should be updated accordingly.

#### Prepare Kubernetes Clusters

Edit [region0/Makefile](region0/Makefile) and [region1/Makefile](region1/Makefile)
and adjust `project`, `region`, and `clusterName`.
We recommend to include `region-0` into the `clusterName`
to abstract away from physical region names like `us-east1-b`.
The physical region name will however be used as a Kubernetes namespace.

```sh
cd region0
make kube
cd ../region1
make kube
cd ..
```

#### Configure Kube-dns

Note : this step should not be executed if you plan to user another solution for cross cluster communication.
Edit the Python script [setup-zeebe.py](./setup-zeebe.py)
and adjust the lists of `contexts` and `regions`.
To get the names of your kubectl "contexts" for each of your clusters, run:

```sh
kubectl config get-contexts
```

Then run that script to adjust the DNS configuration of both Kubernetes clusters
so that they can resolve each others service names.

```sh
./setup-zeebe.py
```
For troubleshooting, you can test the DNS connection as described in the [Kubernetes Documentation on Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/) (you could also build [your own](https://github.com/wkruse/dnsutils-docker) [dnsutils image](https://github.com/docker-archive/dnsutils) if you can't pull one). 

#### Enabling Firewall rules
To allow communication between the zeebe nodes and from the zeebe nodes to the Elasticsearch, we need to authorize the traffic.
The rule should have the correct :
- Target tags : can be retrieved from VM Instance => Network tags
- IP ranges : can be retrieved from cluster detailed => Cluster Pod IPv4 range (default)
- Protocols and ports : tcp:26502 and tcp:9200

#### Installing Camunda

Adjust `ZEEBE_BROKER_CLUSTER_INITIALCONTACTPOINTS` in [region0/camunda-values.yaml](region0/camunda-values.yaml) and [region1/camunda-values.yaml](region1/camunda-values.yaml)


```sh
cd region0
make
cd ../region1
make
```

##### What happens behind the scenes?

`make` is going to run commands similar to the following output of `make --dry-run` for region 0:

```sh
gcloud config set project camunda-researchanddevelopment
gcloud container clusters get-credentials cdame-region-0 --region europe-west4-b
kubectl create namespace europe-west4-b
kubectl config set-context --current --namespace=europe-west4-b
kubectl create secret generic gcs-backup-key --from-file=gcs_backup_key.json=gcs_backup_key.json
helm install --namespace europe-west4-b camunda ../../../../../camunda-platform-helm/charts/camunda-platform -f camunda-values.yaml --skip-crds
```

and for region 1:

```sh
gcloud config set project camunda-researchanddevelopment
gcloud container clusters get-credentials cdame-region-1 --region europe-west1-b
kubectl create namespace europe-west1-b
kubectl config set-context --current --namespace=europe-west1-b
kubectl create secret generic gcs-backup-key --from-file=gcs_backup_key.json=gcs_backup_key.json
helm install --namespace europe-west1-b camunda ../../../../../camunda-platform-helm/charts/camunda-platform -f camunda-values.yaml --skip-crds
```

If you don't want to use make you can also run the above commands
manually or with some other automation tool.

#### Verification

##### Zeebe

You can check the status of the Zeebe cluster using (in any region):

```sh
make zbctl-status
```

The output should look something like this
(Note how brokers alternate between two Kubernetes namespaces
`europe-west4-b` and `us-east1-b` that represent the physical regions,
in which they are hosted.):

```sh
Cluster size: 8
Partitions count: 8
Replication factor: 4
Gateway version: 8.2.8
Brokers:
  Broker 0 - camunda-zeebe-0.camunda-zeebe.europe-west4-b.svc:26501
    Version: 8.2.8
    Partition 1 : Leader, Healthy
    Partition 6 : Leader, Healthy
    Partition 7 : Leader, Healthy
    Partition 8 : Leader, Healthy
  Broker 1 - camunda-zeebe-0.camunda-zeebe.us-east1-b.svc:26501
    Version: 8.2.8
    Partition 1 : Follower, Healthy
    Partition 2 : Follower, Healthy
    Partition 7 : Follower, Healthy
    Partition 8 : Follower, Healthy
  Broker 2 - camunda-zeebe-1.camunda-zeebe.europe-west4-b.svc:26501
    Version: 8.2.8
    Partition 1 : Follower, Healthy
    Partition 2 : Leader, Healthy
    Partition 3 : Leader, Healthy
    Partition 8 : Follower, Healthy
  Broker 3 - camunda-zeebe-1.camunda-zeebe.us-east1-b.svc:26501
    Version: 8.2.8
    Partition 1 : Follower, Healthy
    Partition 2 : Follower, Healthy
    Partition 3 : Follower, Healthy
    Partition 4 : Follower, Healthy
  Broker 4 - camunda-zeebe-2.camunda-zeebe.europe-west4-b.svc:26501
    Version: 8.2.8
    Partition 2 : Follower, Healthy
    Partition 3 : Follower, Healthy
    Partition 4 : Leader, Healthy
    Partition 5 : Leader, Healthy
  Broker 5 - camunda-zeebe-1.camunda-zeebe.us-east1-b.svc:26501
    Version: 8.2.8
    Partition 3 : Follower, Healthy
    Partition 4 : Follower, Healthy
    Partition 5 : Follower, Healthy
    Partition 6 : Follower, Healthy
  Broker 6 - camunda-zeebe-3.camunda-zeebe.europe-west4-b.svc:26501
    Version: 8.2.8
    Partition 4 : Follower, Healthy
    Partition 5 : Follower, Healthy
    Partition 6 : Follower, Healthy
    Partition 7 : Follower, Healthy
  Broker 7 - camunda-zeebe-3.camunda-zeebe.us-east1-b.svc:26501
    Version: 8.2.8
    Partition 5 : Follower, Healthy
    Partition 6 : Follower, Healthy
    Partition 7 : Follower, Healthy
    Partition 8 : Follower, Healthy
```
For troubleshooting and testing, you can also install everything into a single Kubernetes cluster without the DNS chaining configuration but into the region namespaces. That way you can test that the Camunda configuration is correct and reduce troubleshooting to networking and DNS as described above.

##### Operate

Operate has a defect for now and if the zeebe brokers negotiation takes too long, Operate will look "healthy" but will not start the importer. You may need to delete the operate pod to force its recreation once the zeebe cluster is healthy.


##### Elasticsearch

Elastic doesn't support a dual-region active-active setup. You would need a tie breaker in a 3rd region : https://www.elastic.co/guide/en/elasticsearch/reference/current/high-availability-cluster-design-large-clusters.html#high-availability-cluster-design-two-zones
Cross-Cluster Replication is an Active-Passive setup that doesn't fit the current requirement.

So the current approach is to have 2 ES clusters in each region with their own Operate,Tasklist, Optimize on top of it. In case of disaster (loosing a region), procedure would be to pause the exporters & then start the failOver.
Once the failback is started, resume the exporters.

You can check the status of the Elasticsearch cluster using:

```sh
make elastic-nodes
```

### Disaster

In case of disaster, if a region is lost, attempting to start a process instance would lead to an exception :

io.grpc.StatusRuntimeException: RESOURCE_EXHAUSTED: Expected to execute the command on one of the partitions, but all failed; there are no more partitions available to retry. Please try again. If the error persists contact your zeebe operator

the procedure would be to :
* start temporary nodes that will restore the quorum in the surviving region
* restore disaster region
    * restore missing nodes in the disastered region (wihtout operate and tasklist)
	* pause exporters
	* take Operate/Tasklist snapshots in the surviving region
	* restore snapshots in the disastered region
	* resume exporters
* clean the temporary nodes from the surviving region
* restore the initial setup

##### pause exporters

TODO: write a makefile target to pause exporters in the surviving region


##### start temporary nodes (failOver)

In the surviving region, use the "make fail-over-regionX" to create the temporary nodes with the partitions to restore the qorum.
If region0 survived, the command would be

```sh
cd region0
make fail-over-region1
```

If region1 survived, the command would be

```sh
cd region1
make fail-over-region0
```

> :information_source: As a result, we have a working zeebe engine but the exporters are stucked because one ES target is not yet available.

##### restore missing nodes in the disastered region (failBack)

Once you're able to restore the disaster region, you don't want to restart all nodes. Else you will end-up with some brokerIds duplicated (from the failOver). So instead, you want to restart only missing brokerIds.
```sh
cd region0
make fail-back
```

> :information_source: This will indeed create all the brokers. But half of them (the ones in the failOver) will not be started (start script is altered in the configmap). Operate and tasklist are not restarted on purpose to avoid touching ES indices.

##### pause exporters

You now have 2 active regions again and we want to have 2 consistent ES clusters. We will pause exporters, take snapshots in the surviving region, restore them into the restored region and resume exporters.
```sh
cd region0
make pause-exporters
```

##### Take Operate/Tasklist snapshots

A preriquisite is that ES is configured to take/restore snapshots (skip if that was already done) :
```sh
cd region0
make prepare-elastic-backup-repo
```

We have paused exporters. We can safely take our applications snapshots. 
```sh
cd region0
make operate-snapshot
```

##### Restore Operate/Tasklist snapshots in the lost region

A preriquisite is that ES is configured to take/restore snapshots (skip if that was already done) :
```sh
cd region1
make prepare-elastic-backup-repo
```

We can restore our applications snapshots
```sh
cd region1
make restore-operate-snapshot
```

##### resume exporters

We now have our 2 regions with our 2 ES in the same state and we can resume exporters
```sh
cd region0
make resume-exporters
```

##### clean the temporary nodes (prepare transition to initial state)

You can safely delete the temporary nodes from surviving region as the quorum is garantied by the restored brokers in the disastered region.

```sh
cd region0
make clean-fail-over-region1
```

##### restore the initial setup (back to normal)

You now want to recreate the missing brokers in the disastered region.

```sh
cd region1
make fail-back-to-normal
```

> :information_source: This will change the startup script in the configmap and delete the considered pods (to force recreation). The pod deletion should be changed depending on your initial setup.


#### Some extra work before the magic appears :

You need to [create Google Cloud Storage Bucket](https://console.cloud.google.com/storage/create-bucket). We named ours cdame-elasticsearch-backup. We created a regional one.

You need to [set up a service account](https://console.cloud.google.com/iam-admin/serviceaccounts/create) that will be used by Elasticsearch to Backup. You should grant it the "Storage Admin" role to allow it to access the bucket.

Download the JSON API key and save it in each region as gcs_backup_key.json
## FAQ
### Broker names are same in all the regions instead of incremental, e.g. there is a camunda-zeebe-0 in every region?
These pod names are correct. Kubernetes counts each stateful set starting from zero. The fully qualified names are still globally unique due to the different namespace names, e.g. camunda-zeebe-0.camunda-zeebe.**eastus**.svc and camunda-zeebe-0.camunda-zeebe.**centralus**.svc.
The node ids of the brokers are important to be unique, e.g. even numbers in centralus and odd numbers in eastus.
