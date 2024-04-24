## 環境

- Ubuntu 22.04
- kubernetes v1.30.0

```bash
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"30", GitVersion:"v1.30.0", GitCommit:"7c48c2bd72b9bf5c44d21d7338cc7bea77d0ad2a", GitTreeState:"clean", BuildDate:"2024-04-17T17:34:08Z", GoVersion:"go1.22.2", Compiler:"gc", Platform:"linux/amd64"}
```

## 本題

flannel をインストールした際に POD が正常に起動しなかった。

```
$ kubectl get po -A
NAMESPACE      NAME                           READY   STATUS              RESTARTS      AGE
kube-flannel   kube-flannel-ds-njbp7          0/1     CrashLoopBackOff    3 (16s ago)   65s
kube-system    coredns-7db6d8ff4d-5xtdq       0/1     ContainerCreating   0             16m
kube-system    coredns-7db6d8ff4d-8xqhj       0/1     ContainerCreating   0             16m
kube-system    etcd-lime                      1/1     Running             1             16m
kube-system    kube-apiserver-lime            1/1     Running             1             16m
kube-system    kube-controller-manager-lime   1/1     Running             1             16m
kube-system    kube-proxy-fgskc               1/1     Running             0             16m
kube-system    kube-scheduler-lime            1/1     Running             1             16m
```

ログを見てみるとこんなエラーが

```
Error registering network: failed to acquire lease: subnet "10.244.0.0/16" specified in the flannel net config doesn't contain "192.168.0.0/24" PodCIDR of the "lime" node
```

確かに node の PodCIDR が`192.168.0.0./24`になっていた。

```bash
$ kubectl describe node lime
Name:               lime
Roles:              control-plane
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=lime
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=
                    node.kubernetes.io/exclude-from-external-load-balancers=
Annotations:        flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"8a:db:a8:98:d0:22"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 133.18.202.53
                    kubeadm.alpha.kubernetes.io/cri-socket: unix:///run/containerd/containerd.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Wed, 24 Apr 2024 21:23:26 +0900
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  lime
  AcquireTime:     <unset>
  RenewTime:       Wed, 24 Apr 2024 21:45:27 +0900
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Wed, 24 Apr 2024 21:41:41 +0900   Wed, 24 Apr 2024 21:23:24 +0900   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Wed, 24 Apr 2024 21:41:41 +0900   Wed, 24 Apr 2024 21:23:24 +0900   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Wed, 24 Apr 2024 21:41:41 +0900   Wed, 24 Apr 2024 21:23:24 +0900   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Wed, 24 Apr 2024 21:41:41 +0900   Wed, 24 Apr 2024 21:36:15 +0900   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  10.150.11.1
  Hostname:    lime
Capacity:
  cpu:                2
  ephemeral-storage:  25625852Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             2005932Ki
  pods:               110
Allocatable:
  cpu:                2
  ephemeral-storage:  23616785165
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             1903532Ki
  pods:               110
System Info:
  Machine ID:                 55bf6f37f4724e4aa9d8c1e2e0dfdc7e
  System UUID:                55bf6f37-f472-4e4a-a9d8-c1e2e0dfdc7e
  Boot ID:                    1197fc44-f017-4a7e-b35e-719a87801b27
  Kernel Version:             6.5.0-28-generic
  OS Image:                   Ubuntu 22.04.4 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.6.31
  Kubelet Version:            v1.30.0
  Kube-Proxy Version:         v1.30.0
PodCIDR:                      192.168.0.0/24
PodCIDRs:                     192.168.0.0/24
Non-terminated Pods:          (9 in total)
  Namespace                   Name                            CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                            ------------  ----------  ---------------  -------------  ---
  kube-flannel                kube-flannel-ds-njbp7           100m (5%)     0 (0%)      50Mi (2%)        0 (0%)         6m55s
  kube-system                 coredns-7db6d8ff4d-5xtdq        100m (5%)     0 (0%)      70Mi (3%)        170Mi (9%)     21m
  kube-system                 coredns-7db6d8ff4d-8xqhj        100m (5%)     0 (0%)      70Mi (3%)        170Mi (9%)     21m
  kube-system                 etcd-lime                       100m (5%)     0 (0%)      100Mi (5%)       0 (0%)         22m
  kube-system                 kube-apiserver-lime             250m (12%)    0 (0%)      0 (0%)           0 (0%)         22m
  kube-system                 kube-controller-manager-lime    200m (10%)    0 (0%)      0 (0%)           0 (0%)         22m
  kube-system                 kube-proxy-fgskc                0 (0%)        0 (0%)      0 (0%)           0 (0%)         21m
  kube-system                 kube-scheduler-lime             100m (5%)     0 (0%)      0 (0%)           0 (0%)         22m
  velero                      velero-upgrade-crds-b4kg7       0 (0%)        0 (0%)      0 (0%)           0 (0%)         16m
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                950m (47%)   0 (0%)
  memory             290Mi (15%)  340Mi (18%)
  ephemeral-storage  0 (0%)       0 (0%)
  hugepages-1Gi      0 (0%)       0 (0%)
  hugepages-2Mi      0 (0%)       0 (0%)
Events:
  Type     Reason                   Age    From             Message
  ----     ------                   ----   ----             -------
  Normal   Starting                 21m    kube-proxy
  Normal   Starting                 22m    kubelet          Starting kubelet.
  Warning  InvalidDiskCapacity      22m    kubelet          invalid capacity 0 on image filesystem
  Normal   NodeHasSufficientMemory  22m    kubelet          Node lime status is now: NodeHasSufficientMemory
  Normal   NodeHasNoDiskPressure    22m    kubelet          Node lime status is now: NodeHasNoDiskPressure
  Normal   NodeHasSufficientPID     22m    kubelet          Node lime status is now: NodeHasSufficientPID
  Normal   NodeAllocatableEnforced  22m    kubelet          Updated Node Allocatable limit across pods
  Normal   RegisteredNode           21m    node-controller  Node lime event: Registered Node lime in Controller
  Normal   NodeReady                9m18s  kubelet          Node lime status is now: NodeReady
```

こうなる心当たりはあって、`kubeadm-config.yaml`の`podSubnet`を`192.168.0.0/16`にしていた。

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  strictARP: true # MetalLBに必要
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  podSubnet: "192.168.0.0/16"
```

そこで、 podSubnet を`10.244.0.0/16`にしたら無事起動しました！
