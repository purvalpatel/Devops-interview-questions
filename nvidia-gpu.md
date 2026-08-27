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

### What does the NVIDIA driver do?

### What is NVML?
### What is the NVIDIA Container Toolkit?
### What is the NVIDIA Kubernetes Device Plugin?
### How does Kubernetes know a node has GPUs?
```
Explain:

resources:
  limits:
    nvidia.com/gpu: 1
```
### What happens internally when this Pod is scheduled?
### How does kubelet expose the GPU to the container?
### What is an extended resource?
### How does the NVIDIA Device Plugin communicate with kubelet?
### What happens if the device plugin crashes?
### What happens if nvidia-smi works on the host but fails inside the container?
### How do you troubleshoot CUDA/driver mismatch?
### How do you troubleshoot a GPU Pod stuck in Pending?
### How do you monitor GPU health?
### What is DCGM?
### How does DCGM Exporter work with Prometheus?
### How do you detect GPU ECC errors?
### How would you automatically quarantine a failed GPU?
