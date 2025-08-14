# 🚀 Running a MySQL InnoDB ClusterSet Across Multiple Kubernetes Clusters Using Cilium


## 📘 Introduction

This project demonstrates how to deploy a MySQL InnoDB ClusterSet across two Kubernetes clusters interconnected using Cilium Cluster Mesh.

We’re building a unified, multi-cluster MySQL architecture with high availability and service discovery — all running on a single VM using lightweight Kubernetes environments.

⸻

## 🔧 Infrastructure Overview

The infrastructure is composed of a single cloud-hosted virtual machine, running two KIND (Kubernetes IN Docker) clusters. KIND offers a fast and convenient way to spin up Kubernetes clusters locally in Docker containers, making it ideal for development, testing, and experimentation.

We use Cilium as the CNI (Container Network Interface) plugin for both clusters — not just for its performance, but for its advanced networking capabilities, including:
	•	⚡️ Built-in LoadBalancer support
	•	📡 Efficient L2 routing and service announcements
	•	🔐 Powerful eBPF-based network policies
	•	🔗 Cluster Mesh for seamless multi-cluster connectivity

Thanks to Cilium Cluster Mesh, pods and services in each cluster can discover and communicate across cluster boundaries, as if they were in the same flat network — regardless of the Kubernetes distribution or where the clusters are hosted.

⸻

## 🐬 MySQL Deployment

On both clusters, we’ve deployed MySQL Enterprise using the official MySQL Operator for Kubernetes, enabling advanced features like:
	•	Native InnoDB Cluster support
	•	Automatic failover and recovery
	•	Seamless cross-cluster replication

> ⚠️ This repository does not cover the installation of the MySQL Operator or the deployment of the InnoDB clusters in detail.
For step-by-step instructions, please refer to the official documentation:
👉 [MySQL Operator for Kubernetes - Documentation](https://dev.mysql.com/doc/mysql-operator/en/)
👉 [Deploy a MySQL InnoDB Cluster on Kubernetes with MySQL Operator](https://github.com/colussim/mysql-innodb-k8s-operator)

 > **⚠️ Behavior notice (MySQL Operator 9.4 for Kubernetes)**
>
> When you deploy an **InnoDBCluster**, the Operator **automatically creates a
> ClusterSet** with the **same name** as the cluster.
>
> **Implications**
> - You may see a `ClusterSet` resource even if you only asked for a single cluster.
> - **Deleting the ClusterSet will also delete the associated InnoDBCluster.**
> - Plan your naming accordingly and avoid removing the ClusterSet unless you intend
>   to remove the cluster as well.
>
> **Why this happens (at a high level)**
> - The Operator prepares for future scale‑out (adding replica clusters) without extra steps.
>
> **Recommendation**
> - If you want a standalone setup, simply keep the auto‑created ClusterSet and operate your single cluster as usual. If you plan multi‑site later, you can attach additional clusters to this ClusterSet.
 
---


## 🛠️ Step 1 – Setting Up Cluster Mesh

Before connecting the clusters with Cilium Cluster Mesh, make sure you already have two Kubernetes clusters running with Cilium installed as the CNI.

In our setup, both clusters were created using KIND (Kubernetes IN Docker).

**🔍 Check Existing KIND Clusters**

To view the existing KIND clusters on your machine, run:
```bash
kind get clusters

```

This should output something like:
```bash
kubectl config get-contexts

```

Since we have two KIND clusters, we also have two Kubernetes contexts configured. You can list them using:
```bash
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
          kind-k8sdemos2   kind-k8sdemos2   kind-k8sdemos2   
*         kind-k8sdemos3   kind-k8sdemos3   kind-k8sdemos3   

```

To switch context, use:
```bash
kubectl config use-context kind-k8sdemos2

```

**🐬 Checking MySQL InnoDB Cluster**

To get a list of deployed InnoDB Clusters managed by the operator in each Kubernetes cluster, use:
```bash
kubectl get innodbclusters -A --context=kind-k8sdemos2

NAMESPACE   NAME              STATUS   ONLINE   INSTANCES   ROUTERS   TYPE      AGE
innodb01    innodbcluster01   ONLINE   3        3           2         UNKNOWN   9h02m

```

```bash
kubectl get innodbclusters -A --context=kind-k8sdemos3

NAMESPACE   NAME              STATUS   ONLINE   INSTANCES   ROUTERS   TYPE      AGE
innodb02    innodbcluster02   ONLINE   3        3           2         UNKNOWN   8h11m
```


**🛠️ Additional Requirements for Native Routing Paths**

To enable native routing across clusters in Cilium Cluster Mesh, each cluster’s Cilium must be configured with a native routing CIDR that covers all PodCIDR ranges of the connected clusters.And all clusters across a Cluster Mesh must be configured with the same *maxConnectedClusters* value.



**🔍 Identify PodCIDR Ranges of Each Cluster**

Clusters usually allocate PodCIDRs from the private address space *10.0.0.0/8.* For example, a routing CIDR of *10.0.0.0/8* can cover all clusters if their PodCIDRs fall within this range.

You can verify the PodCIDR assigned to nodes by running:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" - "}{.spec.podCIDR}{"\n"}{end}'
```

Example output:

```bash

k8sdemos2-control-plane - 10.10.0.0/24
k8sdemos2-worker - 10.10.1.0/24
k8sdemos2-worker2 - 10.10.2.0/24
k8sdemos2-worker3 - 10.10.3.0/24

```

If your clusters use PodCIDRs in the 10.x.x.x range, then using the global routing CIDR 10.0.0.0/8 will cover all PodCIDRs.

**⚙️ Configure Cilium Native Routing CIDR**

Repeat the PodCIDR check on each cluster. If the PodCIDRs fit in the 10.0.0.0/8 range for all clusters, proceed to configure the Cilium ConfigMap:

```bash
kubectl patch configmap cilium-config -n kube-system --type merge -p '{"data": {"ipv4-native-routing-cidr": "ipv4-native-routing-cidr: 10.0.0.0/8"}}

```

**🔄 Restart Cilium DaemonSet**

```bash
kubectl rollout restart ds/cilium -n kube-system

```

**✅ Verify Configuration**

To confirm the settings have been applied correctly, run:

```bash
kubectl -n kube-system exec -it ds/cilium -- cilium status

```

or use a command (with the wait option to wait for all Cilium services to start): 
```bash
cilium status --wait

```

Example output (if all is well 😀):

```bash
   /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled

DaemonSet              cilium                   Desired: 4, Ready: 4/4, Available: 4/4
DaemonSet              cilium-envoy             Desired: 4, Ready: 4/4, Available: 4/4
Deployment             cilium-operator          Desired: 1, Ready: 1/1, Available: 1/1
Deployment             hubble-relay             Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium                   Running: 4
                       cilium-envoy             Running: 4
                       cilium-operator          Running: 1
                       clustermesh-apiserver    
                       hubble-relay             Running: 1
Cluster Pods:          5/5 managed by Cilium
Helm chart version:    1.17.5
Image versions         cilium             quay.io/cilium/cilium:v1.17.5@sha256:baf8541723ee0b72d6c489c741c81a6fdc5228940d66cb76ef5ea2ce3c639ea6: 4
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.32.6-1749271279-0864395884b263913eac200ee2048fd985f8e626@sha256:9f69e290a7ea3d4edf9192acd81694089af048ae0d8a67fb63bd62dc1d72203e: 4
                       cilium-operator    quay.io/cilium/operator-generic:v1.17.5@sha256:f954c97eeb1b47ed67d08cc8fb4108fb829f869373cbb3e698a7f8ef1085b09e: 1
                       hubble-relay       quay.io/cilium/hubble-relay:v1.17.5@sha256:fbb8a6afa8718200fca9381ad274ed695792dbadd2417b0e99c36210ae4964ff: 1

```

> ⚠️ Repeat the modifications if necessary on the second cluster

---

## 🛠️ Step 2 – Prepare the Clusters

We will connect the two clusters using their respective kubectl contexts, which are stored in the environment variables $CLUSTER1 and $CLUSTER2.

The context name corresponds to what you normally pass to kubectl --context.

**🛠️ Specify Cluster Name and ID**

Cilium must be installed on each cluster before proceeding.

Each cluster requires a unique, human-readable name and a numeric cluster ID (between 1 and 255).

📝 Cluster Name Requirements

•	Maximum length: 32 characters
•	Must start and end with a lowercase alphanumeric character (a-z, 0-9)
•	Can contain lowercase alphanumeric characters and dashes (-) in between

It is best practice to assign the cluster name and ID during installation by configuring the Cilium ConfigMap.

Patch ConfigMap (for cluster_id)  in the kube-system namespace:

```bash
kubectl patch configmap cilium-config -n kube-system --type merge -p '{"data": {"cluster-id": "1"}}
```


> ⚠️ Make sure each cluster has a unique name and cluster ID.

🔄 Restart Cilium DaemonSet

```bash
kubectl rollout restart ds/cilium -n kube-system
kubectl rollout restart deployment/cilium-operator -n kube-system

```


> ⚠️ Repeat the modifications if necessary on the second cluster

---

## 🛠️ Step 3 – Enable Cluster Mesh

Run cilium clustermesh enable in each cluster context to:
	•	Deploy the clustermesh-apiserver
	•	Generate and import the required certificates as Kubernetes secrets
	•	Automatically detect the best LoadBalancer type to expose the ClusterMesh control plane

First, define the Kubernetes contexts for both clusters:
```bash
export CLUSTER1="kind-k8sdemos2"
export CLUSTER2="kind-k8sdemos3"

```

Then enable ClusterMesh on each cluster:

```bash
cilium clustermesh enable --context $CLUSTER1 --enable-kvstoremesh=false
cilium clustermesh enable --context $CLUSTER2 --enable-kvstoremesh=false
```

⏳ Wait for ClusterMesh to be Ready

Run the following to wait for all ClusterMesh components (including LoadBalancer IP assignment) to be fully ready:

```bash
cilium clustermesh status --context $CLUSTER1  --wait 

```

```bash
✅ Service "clustermesh-apiserver" of type "LoadBalancer" found
✅ Cluster access information is available:
  - 172.18.250.102:2379
⌛ Waiting (0s) for deployment clustermesh-apiserver to become ready: only 0 of 1 replicas are available
✅ Deployment clustermesh-apiserver is ready
ℹ️  KVStoreMesh is disabled

🔌 No cluster connected
🔀 Global services: [ min:-1 / avg:0.0 / max:0 ]

```

**🔗 Connect Clusters**

Establish the ClusterMesh link (one-way command, connection becomes bi-directional automatically):

```bash
cilium clustermesh connect --context $CLUSTER1 --destination-context $CLUSTER2

```

The output will look something like this:

```bash

 Extracting access information of cluster kind-k8sdemos2...
🔑 Extracting secrets from cluster kind-k8sdemos2...
ℹ️  Found ClusterMesh service IPs: [172.18.250.102]
✨ Extracting access information of cluster kind-k8sdemos3...
🔑 Extracting secrets from cluster kind-k8sdemos3...
ℹ️  Found ClusterMesh service IPs: [172.18.249.102]
⚠️ Cilium CA certificates do not match between clusters. Multicluster features will be limited!
ℹ️ Configuring Cilium in cluster kind-k8sdemos2 to connect to cluster kind-k8sdemos3
ℹ️ Configuring Cilium in cluster kind-k8sdemos3 to connect to cluster kind-k8sdemos2
✅ Connected cluster kind-k8sdemos2 <=> kind-k8sdemos3!

```

🎉 Congratulations! You have successfully connected your clusters together.

Check Cluster Mesh status :

```bash

cilium clustermesh status --context $CLUSTER1
✅ Service "clustermesh-apiserver" of type "LoadBalancer" found
✅ Cluster access information is available:
  - 172.18.250.102:2379
✅ Deployment clustermesh-apiserver is ready
ℹ️  KVStoreMesh is disabled

✅ All 4 nodes are connected to all clusters [min:1 / avg:1.0 / max:1]

🔌 Cluster Connections:
  - kind-k8sdemos3: 4/4 configured, 4/4 connected

🔀 Global services: [ min:2 / avg:2.0 / max:2 ]

```

```bash

cilium clustermesh status --context $CLUSTER2
✅ Service "clustermesh-apiserver" of type "LoadBalancer" found
✅ Cluster access information is available:
  - 172.18.249.102:2379
✅ Deployment clustermesh-apiserver is ready
ℹ️  KVStoreMesh is disabled

✅ All 4 nodes are connected to all clusters [min:1 / avg:1.0 / max:1]

🔌 Cluster Connections:
  - kind-k8sdemos2: 4/4 configured, 4/4 connected

🔀 Global services: [ min:2 / avg:2.0 / max:2 ]

```

**🔍 Inspect Registered CiliumNodes**

You can list all CiliumNodes (both local and remote) registered in each cluster to verify that ClusterMesh is functioning correctly.

Run:

```bash

kubectl get ciliumnodes --context $CLUSTER1

```

Example output:

```bash

NAME                      CILIUMINTERNALIP   INTERNALIP   AGE
k8sdemos2-control-plane   10.10.0.243        172.18.0.5   26h
k8sdemos2-worker          10.10.3.209        172.18.0.4   26h
k8sdemos2-worker2         10.10.1.111        172.18.0.2   26h
k8sdemos2-worker3         10.10.2.164        172.18.0.3   26h

```

✅ If nodes from both clusters appear in each other’s list (when viewed via cilium status), it confirms that ClusterMesh discovery is working as expected.

---

## 🛠️ Step 4 – Multi-Cluster Services API

By default Cilium does not automatically configure CoreDNS to include the DNS search suffixes of other clusters when using ClusterMesh. 

To enable DNS resolution across clusters for Multi-Cluster Services, you have two options:
	1.	Manually edit the CoreDNS ConfigMap to add the required search domains.
	2.	Install the Cilium Multi-Cluster Services (MCS) API CRDs.

**🌐 About Cilium MCS API CRDs**

The Cilium MCS API CRDs orchestrate DNS resolution and service connectivity between multiple Kubernetes clusters.

At the DNS level, they:
	•	Manage service names across clusters
	•	Enable seamless cross-cluster service discovery
	•	Allow applications in different clusters to access services using consistent and centralized DNS names

This greatly simplifies multi-cluster communication, while preserving Cilium’s security model and network policies.

Cilium’s Multi-Cluster Services API allows you to export and import services between clusters. This means you can export a service from one cluster and make it accessible in another cluster without having to manually configure DNS entries.

It is generally preferable to use the Cilium MCS API rather than the legacy Global Services feature, as MCS-API follows the Kubernetes multicluster standard (ServiceExport / ServiceImport), offers better interoperability, and is the direction Cilium is actively developing and maintaining.


⚠️ **Note:** In Cilium version [**1.18**](https://docs.cilium.io/en/stable/network/clustermesh/mcsapi/#gs-clustermesh-mcsapi), the CoreDNS `multicluster` plugin is **not automatically enabled** when using the Multi-Cluster Services API.  
This behavior will change in [**v1.19**](https://docs.cilium.io/en/latest/network/clustermesh/mcsapi/), where CoreDNS can be automatically configured if `clustermesh.mcsapi.corednsAutoConfigure.enabled` is set to `true`.  

For Cilium **1.18**, you must manually update the CoreDNS ConfigMap to enable the `multicluster` plugin. 

📌 **Why Patch CoreDNS and Modify its ConfigMap (MySQL InnoDB ClusterSet Context)**

While the MCS API handles DNS for *.svc.clusterset.local, MySQL InnoDB ClusterSet has additional requirements:

1.	Full Pod FQDN Resolution
ClusterSet’s AdminAPI identifies each MySQL instance by its exact fully-qualified domain name and validates it during TLS handshakes.
If the hostname in DNS does not match the instance’s report_host, replication setup fails.
3.	Local Domain Differences
Your clusters use different local DNS suffixes (e.g. k8sdemos2.local, k8sdemos3.local).
Without CoreDNS configuration, pods from one cluster cannot resolve the other’s local FQDNs.
4.	NodePort for CoreDNS Service
By default, CoreDNS runs as a ClusterIP service — only accessible inside the cluster.
Patching it to NodePort gives you a stable, externally reachable DNS endpoint for inter-cluster lookups without relying on pod IPs (which are ephemeral).

💡 In short:
	•	Patch ConfigMap → add other cluster’s domain(s) and enable the multicluster plugin.
	•	Patch Service → expose CoreDNS on a NodePort to allow direct DNS queries between clusters when needed.

➡️  Patch CoreDNS service to NodePort (on each cluster):
```bash
kubectl patch svc kube-dns -n kube-system -p '{"spec": {"type": "NodePort"}}' --context <CLUSTER_CONTEXT>
```

➡️ Get the NodePort (on each cluster):
```bash
kubectl get svc kube-dns -n kube-system --context <CLUSTER_CONTEXT>
```

You will see something like:
```bash
AME      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
kube-dns  NodePort   10.31.0.10     <none>        53:32656/UDP,53:32656/TCP... 6d
```
Here, 32656 is the NodePort.

➡️ Get thethe node’s IP address (on each cluster):
```bash
kubectl get nodes --context <CLUSTER_CONTEXT> \
  -o 'custom-columns=NAME:.metadata.name,INTERNAL-IP:.status.addresses[?(@.type=="InternalIP")].address'

```

Use the INTERNAL-IP  of all of the nodes.

➡️ Update CoreDNS (on each cluster):

Update CoreDNS forwarding: In the other cluster’s CoreDNS config, forward to <node-ip>:<nodeport> instead of the ClusterIP.

```bash
kubectl edit configmap coredns -n kube-system --context <CLUSTER_CONTEXT>
```

Example in CLUSTER2 for the domain k8sdemos2.local in CLUSTER1
```yaml
k8sdemos2.local:53 {
  forward . <node-ip>:32656 <node-ip>:32656 <node-ip>:32656 <node-ip>:32656
}
```

Replace <node-ip>:32656 with the actual IPs and NodePort.
Replace <k8sdemos2.local> with domain in CLUSTER1

You should do the reverse on CLUSTER1 forward the domain to CLUSTER2

🔄 Restart CoreDNS

```bash

kubectl --context <CLUSTER_CONTEXT> rollout deployment -n kube-system coredns
```


➡️  Seed ClusterSet using the **local** FQDN of a specific instance

When invoking AdminAPI (e.g., `dba.createReplicaCluster()`), connect to a **specific** seed instance via its **local** FQDN
(the same as its `report_host`), not via `clusterset.local`:

```bash
# From Cluster 1, reach a specific instance in Cluster 2
kubectl -n innodb01 --context $CLUSTER1 exec -it innodbcluster01-0 --   mysqlsh --uri root@innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local -p
```


**⚠️ Prerequisite**

To use the Cilium MCS API, your cluster must run CoreDNS version 1.12.2 or later.

You can check your CoreDNS version with:
```bash

kubectl -n kube-system get deployment coredns -o=jsonpath='{.spec.template.spec.containers[0].image}' --context <CLUSTER_CONTEXT>

```

Example output:
```bash
registry.k8s.io/coredns/coredns:v1.12.2
```

If needed, you can upgrade CoreDNS to the required version using:
```bash
kubectl -n kube-system set image deployment/coredns coredns=registry.k8s.io/coredns/coredns:v1.12.2 --context <CLUSTER_CONTEXT>
```



✅ In this guide, we choose Option 2: using the Cilium MCS API CRDs.

**🛠️ Deploy the Cilium Multi-Cluster Services (MCS) API**

You first need to install the required MCS-API CRDs:
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/mcs-api/62ede9a032dcfbc41b3418d7360678cb83092498/config/crd/multicluster.x-k8s.io_serviceexports.yaml --context <CLUSTER_CONTEXT>

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/mcs-api/62ede9a032dcfbc41b3418d7360678cb83092498/config/crd/multicluster.x-k8s.io_serviceimports.yaml --context <CLUSTER_CONTEXT>
```


Patch Cilium ConfigMap  in the kube-system namespace:

```bash

kubectl patch configmap cilium-config -n kube-system  --type merge --patch '{"data": {"clustermesh-enable-mcs-api": "true"}}' --context <CLUSTER_CONTEXT>
```


🔄 Restart Cilium DaemonSet

```bash

kubectl rollout restart daemonset/cilium -n kube-system --context <CLUSTER_CONTEXT>
kubectl rollout restart deployment/cilium-operator -n kube-system --context <CLUSTER_CONTEXT>
```

> ⚠️  Replace <CLUSTER_CONTEXT> with your cluster context.

Alternatively, you can use Helm :
```bash
helm upgrade cilium cilium/cilium --version 1.18.0 \
--namespace kube-system \
--reuse-values \
--set clustermesh.enableMCSAPISupport=true --context <CLUSTER_CONTEXT>
```


⚠️ CoreDNS Not Automatically Configured for MCS API

👉 Please follow the steps below to configure CoreDNS for MCS support :

We will create a dedicated role and bind it to the CoreDNS service account and then patch the CoreDNS ConfigMap:
```bash

# Adding RBAC to read SericeImports
kubectl --context <CLUSTER_CONTEXT> create clusterrole coredns-mcsapi \
   --verb=list,watch resource=serviceimports.multicluster.x-k8s.io
   
kubectl --context <CLUSTER_CONTEXT> create clusterrolebinding coredns-mcsapi \
   --clusterrole=coredns-mcsapi --serviceaccount=kube-system:coredns 

# Configure CoreDNS to support MCS-API
kubectl --context <CLUSTER_CONTEXT> get configmap -n kube-system coredns -o yaml | \
   sed -e 's/cluster\.local/k8sdemos2.local clusterset.local/g' | \
   sed -E 's/^(.*)kubernetes(.*)\{/\1kubernetes\2{\n\1   multicluster clusterset.local/' | \
   kubectl --context <CLUSTER_CONTEXT> replace -f-

```

🔄 Restart CoreDNS

```bash

kubectl --context <CLUSTER_CONTEXT> rollout deployment -n kube-system coredns

```

> ⚠️  Replace <CLUSTER_CONTEXT> with your cluster context.


With CoreDNS now patched for multi-cluster DNS, we can export our MySQL InnoDBCluster services via the Kubernetes Multi-Cluster Services (MCS) API, enabling cross-cluster discovery through the svc.clusterset.local domain.


---

## 🛠️ Step 5 – Export **Cluster‑Level** MySQL Services

In our setup, we keep **cluster-level** exports for MySQL InnoDBCluster services so apps and tooling can use
standard `*.svc.clusterset.local` names.  
We also create and export **per-pod** services, but we do not actually use them, because MySQL uses each instance’s
`report_host` under the local domains (`*.svc.k8sdemosX.local`), and we have already made those resolvable cross-cluster by exposing
CoreDNS via **NodePort** and adding **forward** rules (see Step 4).

- ✅ Keep: **cluster-level** `ServiceExport` for `innodbcluster01-instances` and `innodbcluster02-instances`.
- ➕ Create: **per-pod** exports (for completeness/consistency), but not used in our configuration since `report_host` names are already resolvable cross-cluster.
---

## 1) Ensure namespaces exist on **both** clusters

MCS creates `ServiceImport` objects in the **same namespace** as the exported `Service`. The target namespace **must already exist** on the importing cluster.

Create missing namespaces so each side can import the other’s service:

```bash
# On Cluster 1 (so it can import innodb02 services)
kubectl --context $CLUSTER1 create namespace innodb02

# On Cluster 2 (so it can import innodb01 services)
kubectl --context $CLUSTER2 create namespace innodb01 
```

---

## 2) Add Cilium annotations on the **cluster‑level** Services (before exporting)

We annotate the InnoDBCluster **instance service** (the headless/load‑balanced service that fronts the pods) to make it
eligible for global sync and remote consumption through Cilium/MCS:

```bash
# On Cluster 1 (service in namespace innodb01)
kubectl --context $CLUSTER1 -n innodb01 patch service innodbcluster01-instances   -p '{"metadata":{"annotations":{"service.cilium.io/affinity":"remote","service.cilium.io/global":"true","service.cilium.io/global-sync-endpoint-slices":"true","service.cilium.io/shared":"true"}}}'

# On Cluster 2 (service in namespace innodb02)
kubectl --context $CLUSTER2 -n innodb02 patch service innodbcluster02-instances   -p '{"metadata":{"annotations":{"service.cilium.io/affinity":"remote","service.cilium.io/global":"true","service.cilium.io/global-sync-endpoint-slices":"true","service.cilium.io/shared":"true"}}}'
```

> If the service names differ in your environment, adjust accordingly.  
> These annotations help Cilium treat the service as globally shared and synchronize endpoint slices across clusters.

---

## 3) Export the **cluster‑level** Services

Create a `ServiceExport` in the **same namespace** as each service:

```yaml
# Cluster 1
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: innodbcluster01-instances
  namespace: innodb01
---
# Cluster 2
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: innodbcluster02-instances
  namespace: innodb02
```

Apply in each cluster:

```bash
kubectl --context=$CLUSTER1 apply -f serviceexport-innodb01.yaml
kubectl --context=$CLUSTER2 apply -f serviceexport-innodb02.yaml
```

---

## 4) Creating per-pod Services and exporting them

For each pod in your MySQL StatefulSet, create a dedicated ClusterIP Service that selects only that pod, then export it with a ServiceExport resource.

Example for innodbcluster01 (namespace innodb01, 3 replicas innodbcluster01-X):

```yaml

apiVersion: v1
kind: Service
metadata:
  name: innodbcluster01-0
  namespace: innodb01
  annotations:
    service.cilium.io/affinity: remote
    service.cilium.io/global: "true"
    service.cilium.io/global-sync-endpoint-slices: "true"
    service.cilium.io/shared: "true"
spec:
  selector:
    statefulset.kubernetes.io/pod-name: innodbcluster01-0
  ports:
    - name: mysql
      port: 3306
      targetPort: 3306
    - name: mysqlx
      port: 33060
      targetPort: 33060
    - name: gr-xcom
      port: 33061
      targetPort: 33061
  type: ClusterIP
---
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: innodbcluster01-0
  namespace: innodb01

```

Repeat for each pod (innodbcluster01-1, innodbcluster01-2, etc.) in both clusters.And create a yaml file.

Apply in each cluster:

```bash
kubectl --context=$CLUSTER1 apply -f per_pod_services_export_innodbcluster01.yaml
kubectl --context=$CLUSTER2 apply -f per_pod_services_export_innodbcluster02.yaml
```

---

## 4) Verify `ServiceImport` objects appear on the opposite cluster

```bash
# On Cluster 1 (should see imports for innodb02)
kubectl --context $CLUSTER1 get serviceimports -A

# On Cluster 2 (should see imports for innodb01)
kubectl --context $CLUSTER2 get serviceimports -A
```

Expected example (on Cluster 1):
```
NAMESPACE   NAME                           TYPE       IP    AGE
innodb01    innodbcluster01-0           ClusterSetIP   ["10.11.223.96"]    2d4h
innodb01    innodbcluster01-1           ClusterSetIP   ["10.11.226.150"]   2d4h
innodb01    innodbcluster01-2           ClusterSetIP   ["10.11.12.77"]     2d4h
innodb01    innodbcluster01-instances   Headless                           3d
innodb02    innodbcluster02-0           ClusterSetIP   ["10.11.81.99"]     5h11m
innodb02    innodbcluster02-1           ClusterSetIP   ["10.11.190.41"]    5h11m
innodb02    innodbcluster02-2           ClusterSetIP   ["10.11.192.7"]     5h11m
innodb02    innodbcluster02-instances   Headless                           5h10m
```

---

## 5) Test DNS resolution

From each cluster, resolve both forms:
- The other cluster’s **local** per‑pod FQDNs (used by MySQL `report_host`)
- The **cluster‑level** MCS name (handy for apps)

Examples (run from Cluster 1):

```bash
# Must resolve: remote pod's local FQDN (used by report_host)
getent hosts innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local

# Optional: cluster-level MCS name for client apps
getent hosts innodbcluster02-instances.innodb02.svc.clusterset.local
```

---

## Step 6 – Adding Replica Cluster to The InnoDB ClusterSet from DR cluster

First, dissolve the existing InnoDB Cluster on the `innodbcluster02` site and prepare its nodes to be added as a new replica cluster.  
We will create a replica cluster named `myreplica` using the first node of the `innodbcluster02` site as the seed member.

> **Important:** Temporarily stop the MySQL Operator before starting this procedure.  
> If the Operator remains running, it will detect that the InnoDBCluster no longer matches the desired state in its CR and will try to automatically recreate or rejoin Group Replication.  
> This can undo your manual changes, cause unexpected pod restarts, re-form the old cluster, or interfere with the `createReplicaCluster` process.

**Pause the MySQL Operator** on the DR cluster:
```javascript
kubectl -n mysql-operator scale deploy/mysql-operator --replicas=0 --context $CLUSTER2
```


Connect to the primary node on the innodbcluster02 :
```javascript
kubectl --context=$CLUSTER2 -n innodb02 exec -it innodbcluster02-0 -- mysqlsh root:@localhost:3306 -p --js
```

Dissolve innodbcluster02 cluster :
```javascript
JS > var cs2 = dba.getClusterSet();
JS > cs2.status();
{
    "clusters": {
        "innodbcluster02": {
            "clusterRole": "PRIMARY", 
            "globalStatus": "OK", 
            "primary": "innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306"
        }
    }, 
    "domainName": "innodbcluster02", 
    "globalPrimaryInstance": "innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306", 
    "primaryCluster": "innodbcluster02", 
    "status": "HEALTHY", 
    "statusText": "All Clusters available."
}
JS > cs2.dissolve({force:true});

* Dissolving the ClusterSet...

* Synchronizing all members and dissolving cluster 'innodbcluster02'...

The ClusterSet has been dissolved, user data was left intact.

JS >
```

Verify standalone instance state on each node:
```sql
SELECT @@server_id, @@server_uuid, @@gtid_mode, @@enforce_gtid_consistency, @@group_replication_start_on_boot;
```

•	Ensure gtid_mode=ON and enforce_gtid_consistency=ON.
•	Ensure @@group_replication_start_on_boot=0 and no active GR members.
•	Assign unique server_id values across the entire ClusterSet:
```sql
SET PERSIST server_id = <unique_id>;
```

**From the PRIMARY site (innodbcluster01), create the Replica Cluster:**

Connect to the primary node on the innodbcluster01 :
```bash
kubectl --context $CLUSTER1 -n innodb01 exec -it innodbcluster01-1 -- mysqlsh root:@localhost:3306 -p --js
```

Adding Replica Cluster 
```javascript
JS > var cs1 = dba.getClusterSet();
JS > cs1.status()
```

Example output:
```json
{
    "clusters": {
        "innodbcluster01": {
            "clusterRole": "PRIMARY", 
            "globalStatus": "OK", 
            "primary": "innodbcluster01-1.innodbcluster01-instances.innodb01.svc.k8sdemos2.local:3306"
        }
    }, 
    "domainName": "innodbcluster01", 
    "globalPrimaryInstance": "innodbcluster01-1.innodbcluster01-instances.innodb01.svc.k8sdemos2.local:3306", 
    "primaryCluster": "innodbcluster01", 
    "status": "HEALTHY", 
    "statusText": "All Clusters available."
}
```

```javascript
JS > repc = cs.createReplicaCluster("innodbcluster02-0.innodb02.svc.clusterset.local:3306","innodbcluster02",{recoveryMethod: "clone",recoveryProgress: 1, timeout: 10});
```

Example output:
```text
Setting up replica 'innodbcluster02' of cluster 'innodbcluster01' at instance 'innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306'.

A new InnoDB Cluster will be created on instance 'innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306'.
...............
...............

Replica Cluster 'innodbcluster02' successfully created on ClusterSet 'innodbcluster01'.
```

```javascript
JS > repc.status();
```

Example output:
```json
{
    "clusterName": "innodbcluster02", 
    "clusterRole": "REPLICA", 
    "clusterSetReplicationStatus": "OK", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306", 
        "ssl": "REQUIRED", 
        "status": "OK_NO_TOLERANCE", 
        "statusText": "Cluster is NOT tolerant to any failures.", 
        "topology": {
            "innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306": {
                "address": "innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306", 
                "memberRole": "PRIMARY", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "replicationLagFromImmediateSource": "", 
                "replicationLagFromOriginalSource": "", 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "9.4.0"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "domainName": "innodbcluster01", 
    "groupInformationSourceMember": "innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306", 
    "metadataServer": "innodbcluster01-1.innodbcluster01-instances.innodb01.svc.k8sdemos2.local:3306"
}
```

Add the remaining nodes from the DR site:
```javascript
JS > repc.addInstance('innodbcluster02-1.innodb02.svc.clusterset.local:3306',{recoveryMethod: "clone",recoveryProgress: 1});
```

Example output:
```text
.....
The instance 'innodbcluster02-1.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306' was successfully added to the cluster.
```

Add the second node :
```javascript
JS >repc.addInstance('innodbcluster02-2.innodb02.svc.clusterset.local:3306',{recoveryMethod: "clone",recoveryProgress: 1});

```

Example output:
```json
.....
The instance 'innodbcluster02-2.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306' was successfully added to the cluster.

```

Check status of the ClusterSet and the new Replica Cluster:
```javascript
JS > cs1.status({extended:1});
```

Example output:
```json
{
    "clusters": {
        "innodbcluster01": {
            "clusterRole": "PRIMARY", 
            "globalStatus": "OK", 
            "primary": "innodbcluster01-1.innodbcluster01-instances.innodb01.svc.k8sdemos2.local:3306", 
            "status": "OK", 
            "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.", 
            "topology": {
                "innodbcluster01-0.innodbcluster01-instances.innodb01.svc.k8sdemos2.local:3306": {
                    "address": "innodbcluster01-0.innodbcluster01-instances.innodb01.svc.k8sdemos2.local:3306", 
                    "memberRole": "SECONDARY", 
                    "mode": "R/O", 
                    "readReplicas": {}, 
                    "replicationLagFromImmediateSource": "", 
                    "replicationLagFromOriginalSource": "", 
                    "role": "HA", 
                    "status": "ONLINE", 
                    "version": "9.4.0"
                }, 
                "innodbcluster01-1.innodbcluster01-instances.innodb01.svc.k8sdemos2.local:3306": {
                    "address": "innodbcluster01-1.innodbcluster01-instances.innodb01.svc.k8sdemos2.local:3306", 
                    "memberRole": "PRIMARY", 
                    "mode": "R/W", 
                    "readReplicas": {}, 
                    "role": "HA", 
                    "status": "ONLINE", 
                    "version": "9.4.0"
                }, 
                "innodbcluster01-2.innodbcluster01-instances.innodb01.svc.k8sdemos2.local:3306": {
                    "address": "innodbcluster01-2.innodbcluster01-instances.innodb01.svc.k8sdemos2.local:3306", 
                    "memberRole": "SECONDARY", 
                    "mode": "R/O", 
                    "readReplicas": {}, 
                    "replicationLagFromImmediateSource": "", 
                    "replicationLagFromOriginalSource": "", 
                    "role": "HA", 
                    "status": "ONLINE", 
                    "version": "9.4.0"
                }
            }, 
            "transactionSet": "4f01723b-737b-11f0-85fc-b2b76c25a31a:1-14,5feb4d6b-737b-11f0-bc19-b2b76c25a31a:1-480:1000290-1000331"
        }, 
        "innodbcluster02": {
            "clusterRole": "REPLICA", 
            "clusterSetReplication": {
                "applierStatus": "APPLIED_ALL", 
                "applierThreadState": "Waiting for an event from Coordinator", 
                "applierWorkerThreads": 4, 
                "receiver": "innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306", 
                "receiverStatus": "ON", 
                "receiverThreadState": "Waiting for source to send event", 
                "replicationSsl": "TLS_AES_128_GCM_SHA256 TLSv1.3", 
                "replicationSslMode": "REQUIRED", 
                "source": "innodbcluster01-1.innodbcluster01-instances.innodb01.svc.k8sdemos2.local:3306"
            }, 
            "clusterSetReplicationStatus": "OK", 
            "globalStatus": "OK", 
            "status": "OK", 
            "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.", 
            "topology": {
                "innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306": {
                    "address": "innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306", 
                    "memberRole": "PRIMARY", 
                    "mode": "R/O", 
                    "readReplicas": {}, 
                    "replicationLagFromImmediateSource": "", 
                    "replicationLagFromOriginalSource": "", 
                    "role": "HA", 
                    "status": "ONLINE", 
                    "version": "9.4.0"
                }, 
                "innodbcluster02-1.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306": {
                    "address": "innodbcluster02-1.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306", 
                    "memberRole": "SECONDARY", 
                    "mode": "R/O", 
                    "readReplicas": {}, 
                    "replicationLagFromImmediateSource": "", 
                    "replicationLagFromOriginalSource": "", 
                    "role": "HA", 
                    "status": "ONLINE", 
                    "version": "9.4.0"
                }, 
                "innodbcluster02-2.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306": {
                    "address": "innodbcluster02-2.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306", 
                    "memberRole": "SECONDARY", 
                    "mode": "R/O", 
                    "readReplicas": {}, 
                    "replicationLagFromImmediateSource": "", 
                    "replicationLagFromOriginalSource": "", 
                    "role": "HA", 
                    "status": "ONLINE", 
                    "version": "9.4.0"
                }
            }, 
            "transactionSet": "4f01723b-737b-11f0-85fc-b2b76c25a31a:1-14,5feb4d6b-737b-11f0-bc19-b2b76c25a31a:1-480:1000290-1000331", 
            "transactionSetConsistencyStatus": "OK", 
            "transactionSetErrantGtidSet": "", 
            "transactionSetMissingGtidSet": ""
        }
    }, 
    "domainName": "innodbcluster01", 
    "globalPrimaryInstance": "innodbcluster01-1.innodbcluster01-instances.innodb01.svc.k8sdemos2.local:3306", 
    "metadataServer": "innodbcluster01-1.innodbcluster01-instances.innodb01.svc.k8sdemos2.local:3306", 
    "primaryCluster": "innodbcluster01", 
    "status": "HEALTHY", 
    "statusText": "All Clusters available."
}
```

Check the new Replica Cluster:
```javascript
JS > repc.status();
```

Example output:
```json
{
    "clusterName": "innodbcluster02", 
    "clusterRole": "REPLICA", 
    "clusterSetReplicationStatus": "OK", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306", 
        "ssl": "REQUIRED", 
        "status": "OK", 
        "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.", 
        "topology": {
            "innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306": {
                "address": "innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306", 
                "memberRole": "PRIMARY", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "replicationLagFromImmediateSource": "", 
                "replicationLagFromOriginalSource": "", 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "9.4.0"
            }, 
            "innodbcluster02-1.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306": {
                "address": "innodbcluster02-1.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306", 
                "memberRole": "SECONDARY", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "replicationLagFromImmediateSource": "", 
                "replicationLagFromOriginalSource": "", 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "9.4.0"
            }, 
            "innodbcluster02-2.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306": {
                "address": "innodbcluster02-2.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306", 
                "memberRole": "SECONDARY", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "replicationLagFromImmediateSource": "", 
                "replicationLagFromOriginalSource": "", 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "9.4.0"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "domainName": "innodbcluster01", 
    "groupInformationSourceMember": "innodbcluster02-0.innodbcluster02-instances.innodb02.svc.k8sdemos3.local:3306", 
    "metadataServer": "innodbcluster01-1.innodbcluster01-instances.innodb01.svc.k8sdemos2.local:3306"
}
```

Resume the MySQL Operator on the DR site:
```bash
kubectl -n mysql-operator scale deploy/mysql-operator --replicas=1 --context=$CLUSTER2
```

Re-deploy MySQL Router on CLUSTER2 cluster to adopt changes on cluster metadata
```bash
kubectl --context $CLUSTER2 -n innodb02 rollout restart deployment innodbcluster02-router
```


⚠️ **Note:** :
>Pausing the Operator ensures your manual topology changes are not overwritten during the conversion. Once the Replica Cluster is successfully created and added to the ClusterSet, the Operator can safely resume normal reconciliation and management.

---

## Conclusion

Congratulations 👏 ! You have successfully deployed a MySQL InnoDB ClusterSet across multiple Kubernetes clusters.
With Cilium in place, you can now take advantage of its ability to seamlessly connect two or more Kubernetes clusters, making them operate as if they were part of the same Kubernetes environment.
This allows an InnoDB Cluster running in one Kubernetes cluster to discover and connect to another InnoDB Cluster in a remote Kubernetes cluster.
As a result, you can configure an InnoDB ClusterSet to replicate data between two or more InnoDB Clusters across multiple Kubernetes clusters.

We are looking forward to the release of Cilium 1.19.0, as it is expected to simplify DNS configuration in multi-cluster setups even further.

---


## References

[Cilium Cluster Mesh](https://docs.cilium.io/en/stable/network/clustermesh/clustermesh/)
[Cilium Multi-Cluster Services API ](https://docs.cilium.io/en/latest/network/clustermesh/mcsapi/)
[Deploying InnoDB ClusterSet](https://dev.mysql.com/doc/mysql-shell/9.4/en/innodb-clusterset-deploy.html)
[Deploying MySQL InnoDB ClusterSet Across Kubernetes Clusters Using Cilium](https://blogs.oracle.com/mysql/post/deploying-mysql-innodb-clusterset-across-kubernetes-clusters-using-cilium)
[Cilium installation values](https://github.com/cilium/cilium/blob/main/install/kubernetes/cilium/values.yaml)

