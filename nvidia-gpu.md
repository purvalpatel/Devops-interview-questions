### Explain NVIDIA GPU architecture at a high level.

Major components:
- SMs — Streaming Multiprocessors
- CUDA cores
- Tensor cores
- L1/shared memory    : L1 is a small, very fast cache associated with an SM.
- L2 cache            : L2 is larger than L1 and is generally shared across the GPU's SMs.
- HBM/GDDR GPU memory
- GPU interconnects such as NVLink
- Copy engines / DMA
- PCIe interface

```
Physical Nvidia GPUs
      |   H100, H200, RTX 4090 etc.
      |   PCIe device exists at hardware level.
      |   Kubernetes can not directly detect hardware device, it requires
  Nvidia Kernel driver
      | Driver installed on host : `lsmod | grep -i nvidia`
      | `nvidia-smi`
      | Now GPU is accessible with Linux, still kubernetes cant read it, here comes.
  NVML ( nvidia management library)
      | it allows application to query  and manage NVIDIA GPUs.
      | interface between Application and NVIDIA driver
      |
  NVIDIA Container toolkit
      |   Container needs to access the Nvidia GPUs which is exposed on Linux
      |   /dev/nvidia*, here nvidia container toolkit comes in.
  Container runtime
      | docker, CRI-O, containerd
      |
      |
  Nvidia Device Plugin
      |   Kubernetes cant attach the GPU device directly, needs to use this to advertise GPUs to K8s.
      |
      |
  K8s API
      - Now we can use device into K8s 
      `nvidia.com/gpu=1`
```
### What does the NVIDIA driver do?

Nvidia Driver is software layer that allows operating system and Applications to communicate with NVIDIA gpus.

1. Initialize GPU
When machine boots nvidia driver discovers and initializes the GPU. you can verify with `nvidia-smi`

2. Manages GPU memory
Driver manage the GPU memory which is required by the Applications.

3. Submits work to GPU
The driver is involved in getting GPU work from the application to the hardware.

4. Provides the GPU access.
Linux exposes the Nvidia GPUs such as `/dev/nvidia*`

5. Supports features such as MIG

NVIDIA Driver  : Responsible for communicating with the GPU hardware and providing the low-level interface.


CUDA : Provides the programming/runtime environment that allows applications to use the GPU.

### What is NVML?

Nvidia Management Library.

its a c-based API/Library for monitoring and managing NVIDIA GPU's.

It allows applications to query and manage the NVIDIA GPU. interface between application and NVIDIA Driver.

### What is the NVIDIA Container Toolkit?
- Software that allows Containers to use Nvidia GPUs.

- Container toolkit connects the NVIDIA GPUs to the applications running inside the container.

### What is the NVIDIA Kubernetes Device Plugin?
- Kubernets directly can not access the GPUs available on host. and it can not attach to the POD directly.
- Kubernetes device plugin is the kubernetes resource that is running as daemon on all nodes and which exposes the GPUs on Node. like `nvidia.com/gpu=1`

### How does Kubernetes know a node has GPUs?
```
Explain:

resources:
  limits:
    nvidia.com/gpu: 1
```
- Makes sure that Nvidia Device plugin is installed.
- check the daemon set and pod of nvidia-device plugin is running fine on all servers.
- check the GPUs are exposed on node using `nvidia describe node <node-name>`


### What happens internally when this Pod is scheduled?
```
Kubectl apply ( pod Creation )
  |
Scheduling queue
  |
Filtering
  |
Scoring
  |
Binding
  |
Kubelet
  |
Running
  |
Container
```
Complete flow:
```
Kubectl apply
  |
KubeAPI server
  - Authentication
  - Authorization
  - Validation
  - Admission
  |
etcd
  - Pod status stored 
  phase = pending
  node.name = empty
  |
  Scheduler watches pod
  |
  Scheduling Queue
  | 
  Find fesible nodes
  |
  Filtering
  |
  Reject Nodes
  Feasible Nodes  -- Scoring  -- Select best node -- PERMIT
                                                        |
                                                      API server
                                                        |
                                                      etcd
                                                        |
                                                      Kubelet
                                                        |
                                                      Containerd
                                                        |
                                                      Running
```


### How does kubelet expose the GPU to the container? How does the NVIDIA Device Plugin communicate with kubelet?
```
Kubelet
  |
Kubernetes Nvidia device plugin
  |
CRI-O
  |
Containerd
  |
Nvidia container toolkit
  |   ( Inject GPU devices + Libraries)
 Pod
  |
Container
  |
Nvidia Driver
  |
 GPU
```
### What is an extended resource?

Extended resource is Custom resources that kubernetes doesnt natively understand.

Like kubernetes will understand:
```
resources:
  limits:
    cpu: "4"
    memory: "8Gi"
```
But it will not understand:
```
nvidia.com/gpu
```
So Nvidia device plugin is extended resource.

### What happens if the device plugin crashes?
The running devices will remain as it is. the impact is only on resource recovery, Future pods.

### What happens if nvidia-smi works on the host but fails inside the container?
If nvidia-smi works on the host but fails inside the container, I know the GPU and host driver are generally healthy, 

so I troubleshoot the container integration layer. 

I first verify `NVIDIA Container Toolkit` with `nvidia-container-cli` info, then verify the container runtime is configured with the NVIDIA runtime. 

I run a minimal CUDA container using `--gpus all` and check whether `/dev/nvidia*`, `libnvidia-ml.so`, and `libcuda.so` are injected. 

If Docker works but Kubernetes doesn't, I check `containerd`, the `NVIDIA runtime configuration`, `kubelet`, and the `NVIDIA device plugin`. 

Finally, I check `cgroup/device` permissions and runtime logs.


### How do you troubleshoot CUDA/driver mismatch?
I first separate the host driver from the CUDA runtime. 

I run `nvidia-smi` on the host to verify the driver and GPU, then check the CUDA runtime used by the application with tools such as `nvcc --version`, PyTorch's `torch.version.cuda`, or the container's CUDA version. 

I verify that the host NVIDIA driver is new enough for that runtime. 

Then I test a clean CUDA container with  `docker run --gpus all ... nvidia-smi` to isolate the NVIDIA Container Toolkit and runtime layer. 

If Kubernetes is involved, I check the `NVIDIA device plugin`, `kubelet`, and `nvidia.com/gpu` allocatable resources. 

Finally, I inspect `libcuda.so`, `libcudart.so`, `LD_LIBRARY_PATH`, and application-specific libraries. 

I don't assume that different CUDA versions between the host and container are automatically a problem.


### How do you troubleshoot a GPU Pod stuck in Pending?
For a GPU Pod stuck in Pending, I first run `kubectl describe pod` and look at the `FailedScheduling` event. 

I determine whether the problem is `insufficient nvidia.com/gpu`, CPU or memory, node affinity, taints, or another scheduling constraint. 

Then I check the node's GPU Capacity and Allocatable resources with `kubectl describe node`. 

If the GPU resource is missing, I investigate the `NVIDIA device plugin` and `kubelet`, followed by the NVIDIA driver and container runtime. 


If the cluster uses `MIG`, I also verify that the Pod is requesting the correct `MIG resource` such as `nvidia.com/mig-3g.40gb`. 

I only start debugging CUDA libraries after the Pod has actually been scheduled and started.

### How do you monitor GPU health?
DCGM Exporter
Nvidia NSight Systems

### What is DCGM?
Datacenter GPU Manager.

It is an NVIDIA software framework used to monitor, manage, and diagnose NVIDIA GPUs, especially in data-center and Kubernetes environments.

### How does DCGM Exporter work with Prometheus?
DCGM Exporter exposes the metrics and and prometheus will collect that data.
```
GPU utilization
GPU memory utilization
GPU temperature
Power usage
ECC errors
PCIe information
NVLink statistics
XID errors
Clocks
GPU health
Process information
MIG-related metrics
```

### How do you detect GPU ECC errors?
I first use `nvidia-smi -q -d ECC` to check correctable and uncorrectable ECC counters. 

I also check `nvidia-smi -q` for retired pages and other health information, and inspect kernel logs for NVIDIA XID errors. 

In a production Kubernetes cluster, I use `DCGM` and `DCGM Exporter` to continuously expose ECC metrics to Prometheus and create alerts. 

I distinguish correctable errors from uncorrectable errors; repeated correctable errors require investigation, while persistent uncorrectable ECC errors can indicate a serious GPU or HBM problem. 

If the GPU is unhealthy, I identify the affected workload, prevent further scheduling onto the GPU where appropriate, collect diagnostics, and escalate for hardware replacement if necessary.

### How would you automatically quarantine a failed GPU?

I would automate GPU quarantine using `DCGM metrics` and a `Kubernetes controller`. 

`DCGM` detects conditions such as uncorrectable `ECC` or critical `XID errors`, `Prometheus and Alertmanager` trigger the automation, and the controller identifies the affected GPU using its UUID. 

I would mark that `device unhealthy` so the `NVIDIA device plugin` stops allocating it to new workloads. 

I would then identify workloads using that `GPU` and restart them if they are safely restartable. 

If the failure indicates a node-level problem, such as a driver or PCIe failure affecting multiple GPUs, I would `cordon` and potentially drain the entire node. 

I would also require verification or persistence of the error before quarantine to avoid reacting to transient events.
