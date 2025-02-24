## Introduction

GPU time-slicing enables workloads that are scheduled on oversubscribed GPUs to interleave with one another

## To deploy GPU operator

The deployment assumes that there is a functioning kubernetes cluster

1. `helm repo add nvidia https://helm.ngc.nvidia.com/nvidia 
    && helm repo update`
2. `helm install --wait --generate-name 
    -n gpu-operator --create-namespace 
    nvidia/gpu-operator 
    --version=v24.9.2 `

This will deploy the GPU opertor in all nodes.

The pods deployed by the operator are, when the nodes have the GPU driver installed 

```
NAME                                                          READY   STATUS      RESTARTS   AGE
gpu-feature-discovery-4gxnf                                   2/2     Running     0          98m
gpu-feature-discovery-df2fz                                   2/2     Running     0          98m
gpu-operator-86765669fc-9cs22                                 1/1     Running     0          4h2m
gpu-operator-node-feature-discovery-gc-555ccf7687-764rh       1/1     Running     0          4h2m
gpu-operator-node-feature-discovery-master-68d694564d-fbrnj   1/1     Running     0          4h2m
gpu-operator-node-feature-discovery-worker-89wfs              1/1     Running     0          4h2m
gpu-operator-node-feature-discovery-worker-n7bsh              1/1     Running     0          4h2m
nvidia-container-toolkit-daemonset-6m526                      1/1     Running     0          4h2m
nvidia-container-toolkit-daemonset-bc8zl                      1/1     Running     0          4h2m
nvidia-cuda-validator-2wsz7                                   0/1     Completed   0          4h2m
nvidia-cuda-validator-7sf88                                   0/1     Completed   0          4h2m
nvidia-dcgm-exporter-4qsv2                                    1/1     Running     0          4h2m
nvidia-dcgm-exporter-cskjj                                    1/1     Running     0          4h2m
nvidia-device-plugin-daemonset-sd2kt                          2/2     Running     0          98m
nvidia-device-plugin-daemonset-xvckj                          2/2     Running     0          98m
nvidia-operator-validator-bmmlr                               1/1     Running     0          4h2m
nvidia-operator-validator-gvk6z                               1/1     Running     0          4h2m
```

## To time-slice the GPUs

1. Create a `ConfigMap` in the `gpu-operator` namespace
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config-fine
data:
  tesla-t4: |-
    version: v1
    flags:
      migStrategy: none
    sharing:
      timeSlicing:
        resources:
        - name: nvidia.com/gpu
          replicas: 4 # Can be desired number of replicas
  Tesla-V100-PCIE-16GB: |-
    version: v1
    flags:
      migStrategy: none
    sharing:
      timeSlicing:
        resources:
        - name: nvidia.com/gpu
          replicas: 4 # Can be desired number of replicas
```
2. Label all the GPU nodes with their respective GPU name by executing \
   `kubectl label node --selector=nvidia.com/gpu.product=Tesla-T4  nvidia.com/device-plugin.config=tesla-t4` 
   `kubectl label node --selector=nvidia.com/gpu.product=Tesla-V100-PCIE-16GB   nvidia.com/device-plugin.config=Tesla-V100-PCIE-16GB`\
3. Patch the cluster policy with `time-slicing-config-fine` configmap 
  `kubectl patch clusterpolicies.nvidia.com/cluster-policy     -n gpu-operator --type merge     -p '{"spec": {"devicePlugin": {"config": {"name": "time-slicing-config-fine"}}}}'` \
4. Restart the GPU operator daemonset
   `kubectl rollout restart -n gpu-operator daemonset/nvidia-device-plugin-daemonset` 
## Before time slicing
```
Name:               node1
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    feature.node.kubernetes.io/cpu-cpuid.ADX=true
                    feature.node.kubernetes.io/cpu-cpuid.AESNI=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX2=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512BW=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512CD=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512DQ=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512F=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VL=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VNNI=true
                    feature.node.kubernetes.io/cpu-cpuid.CMPXCHG8=true
                    feature.node.kubernetes.io/cpu-cpuid.FLUSH_L1D=true
                    feature.node.kubernetes.io/cpu-cpuid.FMA3=true
                    feature.node.kubernetes.io/cpu-cpuid.FXSR=true
                    feature.node.kubernetes.io/cpu-cpuid.FXSROPT=true
                    feature.node.kubernetes.io/cpu-cpuid.IA32_ARCH_CAP=true
                    feature.node.kubernetes.io/cpu-cpuid.IBPB=true
                    feature.node.kubernetes.io/cpu-cpuid.LAHF=true
                    feature.node.kubernetes.io/cpu-cpuid.MD_CLEAR=true
                    feature.node.kubernetes.io/cpu-cpuid.MOVBE=true
                    feature.node.kubernetes.io/cpu-cpuid.MPX=true
                    feature.node.kubernetes.io/cpu-cpuid.OSXSAVE=true
                    feature.node.kubernetes.io/cpu-cpuid.SPEC_CTRL_SSBD=true
                    feature.node.kubernetes.io/cpu-cpuid.STIBP=true
                    feature.node.kubernetes.io/cpu-cpuid.SYSCALL=true
                    feature.node.kubernetes.io/cpu-cpuid.SYSEE=true
                    feature.node.kubernetes.io/cpu-cpuid.VMX=true
                    feature.node.kubernetes.io/cpu-cpuid.X87=true
                    feature.node.kubernetes.io/cpu-cpuid.XGETBV1=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVE=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVEC=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVEOPT=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVES=true
                    feature.node.kubernetes.io/cpu-cstate.enabled=true
                    feature.node.kubernetes.io/cpu-hardware_multithreading=true
                    feature.node.kubernetes.io/cpu-model.family=6
                    feature.node.kubernetes.io/cpu-model.id=85
                    feature.node.kubernetes.io/cpu-model.vendor_id=Intel
                    feature.node.kubernetes.io/cpu-rdt.RDTCMT=true
                    feature.node.kubernetes.io/cpu-rdt.RDTL3CA=true
                    feature.node.kubernetes.io/cpu-rdt.RDTMBA=true
                    feature.node.kubernetes.io/cpu-rdt.RDTMBM=true
                    feature.node.kubernetes.io/cpu-rdt.RDTMON=true
                    feature.node.kubernetes.io/kernel-config.NO_HZ=true
                    feature.node.kubernetes.io/kernel-config.NO_HZ_FULL=true
                    feature.node.kubernetes.io/kernel-version.full=6.8.0-52-generic
                    feature.node.kubernetes.io/kernel-version.major=6
                    feature.node.kubernetes.io/kernel-version.minor=8
                    feature.node.kubernetes.io/kernel-version.revision=0
                    feature.node.kubernetes.io/memory-numa=true
                    feature.node.kubernetes.io/network-sriov.capable=true
                    feature.node.kubernetes.io/pci-10de.present=true
                    feature.node.kubernetes.io/pci-10de.sriov.capable=true
                    feature.node.kubernetes.io/pci-1a03.present=true
                    feature.node.kubernetes.io/pci-8086.present=true
                    feature.node.kubernetes.io/pci-8086.sriov.capable=true
                    feature.node.kubernetes.io/storage-nonrotationaldisk=true
                    feature.node.kubernetes.io/system-os_release.ID=ubuntu
                    feature.node.kubernetes.io/system-os_release.VERSION_ID=22.04
                    feature.node.kubernetes.io/system-os_release.VERSION_ID.major=22
                    feature.node.kubernetes.io/system-os_release.VERSION_ID.minor=04
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=node1
                    kubernetes.io/os=linux
                    microk8s.io/cluster=true
                    node.kubernetes.io/microk8s-controlplane=microk8s-controlplane
                    nvidia.com/cuda.driver.major=535
                    nvidia.com/cuda.driver.minor=183
                    nvidia.com/cuda.driver.rev=01
                    nvidia.com/cuda.runtime.major=12
                    nvidia.com/cuda.runtime.minor=2
                    nvidia.com/gfd.timestamp=1738200200
                    nvidia.com/gpu.compute.major=7
                    nvidia.com/gpu.compute.minor=5
                    nvidia.com/gpu.count=1
                    nvidia.com/gpu.deploy.container-toolkit=true
                    nvidia.com/gpu.deploy.dcgm=true
                    nvidia.com/gpu.deploy.dcgm-exporter=true
                    nvidia.com/gpu.deploy.device-plugin=true
                    nvidia.com/gpu.deploy.driver=true
                    nvidia.com/gpu.deploy.gpu-feature-discovery=true
                    nvidia.com/gpu.deploy.node-status-exporter=true
                    nvidia.com/gpu.deploy.operator-validator=true
                    nvidia.com/gpu.family=turing
                    nvidia.com/gpu.machine=QuantaPlex-T42SP-2U-LBG-4-21S5SMA0110
                    nvidia.com/gpu.memory=15360
                    nvidia.com/gpu.present=true
                    nvidia.com/gpu.product=Tesla-T4
                    nvidia.com/gpu.replicas=1
                    nvidia.com/mig.capable=false
                    nvidia.com/mig.strategy=single
Annotations:        nfd.node.kubernetes.io/feature-labels:
                      cpu-cpuid.ADX,cpu-cpuid.AESNI,cpu-cpuid.AVX,cpu-cpuid.AVX2,cpu-cpuid.AVX512BW,cpu-cpuid.AVX512CD,cpu-cpuid.AVX512DQ,cpu-cpuid.AVX512F,cpu-...
                    nfd.node.kubernetes.io/worker.version: v0.14.2
                    node.alpha.kubernetes.io/ttl: 0
                    nvidia.com/gpu-driver-upgrade-enabled: true
                    projectcalico.org/IPv4Address: xxx.xxx.xxx.xxx/24
                    projectcalico.org/IPv4VXLANTunnelAddr: 10.1.166.128
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Thu, 30 Jan 2025 12:11:05 +1100
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  node1
  AcquireTime:     <unset>
  RenewTime:       Thu, 30 Jan 2025 14:44:26 +1100
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Thu, 30 Jan 2025 12:12:35 +1100   Thu, 30 Jan 2025 12:12:35 +1100   CalicoIsUp                   Calico is running on this node
  MemoryPressure       False   Thu, 30 Jan 2025 14:44:22 +1100   Thu, 30 Jan 2025 12:11:05 +1100   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Thu, 30 Jan 2025 14:44:22 +1100   Thu, 30 Jan 2025 12:11:05 +1100   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Thu, 30 Jan 2025 14:44:22 +1100   Thu, 30 Jan 2025 12:11:05 +1100   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Thu, 30 Jan 2025 14:44:22 +1100   Thu, 30 Jan 2025 12:11:05 +1100   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  xxx.xxx.xxx.xxx
  Hostname:    node1
Capacity:
  cpu:                40
  ephemeral-storage:  211990028Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             131630256Ki
  nvidia.com/gpu:     1
  pods:               110
Allocatable:
  cpu:                40
  ephemeral-storage:  210941452Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             131527856Ki
  nvidia.com/gpu:     0
  pods:               110
System Info:
  Machine ID:                 7fe0ef2102af4f19b076ea0d847a632a
  System UUID:                e89d66d4-adc6-11eb-affd-c01850406968
  Boot ID:                    b37d6531-8976-41ca-a9a5-69d02039c86b
  Kernel Version:             6.8.0-52-generic
  OS Image:                   Ubuntu 22.04.5 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.6.28
  Kubelet Version:            v1.31.5
  Kube-Proxy Version:         v1.31.5
Non-terminated Pods:          (14 in total)
  Namespace                   Name                                                               CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                                               ------------  ----------  ---------------  -------------  ---
  default                     dcgm-exporter-tlmt8                                                0 (0%)        0 (0%)      0 (0%)           0 (0%)         121m
  gpu-operator-resources      gpu-feature-discovery-7s4w2                                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         141m
  gpu-operator-resources      gpu-operator-node-feature-discovery-worker-n7bsh                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         141m
  gpu-operator-resources      nvidia-container-toolkit-daemonset-6m526                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         141m
  gpu-operator-resources      nvidia-dcgm-exporter-cskjj                                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         141m
  gpu-operator-resources      nvidia-device-plugin-daemonset-l9v5l                               0 (0%)        0 (0%)      0 (0%)           0 (0%)         6s
  gpu-operator-resources      nvidia-operator-validator-bmmlr                                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         141m
  kube-system                 calico-kube-controllers-759cd8b574-dflk7                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         151m
  kube-system                 calico-node-pwscp                                                  250m (0%)     0 (0%)      0 (0%)           0 (0%)         151m
  kube-system                 dashboard-metrics-scraper-6b96ff7878-t78jf                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         146m
  kube-system                 hostpath-provisioner-5fbc49d86c-9x7qf                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         110m
  kube-system                 kubernetes-dashboard-7d869bcd96-vn29g                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         146m
  kube-system                 metrics-server-df8dbf7f5-mtkk7                                     100m (0%)     0 (0%)      200Mi (0%)       0 (0%)         146m
  prometheus                  kube-prometheus-stack-1738204901-prometheus-node-exporter-qqjkr    0 (0%)        0 (0%)      0 (0%)           0 (0%)         62m
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                350m (0%)   0 (0%)
  memory             200Mi (0%)  0 (0%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
  nvidia.com/gpu     0           0
Events:              <none>


Name:               node2
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    feature.node.kubernetes.io/cpu-cpuid.ADX=true
                    feature.node.kubernetes.io/cpu-cpuid.AESNI=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX2=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512BF16=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512BITALG=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512BW=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512CD=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512DQ=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512F=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512IFMA=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VBMI=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VBMI2=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VL=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VNNI=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VPOPCNTDQ=true
                    feature.node.kubernetes.io/cpu-cpuid.CETSS=true
                    feature.node.kubernetes.io/cpu-cpuid.CLZERO=true
                    feature.node.kubernetes.io/cpu-cpuid.CMPXCHG8=true
                    feature.node.kubernetes.io/cpu-cpuid.CPBOOST=true
                    feature.node.kubernetes.io/cpu-cpuid.CPPC=true
                    feature.node.kubernetes.io/cpu-cpuid.EFER_LMSLE_UNS=true
                    feature.node.kubernetes.io/cpu-cpuid.FLUSH_L1D=true
                    feature.node.kubernetes.io/cpu-cpuid.FMA3=true
                    feature.node.kubernetes.io/cpu-cpuid.FP256=true
                    feature.node.kubernetes.io/cpu-cpuid.FSRM=true
                    feature.node.kubernetes.io/cpu-cpuid.FXSR=true
                    feature.node.kubernetes.io/cpu-cpuid.FXSROPT=true
                    feature.node.kubernetes.io/cpu-cpuid.GFNI=true
                    feature.node.kubernetes.io/cpu-cpuid.IBPB=true
                    feature.node.kubernetes.io/cpu-cpuid.IBRS=true
                    feature.node.kubernetes.io/cpu-cpuid.IBRS_PREFERRED=true
                    feature.node.kubernetes.io/cpu-cpuid.IBRS_PROVIDES_SMP=true
                    feature.node.kubernetes.io/cpu-cpuid.IBS=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSBRNTRGT=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSFETCHSAM=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSFFV=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSOPCNT=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSOPCNTEXT=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSOPSAM=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSRDWROPCNT=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSRIPINVALIDCHK=true
                    feature.node.kubernetes.io/cpu-cpuid.IBS_FETCH_CTLX=true
                    feature.node.kubernetes.io/cpu-cpuid.IBS_OPFUSE=true
                    feature.node.kubernetes.io/cpu-cpuid.IBS_PREVENTHOST=true
                    feature.node.kubernetes.io/cpu-cpuid.IBS_ZEN4=true
                    feature.node.kubernetes.io/cpu-cpuid.INT_WBINVD=true
                    feature.node.kubernetes.io/cpu-cpuid.INVLPGB=true
                    feature.node.kubernetes.io/cpu-cpuid.LAHF=true
                    feature.node.kubernetes.io/cpu-cpuid.LBRVIRT=true
                    feature.node.kubernetes.io/cpu-cpuid.MCAOVERFLOW=true
                    feature.node.kubernetes.io/cpu-cpuid.MOVBE=true
                    feature.node.kubernetes.io/cpu-cpuid.MOVU=true
                    feature.node.kubernetes.io/cpu-cpuid.MSRIRC=true
                    feature.node.kubernetes.io/cpu-cpuid.NRIPS=true
                    feature.node.kubernetes.io/cpu-cpuid.OSXSAVE=true
                    feature.node.kubernetes.io/cpu-cpuid.PPIN=true
                    feature.node.kubernetes.io/cpu-cpuid.PSFD=true
                    feature.node.kubernetes.io/cpu-cpuid.RDPRU=true
                    feature.node.kubernetes.io/cpu-cpuid.SEV=true
                    feature.node.kubernetes.io/cpu-cpuid.SEV_64BIT=true
                    feature.node.kubernetes.io/cpu-cpuid.SEV_ALTERNATIVE=true
                    feature.node.kubernetes.io/cpu-cpuid.SEV_DEBUGSWAP=true
                    feature.node.kubernetes.io/cpu-cpuid.SEV_ES=true
                    feature.node.kubernetes.io/cpu-cpuid.SEV_RESTRICTED=true
                    feature.node.kubernetes.io/cpu-cpuid.SEV_SNP=true
                    feature.node.kubernetes.io/cpu-cpuid.SHA=true
                    feature.node.kubernetes.io/cpu-cpuid.SME=true
                    feature.node.kubernetes.io/cpu-cpuid.SME_COHERENT=true
                    feature.node.kubernetes.io/cpu-cpuid.SPEC_CTRL_SSBD=true
                    feature.node.kubernetes.io/cpu-cpuid.SSE4A=true
                    feature.node.kubernetes.io/cpu-cpuid.STIBP=true
                    feature.node.kubernetes.io/cpu-cpuid.STIBP_ALWAYSON=true
                    feature.node.kubernetes.io/cpu-cpuid.SUCCOR=true
                    feature.node.kubernetes.io/cpu-cpuid.SVM=true
                    feature.node.kubernetes.io/cpu-cpuid.SVMDA=true
                    feature.node.kubernetes.io/cpu-cpuid.SVMFBASID=true
                    feature.node.kubernetes.io/cpu-cpuid.SVML=true
                    feature.node.kubernetes.io/cpu-cpuid.SVMNP=true
                    feature.node.kubernetes.io/cpu-cpuid.SVMPF=true
                    feature.node.kubernetes.io/cpu-cpuid.SVMPFT=true
                    feature.node.kubernetes.io/cpu-cpuid.SYSCALL=true
                    feature.node.kubernetes.io/cpu-cpuid.SYSEE=true
                    feature.node.kubernetes.io/cpu-cpuid.TLB_FLUSH_NESTED=true
                    feature.node.kubernetes.io/cpu-cpuid.TOPEXT=true
                    feature.node.kubernetes.io/cpu-cpuid.TSCRATEMSR=true
                    feature.node.kubernetes.io/cpu-cpuid.VAES=true
                    feature.node.kubernetes.io/cpu-cpuid.VMCBCLEAN=true
                    feature.node.kubernetes.io/cpu-cpuid.VMPL=true
                    feature.node.kubernetes.io/cpu-cpuid.VMSA_REGPROT=true
                    feature.node.kubernetes.io/cpu-cpuid.VPCLMULQDQ=true
                    feature.node.kubernetes.io/cpu-cpuid.VTE=true
                    feature.node.kubernetes.io/cpu-cpuid.WBNOINVD=true
                    feature.node.kubernetes.io/cpu-cpuid.X87=true
                    feature.node.kubernetes.io/cpu-cpuid.XGETBV1=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVE=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVEC=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVEOPT=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVES=true
                    feature.node.kubernetes.io/cpu-hardware_multithreading=true
                    feature.node.kubernetes.io/cpu-model.family=25
                    feature.node.kubernetes.io/cpu-model.id=17
                    feature.node.kubernetes.io/cpu-model.vendor_id=AMD
                    feature.node.kubernetes.io/cpu-rdt.RDTCMT=true
                    feature.node.kubernetes.io/cpu-rdt.RDTL3CA=true
                    feature.node.kubernetes.io/cpu-rdt.RDTMBM=true
                    feature.node.kubernetes.io/cpu-rdt.RDTMON=true
                    feature.node.kubernetes.io/kernel-config.NO_HZ=true
                    feature.node.kubernetes.io/kernel-config.NO_HZ_FULL=true
                    feature.node.kubernetes.io/kernel-version.full=6.8.0-52-generic
                    feature.node.kubernetes.io/kernel-version.major=6
                    feature.node.kubernetes.io/kernel-version.minor=8
                    feature.node.kubernetes.io/kernel-version.revision=0
                    feature.node.kubernetes.io/memory-numa=true
                    feature.node.kubernetes.io/network-sriov.capable=true
                    feature.node.kubernetes.io/pci-10de.present=true
                    feature.node.kubernetes.io/pci-1a03.present=true
                    feature.node.kubernetes.io/pci-8086.present=true
                    feature.node.kubernetes.io/pci-8086.sriov.capable=true
                    feature.node.kubernetes.io/storage-nonrotationaldisk=true
                    feature.node.kubernetes.io/system-os_release.ID=ubuntu
                    feature.node.kubernetes.io/system-os_release.VERSION_ID=22.04
                    feature.node.kubernetes.io/system-os_release.VERSION_ID.major=22
                    feature.node.kubernetes.io/system-os_release.VERSION_ID.minor=04
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=node2
                    kubernetes.io/os=linux
                    microk8s.io/cluster=true
                    node.kubernetes.io/microk8s-controlplane=microk8s-controlplane
                    nvidia.com/cuda.driver.major=535
                    nvidia.com/cuda.driver.minor=183
                    nvidia.com/cuda.driver.rev=01
                    nvidia.com/cuda.runtime.major=12
                    nvidia.com/cuda.runtime.minor=2
                    nvidia.com/gfd.timestamp=1738200202
                    nvidia.com/gpu.compute.major=7
                    nvidia.com/gpu.compute.minor=0
                    nvidia.com/gpu.count=1
                    nvidia.com/gpu.deploy.container-toolkit=true
                    nvidia.com/gpu.deploy.dcgm=true
                    nvidia.com/gpu.deploy.dcgm-exporter=true
                    nvidia.com/gpu.deploy.device-plugin=true
                    nvidia.com/gpu.deploy.driver=true
                    nvidia.com/gpu.deploy.gpu-feature-discovery=true
                    nvidia.com/gpu.deploy.node-status-exporter=true
                    nvidia.com/gpu.deploy.operator-validator=true
                    nvidia.com/gpu.family=volta
                    nvidia.com/gpu.machine=RS700A-E12-RS4U
                    nvidia.com/gpu.memory=16384
                    nvidia.com/gpu.present=true
                    nvidia.com/gpu.product=Tesla-V100-PCIE-16GB
                    nvidia.com/gpu.replicas=1
                    nvidia.com/mig.capable=false
                    nvidia.com/mig.strategy=single
Annotations:        nfd.node.kubernetes.io/feature-labels:
                      cpu-cpuid.ADX,cpu-cpuid.AESNI,cpu-cpuid.AVX,cpu-cpuid.AVX2,cpu-cpuid.AVX512BF16,cpu-cpuid.AVX512BITALG,cpu-cpuid.AVX512BW,cpu-cpuid.AVX512...
                    nfd.node.kubernetes.io/master.version: v0.14.2
                    nfd.node.kubernetes.io/worker.version: v0.14.2
                    node.alpha.kubernetes.io/ttl: 0
                    nvidia.com/gpu-driver-upgrade-enabled: true
                    projectcalico.org/IPv4Address: xxx.xxx.xxx.xxx/24
                    projectcalico.org/IPv4VXLANTunnelAddr: 10.1.104.0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Thu, 30 Jan 2025 12:20:49 +1100
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  node2
  AcquireTime:     <unset>
  RenewTime:       Thu, 30 Jan 2025 14:44:27 +1100
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Thu, 30 Jan 2025 12:20:59 +1100   Thu, 30 Jan 2025 12:20:59 +1100   CalicoIsUp                   Calico is running on this node
  MemoryPressure       False   Thu, 30 Jan 2025 14:42:15 +1100   Thu, 30 Jan 2025 12:20:49 +1100   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Thu, 30 Jan 2025 14:42:15 +1100   Thu, 30 Jan 2025 12:20:49 +1100   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Thu, 30 Jan 2025 14:42:15 +1100   Thu, 30 Jan 2025 12:20:49 +1100   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Thu, 30 Jan 2025 14:42:15 +1100   Thu, 30 Jan 2025 12:20:49 +1100   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  xxx.xxx.xxx.xxx
  Hostname:    node2
Capacity:
  cpu:                64
  ephemeral-storage:  459850824Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             131563384Ki
  nvidia.com/gpu:     1
  pods:               110
Allocatable:
  cpu:                64
  ephemeral-storage:  458802248Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             131460984Ki
  nvidia.com/gpu:     1
  pods:               110
System Info:
  Machine ID:                 2a008e07c71648caa5dded97337a55ae
  System UUID:                86069482-6cc6-78b8-44bf-a036bccb9707
  Boot ID:                    2359e07f-8a87-45ad-8250-d598147ea81f
  Kernel Version:             6.8.0-52-generic
  OS Image:                   Ubuntu 22.04.5 2024.10.16 LTS (Cubic 2024-10-16 14:33)
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.6.28
  Kubelet Version:            v1.31.5
  Kube-Proxy Version:         v1.31.5
Non-terminated Pods:          (18 in total)
  Namespace                   Name                                                               CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                                               ------------  ----------  ---------------  -------------  ---
  default                     dcgm-exporter-ld7bw                                                0 (0%)        0 (0%)      0 (0%)           0 (0%)         121m
  gpu-operator-resources      gpu-feature-discovery-d5gch                                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         141m
  gpu-operator-resources      gpu-operator-86765669fc-9cs22                                      200m (0%)     500m (0%)   100Mi (0%)       350Mi (0%)     141m
  gpu-operator-resources      gpu-operator-node-feature-discovery-gc-555ccf7687-764rh            0 (0%)        0 (0%)      0 (0%)           0 (0%)         141m
  gpu-operator-resources      gpu-operator-node-feature-discovery-master-68d694564d-fbrnj        0 (0%)        0 (0%)      0 (0%)           0 (0%)         141m
  gpu-operator-resources      gpu-operator-node-feature-discovery-worker-89wfs                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         141m
  gpu-operator-resources      nvidia-container-toolkit-daemonset-bc8zl                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         141m
  gpu-operator-resources      nvidia-dcgm-exporter-4qsv2                                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         141m
  gpu-operator-resources      nvidia-device-plugin-daemonset-8dcdn                               0 (0%)        0 (0%)      0 (0%)           0 (0%)         10s
  gpu-operator-resources      nvidia-operator-validator-gvk6z                                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         141m
  kube-system                 calico-node-6s54w                                                  250m (0%)     0 (0%)      0 (0%)           0 (0%)         143m
  kube-system                 coredns-7896dbf49-cr5m7                                            100m (0%)     0 (0%)      70Mi (0%)        170Mi (0%)     141m
  prometheus                  alertmanager-kube-prometheus-stack-1738-alertmanager-0             0 (0%)        0 (0%)      200Mi (0%)       0 (0%)         62m
  prometheus                  kube-prometheus-stack-1738-operator-7974c894dc-qbqdl               0 (0%)        0 (0%)      0 (0%)           0 (0%)         62m
  prometheus                  kube-prometheus-stack-1738204901-grafana-66d9864858-vjc77          0 (0%)        0 (0%)      0 (0%)           0 (0%)         62m
  prometheus                  kube-prometheus-stack-1738204901-kube-state-metrics-78789dk9rdc    0 (0%)        0 (0%)      0 (0%)           0 (0%)         62m
  prometheus                  kube-prometheus-stack-1738204901-prometheus-node-exporter-z98pc    0 (0%)        0 (0%)      0 (0%)           0 (0%)         62m
  prometheus                  prometheus-kube-prometheus-stack-1738-prometheus-0                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         62m
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                550m (0%)   500m (0%)
  memory             370Mi (0%)  520Mi (0%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
  nvidia.com/gpu     0           0
Events:              <none>
```
## After time slicing
```
Name:               node1
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    feature.node.kubernetes.io/cpu-cpuid.ADX=true
                    feature.node.kubernetes.io/cpu-cpuid.AESNI=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX2=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512BW=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512CD=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512DQ=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512F=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VL=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VNNI=true
                    feature.node.kubernetes.io/cpu-cpuid.CMPXCHG8=true
                    feature.node.kubernetes.io/cpu-cpuid.FLUSH_L1D=true
                    feature.node.kubernetes.io/cpu-cpuid.FMA3=true
                    feature.node.kubernetes.io/cpu-cpuid.FXSR=true
                    feature.node.kubernetes.io/cpu-cpuid.FXSROPT=true
                    feature.node.kubernetes.io/cpu-cpuid.IA32_ARCH_CAP=true
                    feature.node.kubernetes.io/cpu-cpuid.IBPB=true
                    feature.node.kubernetes.io/cpu-cpuid.LAHF=true
                    feature.node.kubernetes.io/cpu-cpuid.MD_CLEAR=true
                    feature.node.kubernetes.io/cpu-cpuid.MOVBE=true
                    feature.node.kubernetes.io/cpu-cpuid.MPX=true
                    feature.node.kubernetes.io/cpu-cpuid.OSXSAVE=true
                    feature.node.kubernetes.io/cpu-cpuid.SPEC_CTRL_SSBD=true
                    feature.node.kubernetes.io/cpu-cpuid.STIBP=true
                    feature.node.kubernetes.io/cpu-cpuid.SYSCALL=true
                    feature.node.kubernetes.io/cpu-cpuid.SYSEE=true
                    feature.node.kubernetes.io/cpu-cpuid.VMX=true
                    feature.node.kubernetes.io/cpu-cpuid.X87=true
                    feature.node.kubernetes.io/cpu-cpuid.XGETBV1=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVE=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVEC=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVEOPT=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVES=true
                    feature.node.kubernetes.io/cpu-cstate.enabled=true
                    feature.node.kubernetes.io/cpu-hardware_multithreading=true
                    feature.node.kubernetes.io/cpu-model.family=6
                    feature.node.kubernetes.io/cpu-model.id=85
                    feature.node.kubernetes.io/cpu-model.vendor_id=Intel
                    feature.node.kubernetes.io/cpu-rdt.RDTCMT=true
                    feature.node.kubernetes.io/cpu-rdt.RDTL3CA=true
                    feature.node.kubernetes.io/cpu-rdt.RDTMBA=true
                    feature.node.kubernetes.io/cpu-rdt.RDTMBM=true
                    feature.node.kubernetes.io/cpu-rdt.RDTMON=true
                    feature.node.kubernetes.io/kernel-config.NO_HZ=true
                    feature.node.kubernetes.io/kernel-config.NO_HZ_FULL=true
                    feature.node.kubernetes.io/kernel-version.full=6.8.0-52-generic
                    feature.node.kubernetes.io/kernel-version.major=6
                    feature.node.kubernetes.io/kernel-version.minor=8
                    feature.node.kubernetes.io/kernel-version.revision=0
                    feature.node.kubernetes.io/memory-numa=true
                    feature.node.kubernetes.io/network-sriov.capable=true
                    feature.node.kubernetes.io/pci-10de.present=true
                    feature.node.kubernetes.io/pci-10de.sriov.capable=true
                    feature.node.kubernetes.io/pci-1a03.present=true
                    feature.node.kubernetes.io/pci-8086.present=true
                    feature.node.kubernetes.io/pci-8086.sriov.capable=true
                    feature.node.kubernetes.io/storage-nonrotationaldisk=true
                    feature.node.kubernetes.io/system-os_release.ID=ubuntu
                    feature.node.kubernetes.io/system-os_release.VERSION_ID=22.04
                    feature.node.kubernetes.io/system-os_release.VERSION_ID.major=22
                    feature.node.kubernetes.io/system-os_release.VERSION_ID.minor=04
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=node1
                    kubernetes.io/os=linux
                    microk8s.io/cluster=true
                    node.kubernetes.io/microk8s-controlplane=microk8s-controlplane
                    nvidia.com/cuda.driver.major=535
                    nvidia.com/cuda.driver.minor=183
                    nvidia.com/cuda.driver.rev=01
                    nvidia.com/cuda.runtime.major=12
                    nvidia.com/cuda.runtime.minor=2
                    nvidia.com/device-plugin.config=tesla-t4
                    nvidia.com/gfd.timestamp=1738208845
                    nvidia.com/gpu.compute.major=7
                    nvidia.com/gpu.compute.minor=5
                    nvidia.com/gpu.count=1
                    nvidia.com/gpu.deploy.container-toolkit=true
                    nvidia.com/gpu.deploy.dcgm=true
                    nvidia.com/gpu.deploy.dcgm-exporter=true
                    nvidia.com/gpu.deploy.device-plugin=true
                    nvidia.com/gpu.deploy.driver=true
                    nvidia.com/gpu.deploy.gpu-feature-discovery=true
                    nvidia.com/gpu.deploy.node-status-exporter=true
                    nvidia.com/gpu.deploy.operator-validator=true
                    nvidia.com/gpu.family=turing
                    nvidia.com/gpu.machine=QuantaPlex-T42SP-2U-LBG-4-21S5SMA0110
                    nvidia.com/gpu.memory=15360
                    nvidia.com/gpu.present=true
                    nvidia.com/gpu.product=Tesla-T4-SHARED
                    nvidia.com/gpu.replicas=4
                    nvidia.com/mig.capable=false
                    nvidia.com/mig.strategy=single
Annotations:        nfd.node.kubernetes.io/feature-labels:
                      cpu-cpuid.ADX,cpu-cpuid.AESNI,cpu-cpuid.AVX,cpu-cpuid.AVX2,cpu-cpuid.AVX512BW,cpu-cpuid.AVX512CD,cpu-cpuid.AVX512DQ,cpu-cpuid.AVX512F,cpu-...
                    nfd.node.kubernetes.io/worker.version: v0.14.2
                    node.alpha.kubernetes.io/ttl: 0
                    nvidia.com/gpu-driver-upgrade-enabled: true
                    projectcalico.org/IPv4Address: xxx.xxx.xxx.xxx/24
                    projectcalico.org/IPv4VXLANTunnelAddr: 10.1.166.128
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Thu, 30 Jan 2025 12:11:05 +1100
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  node1
  AcquireTime:     <unset>
  RenewTime:       Thu, 30 Jan 2025 15:59:00 +1100
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Thu, 30 Jan 2025 12:12:35 +1100   Thu, 30 Jan 2025 12:12:35 +1100   CalicoIsUp                   Calico is running on this node
  MemoryPressure       False   Thu, 30 Jan 2025 15:54:09 +1100   Thu, 30 Jan 2025 12:11:05 +1100   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Thu, 30 Jan 2025 15:54:09 +1100   Thu, 30 Jan 2025 12:11:05 +1100   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Thu, 30 Jan 2025 15:54:09 +1100   Thu, 30 Jan 2025 12:11:05 +1100   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Thu, 30 Jan 2025 15:54:09 +1100   Thu, 30 Jan 2025 12:11:05 +1100   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  xxx.xxx.xxx.xxx
  Hostname:    node1
Capacity:
  cpu:                40
  ephemeral-storage:  211990028Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             131630256Ki
  nvidia.com/gpu:     4
  pods:               110
Allocatable:
  cpu:                40
  ephemeral-storage:  210941452Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             131527856Ki
  nvidia.com/gpu:     4
  pods:               110
System Info:
  Machine ID:                 7fe0ef2102af4f19b076ea0d847a632a
  System UUID:                e89d66d4-adc6-11eb-affd-c01850406968
  Boot ID:                    b37d6531-8976-41ca-a9a5-69d02039c86b
  Kernel Version:             6.8.0-52-generic
  OS Image:                   Ubuntu 22.04.5 LTS
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.6.28
  Kubelet Version:            v1.31.5
  Kube-Proxy Version:         v1.31.5
Non-terminated Pods:          (15 in total)
  Namespace                   Name                                                               CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                                               ------------  ----------  ---------------  -------------  ---
  default                     dcgm-exporter-tlmt8                                                0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h16m
  gpu-operator-resources      gpu-feature-discovery-df2fz                                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         71m
  gpu-operator-resources      gpu-operator-node-feature-discovery-worker-n7bsh                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h36m
  gpu-operator-resources      nvidia-container-toolkit-daemonset-6m526                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h35m
  gpu-operator-resources      nvidia-dcgm-exporter-cskjj                                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h35m
  gpu-operator-resources      nvidia-device-plugin-daemonset-sd2kt                               0 (0%)        0 (0%)      0 (0%)           0 (0%)         71m
  gpu-operator-resources      nvidia-operator-validator-bmmlr                                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h35m
  intrallm                    text-generation-webui-78f9656fdb-dsxz8                             0 (0%)        0 (0%)      0 (0%)           0 (0%)         69m
  kube-system                 calico-kube-controllers-759cd8b574-dflk7                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h46m
  kube-system                 calico-node-pwscp                                                  250m (0%)     0 (0%)      0 (0%)           0 (0%)         3h46m
  kube-system                 dashboard-metrics-scraper-6b96ff7878-t78jf                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h40m
  kube-system                 hostpath-provisioner-5fbc49d86c-9x7qf                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h5m
  kube-system                 kubernetes-dashboard-7d869bcd96-vn29g                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h40m
  kube-system                 metrics-server-df8dbf7f5-mtkk7                                     100m (0%)     0 (0%)      200Mi (0%)       0 (0%)         3h40m
  prometheus                  kube-prometheus-stack-1738204901-prometheus-node-exporter-qqjkr    0 (0%)        0 (0%)      0 (0%)           0 (0%)         137m
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                350m (0%)   0 (0%)
  memory             200Mi (0%)  0 (0%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
  nvidia.com/gpu     1           1
Events:              <none>


Name:               node2
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    feature.node.kubernetes.io/cpu-cpuid.ADX=true
                    feature.node.kubernetes.io/cpu-cpuid.AESNI=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX2=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512BF16=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512BITALG=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512BW=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512CD=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512DQ=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512F=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512IFMA=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VBMI=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VBMI2=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VL=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VNNI=true
                    feature.node.kubernetes.io/cpu-cpuid.AVX512VPOPCNTDQ=true
                    feature.node.kubernetes.io/cpu-cpuid.CETSS=true
                    feature.node.kubernetes.io/cpu-cpuid.CLZERO=true
                    feature.node.kubernetes.io/cpu-cpuid.CMPXCHG8=true
                    feature.node.kubernetes.io/cpu-cpuid.CPBOOST=true
                    feature.node.kubernetes.io/cpu-cpuid.CPPC=true
                    feature.node.kubernetes.io/cpu-cpuid.EFER_LMSLE_UNS=true
                    feature.node.kubernetes.io/cpu-cpuid.FLUSH_L1D=true
                    feature.node.kubernetes.io/cpu-cpuid.FMA3=true
                    feature.node.kubernetes.io/cpu-cpuid.FP256=true
                    feature.node.kubernetes.io/cpu-cpuid.FSRM=true
                    feature.node.kubernetes.io/cpu-cpuid.FXSR=true
                    feature.node.kubernetes.io/cpu-cpuid.FXSROPT=true
                    feature.node.kubernetes.io/cpu-cpuid.GFNI=true
                    feature.node.kubernetes.io/cpu-cpuid.IBPB=true
                    feature.node.kubernetes.io/cpu-cpuid.IBRS=true
                    feature.node.kubernetes.io/cpu-cpuid.IBRS_PREFERRED=true
                    feature.node.kubernetes.io/cpu-cpuid.IBRS_PROVIDES_SMP=true
                    feature.node.kubernetes.io/cpu-cpuid.IBS=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSBRNTRGT=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSFETCHSAM=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSFFV=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSOPCNT=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSOPCNTEXT=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSOPSAM=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSRDWROPCNT=true
                    feature.node.kubernetes.io/cpu-cpuid.IBSRIPINVALIDCHK=true
                    feature.node.kubernetes.io/cpu-cpuid.IBS_FETCH_CTLX=true
                    feature.node.kubernetes.io/cpu-cpuid.IBS_OPFUSE=true
                    feature.node.kubernetes.io/cpu-cpuid.IBS_PREVENTHOST=true
                    feature.node.kubernetes.io/cpu-cpuid.IBS_ZEN4=true
                    feature.node.kubernetes.io/cpu-cpuid.INT_WBINVD=true
                    feature.node.kubernetes.io/cpu-cpuid.INVLPGB=true
                    feature.node.kubernetes.io/cpu-cpuid.LAHF=true
                    feature.node.kubernetes.io/cpu-cpuid.LBRVIRT=true
                    feature.node.kubernetes.io/cpu-cpuid.MCAOVERFLOW=true
                    feature.node.kubernetes.io/cpu-cpuid.MOVBE=true
                    feature.node.kubernetes.io/cpu-cpuid.MOVU=true
                    feature.node.kubernetes.io/cpu-cpuid.MSRIRC=true
                    feature.node.kubernetes.io/cpu-cpuid.NRIPS=true
                    feature.node.kubernetes.io/cpu-cpuid.OSXSAVE=true
                    feature.node.kubernetes.io/cpu-cpuid.PPIN=true
                    feature.node.kubernetes.io/cpu-cpuid.PSFD=true
                    feature.node.kubernetes.io/cpu-cpuid.RDPRU=true
                    feature.node.kubernetes.io/cpu-cpuid.SEV=true
                    feature.node.kubernetes.io/cpu-cpuid.SEV_64BIT=true
                    feature.node.kubernetes.io/cpu-cpuid.SEV_ALTERNATIVE=true
                    feature.node.kubernetes.io/cpu-cpuid.SEV_DEBUGSWAP=true
                    feature.node.kubernetes.io/cpu-cpuid.SEV_ES=true
                    feature.node.kubernetes.io/cpu-cpuid.SEV_RESTRICTED=true
                    feature.node.kubernetes.io/cpu-cpuid.SEV_SNP=true
                    feature.node.kubernetes.io/cpu-cpuid.SHA=true
                    feature.node.kubernetes.io/cpu-cpuid.SME=true
                    feature.node.kubernetes.io/cpu-cpuid.SME_COHERENT=true
                    feature.node.kubernetes.io/cpu-cpuid.SPEC_CTRL_SSBD=true
                    feature.node.kubernetes.io/cpu-cpuid.SSE4A=true
                    feature.node.kubernetes.io/cpu-cpuid.STIBP=true
                    feature.node.kubernetes.io/cpu-cpuid.STIBP_ALWAYSON=true
                    feature.node.kubernetes.io/cpu-cpuid.SUCCOR=true
                    feature.node.kubernetes.io/cpu-cpuid.SVM=true
                    feature.node.kubernetes.io/cpu-cpuid.SVMDA=true
                    feature.node.kubernetes.io/cpu-cpuid.SVMFBASID=true
                    feature.node.kubernetes.io/cpu-cpuid.SVML=true
                    feature.node.kubernetes.io/cpu-cpuid.SVMNP=true
                    feature.node.kubernetes.io/cpu-cpuid.SVMPF=true
                    feature.node.kubernetes.io/cpu-cpuid.SVMPFT=true
                    feature.node.kubernetes.io/cpu-cpuid.SYSCALL=true
                    feature.node.kubernetes.io/cpu-cpuid.SYSEE=true
                    feature.node.kubernetes.io/cpu-cpuid.TLB_FLUSH_NESTED=true
                    feature.node.kubernetes.io/cpu-cpuid.TOPEXT=true
                    feature.node.kubernetes.io/cpu-cpuid.TSCRATEMSR=true
                    feature.node.kubernetes.io/cpu-cpuid.VAES=true
                    feature.node.kubernetes.io/cpu-cpuid.VMCBCLEAN=true
                    feature.node.kubernetes.io/cpu-cpuid.VMPL=true
                    feature.node.kubernetes.io/cpu-cpuid.VMSA_REGPROT=true
                    feature.node.kubernetes.io/cpu-cpuid.VPCLMULQDQ=true
                    feature.node.kubernetes.io/cpu-cpuid.VTE=true
                    feature.node.kubernetes.io/cpu-cpuid.WBNOINVD=true
                    feature.node.kubernetes.io/cpu-cpuid.X87=true
                    feature.node.kubernetes.io/cpu-cpuid.XGETBV1=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVE=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVEC=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVEOPT=true
                    feature.node.kubernetes.io/cpu-cpuid.XSAVES=true
                    feature.node.kubernetes.io/cpu-hardware_multithreading=true
                    feature.node.kubernetes.io/cpu-model.family=25
                    feature.node.kubernetes.io/cpu-model.id=17
                    feature.node.kubernetes.io/cpu-model.vendor_id=AMD
                    feature.node.kubernetes.io/cpu-rdt.RDTCMT=true
                    feature.node.kubernetes.io/cpu-rdt.RDTL3CA=true
                    feature.node.kubernetes.io/cpu-rdt.RDTMBM=true
                    feature.node.kubernetes.io/cpu-rdt.RDTMON=true
                    feature.node.kubernetes.io/kernel-config.NO_HZ=true
                    feature.node.kubernetes.io/kernel-config.NO_HZ_FULL=true
                    feature.node.kubernetes.io/kernel-version.full=6.8.0-52-generic
                    feature.node.kubernetes.io/kernel-version.major=6
                    feature.node.kubernetes.io/kernel-version.minor=8
                    feature.node.kubernetes.io/kernel-version.revision=0
                    feature.node.kubernetes.io/memory-numa=true
                    feature.node.kubernetes.io/network-sriov.capable=true
                    feature.node.kubernetes.io/pci-10de.present=true
                    feature.node.kubernetes.io/pci-1a03.present=true
                    feature.node.kubernetes.io/pci-8086.present=true
                    feature.node.kubernetes.io/pci-8086.sriov.capable=true
                    feature.node.kubernetes.io/storage-nonrotationaldisk=true
                    feature.node.kubernetes.io/system-os_release.ID=ubuntu
                    feature.node.kubernetes.io/system-os_release.VERSION_ID=22.04
                    feature.node.kubernetes.io/system-os_release.VERSION_ID.major=22
                    feature.node.kubernetes.io/system-os_release.VERSION_ID.minor=04
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=node2
                    kubernetes.io/os=linux
                    microk8s.io/cluster=true
                    node.kubernetes.io/microk8s-controlplane=microk8s-controlplane
                    nvidia.com/cuda.driver.major=535
                    nvidia.com/cuda.driver.minor=183
                    nvidia.com/cuda.driver.rev=01
                    nvidia.com/cuda.runtime.major=12
                    nvidia.com/cuda.runtime.minor=2
                    nvidia.com/device-plugin.config=Tesla-V100-PCIE-16GB
                    nvidia.com/gfd.timestamp=1738208849
                    nvidia.com/gpu.compute.major=7
                    nvidia.com/gpu.compute.minor=0
                    nvidia.com/gpu.count=1
                    nvidia.com/gpu.deploy.container-toolkit=true
                    nvidia.com/gpu.deploy.dcgm=true
                    nvidia.com/gpu.deploy.dcgm-exporter=true
                    nvidia.com/gpu.deploy.device-plugin=true
                    nvidia.com/gpu.deploy.driver=true
                    nvidia.com/gpu.deploy.gpu-feature-discovery=true
                    nvidia.com/gpu.deploy.node-status-exporter=true
                    nvidia.com/gpu.deploy.operator-validator=true
                    nvidia.com/gpu.family=volta
                    nvidia.com/gpu.machine=RS700A-E12-RS4U
                    nvidia.com/gpu.memory=16384
                    nvidia.com/gpu.present=true
                    nvidia.com/gpu.product=Tesla-V100-PCIE-16GB-SHARED
                    nvidia.com/gpu.replicas=4
                    nvidia.com/mig.capable=false
                    nvidia.com/mig.strategy=single
Annotations:        nfd.node.kubernetes.io/feature-labels:
                      cpu-cpuid.ADX,cpu-cpuid.AESNI,cpu-cpuid.AVX,cpu-cpuid.AVX2,cpu-cpuid.AVX512BF16,cpu-cpuid.AVX512BITALG,cpu-cpuid.AVX512BW,cpu-cpuid.AVX512...
                    nfd.node.kubernetes.io/master.version: v0.14.2
                    nfd.node.kubernetes.io/worker.version: v0.14.2
                    node.alpha.kubernetes.io/ttl: 0
                    nvidia.com/gpu-driver-upgrade-enabled: true
                    projectcalico.org/IPv4Address: xxx.xxx.xxx.xxx/24
                    projectcalico.org/IPv4VXLANTunnelAddr: 10.1.104.0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Thu, 30 Jan 2025 12:20:49 +1100
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  node2
  AcquireTime:     <unset>
  RenewTime:       Thu, 30 Jan 2025 15:58:56 +1100
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Thu, 30 Jan 2025 12:20:59 +1100   Thu, 30 Jan 2025 12:20:59 +1100   CalicoIsUp                   Calico is running on this node
  MemoryPressure       False   Thu, 30 Jan 2025 15:58:53 +1100   Thu, 30 Jan 2025 12:20:49 +1100   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Thu, 30 Jan 2025 15:58:53 +1100   Thu, 30 Jan 2025 12:20:49 +1100   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Thu, 30 Jan 2025 15:58:53 +1100   Thu, 30 Jan 2025 12:20:49 +1100   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Thu, 30 Jan 2025 15:58:53 +1100   Thu, 30 Jan 2025 12:20:49 +1100   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  xxx.xxx.xxx.xxx
  Hostname:    node2
Capacity:
  cpu:                64
  ephemeral-storage:  459850824Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             131563384Ki
  nvidia.com/gpu:     4
  pods:               110
Allocatable:
  cpu:                64
  ephemeral-storage:  458802248Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             131460984Ki
  nvidia.com/gpu:     4
  pods:               110
System Info:
  Machine ID:                 2a008e07c71648caa5dded97337a55ae
  System UUID:                86069482-6cc6-78b8-44bf-a036bccb9707
  Boot ID:                    2359e07f-8a87-45ad-8250-d598147ea81f
  Kernel Version:             6.8.0-52-generic
  OS Image:                   Ubuntu 22.04.5 2024.10.16 LTS (Cubic 2024-10-16 14:33)
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.6.28
  Kubelet Version:            v1.31.5
  Kube-Proxy Version:         v1.31.5
Non-terminated Pods:          (18 in total)
  Namespace                   Name                                                               CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                                               ------------  ----------  ---------------  -------------  ---
  default                     dcgm-exporter-ld7bw                                                0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h16m
  gpu-operator-resources      gpu-feature-discovery-4gxnf                                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         71m
  gpu-operator-resources      gpu-operator-86765669fc-9cs22                                      200m (0%)     500m (0%)   100Mi (0%)       350Mi (0%)     3h36m
  gpu-operator-resources      gpu-operator-node-feature-discovery-gc-555ccf7687-764rh            0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h36m
  gpu-operator-resources      gpu-operator-node-feature-discovery-master-68d694564d-fbrnj        0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h36m
  gpu-operator-resources      gpu-operator-node-feature-discovery-worker-89wfs                   0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h36m
  gpu-operator-resources      nvidia-container-toolkit-daemonset-bc8zl                           0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h35m
  gpu-operator-resources      nvidia-dcgm-exporter-4qsv2                                         0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h35m
  gpu-operator-resources      nvidia-device-plugin-daemonset-xvckj                               0 (0%)        0 (0%)      0 (0%)           0 (0%)         71m
  gpu-operator-resources      nvidia-operator-validator-gvk6z                                    0 (0%)        0 (0%)      0 (0%)           0 (0%)         3h35m
  kube-system                 calico-node-6s54w                                                  250m (0%)     0 (0%)      0 (0%)           0 (0%)         3h38m
  kube-system                 coredns-7896dbf49-cr5m7                                            100m (0%)     0 (0%)      70Mi (0%)        170Mi (0%)     3h36m
  prometheus                  alertmanager-kube-prometheus-stack-1738-alertmanager-0             0 (0%)        0 (0%)      200Mi (0%)       0 (0%)         137m
  prometheus                  kube-prometheus-stack-1738-operator-7974c894dc-qbqdl               0 (0%)        0 (0%)      0 (0%)           0 (0%)         137m
  prometheus                  kube-prometheus-stack-1738204901-grafana-66d9864858-vjc77          0 (0%)        0 (0%)      0 (0%)           0 (0%)         137m
  prometheus                  kube-prometheus-stack-1738204901-kube-state-metrics-78789dk9rdc    0 (0%)        0 (0%)      0 (0%)           0 (0%)         137m
  prometheus                  kube-prometheus-stack-1738204901-prometheus-node-exporter-z98pc    0 (0%)        0 (0%)      0 (0%)           0 (0%)         137m
  prometheus                  prometheus-kube-prometheus-stack-1738-prometheus-0                 0 (0%)        0 (0%)      0 (0%)           0 (0%)         137m
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                550m (0%)   500m (0%)
  memory             370Mi (0%)  520Mi (0%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-1Gi      0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
  nvidia.com/gpu     0           0
Events:              <none>
```
