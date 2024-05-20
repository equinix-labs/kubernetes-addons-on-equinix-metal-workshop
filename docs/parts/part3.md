<!-- See https://squidfunk.github.io/mkdocs-material/reference/ -->
# Part 3: Verifying the Deployment of Kubernetes addon

We have a Kubernetes cluster and the CPEM addon up and running. Let's verify that the CPEM addon has been successfully deployed into the kubernetes cluster.

The steps below will guide you to verify the deplyment of the CPEM addon.

## Steps

### 1. Identify the pod created for CPEM

Let's verify that the `daemon` named `cloud-provider-equinix-metal` is running on the cluster pod:

```shell
$ kubectl get daemonset --namespace kube-system cloud-provider-equinix-metal
NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
cloud-provider-equinix-metal   1         1         1       1            1           <none>          25m
```

Next, we need to find which pod the daemon is running on in the cluster:

```shell
$ kubectl get pods --namespace kube-system -l app=cloud-provider-equinix-metal
NAME                                 READY   STATUS    RESTARTS   AGE
cloud-provider-equinix-metal-zkjqb   1/1     Running   0          30m
```

### 2. Verify the Deployment

Now that we now which pod the addon is deployed in, let check the logs of that pod:

```shell
$ kubectl logs --namespace kube-system cloud-provider-equinix-metal-zkjqb
I0603 18:33:02.387385       1 serving.go:348] Generated self-signed cert in-memory
W0603 18:33:02.387421       1 client_config.go:618] Neither --kubeconfig nor --master was specified.  Using the inClusterConfig.  This might not work.
I0603 18:33:02.511766       1 config.go:210] authToken: '<masked>'
I0603 18:33:02.511770       1 config.go:210] projectID: '49d38495-3601-4cd1-9de3-16a060ead426'
I0603 18:33:02.511772       1 config.go:210] load balancer config: 'kube-vip://'
I0603 18:33:02.511773       1 config.go:210] metro: ''
I0603 18:33:02.511774       1 config.go:210] facility: ''
I0603 18:33:02.511775       1 config.go:210] local ASN: '65000'
I0603 18:33:02.511776       1 config.go:210] Elastic IP Tag: ''
I0603 18:33:02.511777       1 config.go:210] API Server Port: '0'
I0603 18:33:02.511778       1 config.go:210] BGP Node Selector: ''
I0603 18:33:02.511795       1 config.go:210] Load Balancer ID: ''
I0603 18:33:02.511808       1 controllermanager.go:152] Version: v3.8.0
I0603 18:33:02.512617       1 secure_serving.go:213] Serving securely on [::]:10258
I0603 18:33:02.512669       1 tlsconfig.go:240] "Starting DynamicServingCertificateController"
I0603 18:33:02.512738       1 leaderelection.go:248] attempting to acquire leader lease kube-system/cloud-controller-manager...
I0603 18:33:02.519099       1 leaderelection.go:258] successfully acquired lease kube-system/cloud-controller-manager
I0603 18:33:02.519155       1 event.go:294] "Event occurred" object="kube-system/cloud-controller-manager" fieldPath="" kind="Lease" apiVersion="coordination.k8s.io/v1" type="Normal" reason="LeaderElection" message="k8s-cluster1-pool1-cp-1_2fe04540-8fd1-46cd-bffc-d6f1a53f76e0 became leader"
I0603 18:33:02.526317       1 eip_controlplane_reconciliation.go:71] EIP Tag is not configured skipping control plane endpoint management.
I0603 18:33:02.526333       1 controlplane_load_balancer_manager.go:41] Load balancer ID is not configured, skipping control plane load balancer management
I0603 18:33:02.735733       1 loadbalancers.go:93] loadbalancer implementation enabled: kube-vip
I0603 18:33:02.735766       1 cloud.go:104] Initialize of cloud provider complete
I0603 18:33:02.736310       1 controllermanager.go:311] Started "service"
I0603 18:33:02.736496       1 controller.go:227] Starting service controller
I0603 18:33:02.736554       1 shared_informer.go:270] Waiting for caches to sync for service
I0603 18:33:02.736670       1 controllermanager.go:311] Started "cloud-node"
I0603 18:33:02.736741       1 node_controller.go:157] Sending events to api server.
I0603 18:33:02.736879       1 node_controller.go:166] Waiting for informer caches to sync
I0603 18:33:02.736978       1 controllermanager.go:311] Started "cloud-node-lifecycle"
I0603 18:33:02.737066       1 node_lifecycle_controller.go:113] Sending events to api server
I0603 18:33:02.837272       1 shared_informer.go:277] Caches are synced for service
I0603 18:33:02.837440       1 node_controller.go:415] Initializing node k8s-cluster1-pool1-cp-1 with cloud provider
I0603 18:33:03.471796       1 node_controller.go:484] Successfully initialized node k8s-cluster1-pool1-cp-1 with cloud provider
I0603 18:33:03.471957       1 event.go:294] "Event occurred" object="k8s-cluster1-pool1-cp-1" fieldPath="" kind="Node" apiVersion="v1" type="Normal" reason="Synced" message="Node synced successfully"
I0603 18:33:04.239497       1 node_controller.go:415] Initializing node k8s-cluster1-pool1-worker-1 with cloud provider
I0603 18:33:04.862896       1 node_controller.go:484] Successfully initialized node k8s-cluster1-pool1-worker-1 with cloud provider
I0603 18:33:04.862992       1 event.go:294] "Event occurred" object="k8s-cluster1-pool1-worker-1" fieldPath="" kind="Node" apiVersion="v1" type="Normal" reason="Synced" message="Node synced successfully"
```

Take a look at the logs and note `I0603 18:33:02.519099       1 leaderelection.go:258] successfully acquired lease kube-system/cloud-controller-manager`. This means that the cloud controller manager operator pod is working as expected and has acquired the lease to run as a controller manager in the cluster.

### 3. Test the Add-on

We have verified that the CPEM addon has been successfully deployed into the kubernetes cluster. Let's test it!

We are going to check provider ID associated with the control plane node(s) deployed in the cluster. The provider ID is the unique ID of the instance that an external provider (i.e. cloud provider) can use to identify a specific node. This is a field that is used whenever kubernetes components need to identify one of our nodes to our API.

Let's find the control plane node name:

```shell
$ kubectl get nodes
NAME                          STATUS     ROLES           AGE    VERSION
k8s-cluster1-pool1-cp-1       NotReady   control-plane   3d1h   v1.27.5
k8s-cluster1-pool1-worker-1   NotReady   <none>          3d1h   v1.27.5
```

Now let's find the provider ID associated with the `k8s-cluster1-pool1-cp-1` node:

```shell
$ kubectl describe nodes k8s-cluster1-pool1-cp-1
Name:               k8s-cluster1-pool1-cp-1
Roles:              control-plane
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=m3.small.x86
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=sv
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=k8s-cluster1-pool1-cp-1
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=
                    node.kubernetes.io/exclude-from-external-load-balancers=
                    node.kubernetes.io/instance-type=m3.small.x86
                    topology.kubernetes.io/region=sv
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: unix:///var/run/containerd/containerd.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Mon, 03 Jun 2024 11:32:44 -0700
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
                    node.kubernetes.io/not-ready:NoSchedule
Unschedulable:      false
Lease:
  HolderIdentity:  k8s-cluster1-pool1-cp-1
  AcquireTime:     <unset>
  RenewTime:       Thu, 06 Jun 2024 15:25:13 -0700
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Thu, 06 Jun 2024 15:24:19 -0700   Mon, 03 Jun 2024 11:32:43 -0700   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Thu, 06 Jun 2024 15:24:19 -0700   Mon, 03 Jun 2024 11:32:43 -0700   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Thu, 06 Jun 2024 15:24:19 -0700   Mon, 03 Jun 2024 11:32:43 -0700   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            False   Thu, 06 Jun 2024 15:24:19 -0700   Mon, 03 Jun 2024 11:32:43 -0700   KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized
Addresses:
  Hostname:    k8s-cluster1-pool1-cp-1
  ExternalIP:  147.75.71.245
  InternalIP:  10.67.129.133
Capacity:
  cpu:                16
  ephemeral-storage:  457887756Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             65657932Ki
  pods:               110
Allocatable:
  cpu:                16
  ephemeral-storage:  421989355231
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             65555532Ki
  pods:               110
System Info:
  Machine ID:                 73cf12b09b6340be83c4a2b7a8ebfb67
  System UUID:                1f145e00-c7d6-11ec-8000-3cecefcde6c2
  Boot ID:                    fa91db61-0e64-40e7-a298-2cf13ddc83ea
  Kernel Version:             5.15.0-107-generic
  OS Image:                   Ubuntu 20.04.6 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.6.32
  Kubelet Version:            v1.27.5
  Kube-Proxy Version:         v1.27.5
PodCIDR:                      10.244.0.0/24
PodCIDRs:                     10.244.0.0/24
ProviderID:                   equinixmetal://de9fbfec-c878-4627-be5b-4c368be25ac7
Non-terminated Pods:          (7 in total)
  Namespace                   Name                                               CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                               ------------  ----------  ---------------  -------------  ---
  kube-system                 cloud-provider-equinix-metal-zkjqb                 100m (0%)     0 (0%)      50Mi (0%)        0 (0%)         3d3h
  kube-system                 etcd-k8s-cluster1-pool1-cp-1                       100m (0%)     0 (0%)      100Mi (0%)       0 (0%)         3d3h
  kube-system                 kube-apiserver-k8s-cluster1-pool1-cp-1             250m (1%)     0 (0%)      0 (0%)           0 (0%)         3d3h
  kube-system                 kube-controller-manager-k8s-cluster1-pool1-cp-1    200m (1%)     0 (0%)      0 (0%)           0 (0%)         3d3h
  kube-system                 kube-proxy-lkhs9                                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         3d3h
  kube-system                 kube-scheduler-k8s-cluster1-pool1-cp-1             100m (0%)     0 (0%)      0 (0%)           0 (0%)         3d3h
  kube-system                 kube-vip-k8s-cluster1-pool1-cp-1                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         3d3h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                750m (4%)   0 (0%)
  memory             150Mi (0%)  0 (0%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
Events:              <none>
```

Note the attribute value associated with the `ProviderID:` key: `equinixmetal://de9fbfec-c878-4627-be5b-4c368be25ac7`. This value has been issued by CPEM, thus we can conclude that the CPEM addon is working as expected.

## Discussion

Let's take a few minutes to discuss what we did. Here are some questions to start the discussion.

* How is a cloud controller manager deployed in a Kubernetes cluster?
* How do we verify that the cloud controller manager has been deployed correctly?