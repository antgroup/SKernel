# SKernel

[![EuroSys '26](https://img.shields.io/badge/EuroSys-2026-blue)](https://doi.org/10.1145/3767295.3769332)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Production](https://img.shields.io/badge/Production-40K%2B%20nodes-green)]()

**SKernel** is an elastic and efficient secure container system for cloud-native scenarios, featuring an innovative **Split-Kernel Architecture** that achieves near-native performance while maintaining container elasticity.

> 📄 Based on EuroSys '26 paper: *"SKernel: An Elastic and Efficient Secure Container System at Scale with a Split-Kernel Architecture"*

---

## 📖 Table of Contents

- [Introduction](#-introduction)
- [Architecture Overview](#-architecture-overview)
- [Key Features](#-key-features)
- [Performance Benefits](#-performance-benefits)
- [Production Scale](#-production-scale)
- [Open Source Components](#-open-source-components)
- [Citation](#-citation)
- [Related Projects](#-related-projects)

---

## 🌟 Introduction

Secure containers leverage hardware virtualization to isolate container sandboxes, assigning each container its own guest kernel to effectively mitigate security risks from shared kernels in traditional containers. However, existing solutions face a **fundamental trade-off between elasticity and performance**:

- **VM-based approaches** (e.g., Kata Containers): Excellent performance but limited resource elasticity, struggling with bursty cloud-native workloads
- **Lightweight approaches** (e.g., gVisor): Good elasticity but significant performance degradation due to frequent guest-host context switches

**SKernel** resolves this trade-off through an innovative split-kernel architecture. We decouple the guest kernel into two specialized components:

| Component | Responsibility | Design Goal |
|-----------|----------------|-------------|
| **SKernel-R** (Resource Kernel) | Resource management (CPU/memory) | Collaborate with host kernel for global resource elasticity |
| **SKernel-D** (Data Kernel) | High-performance I/O operations | Manage directly within guest kernel for high throughput and low latency |

This design enables SKernel to deliver both **VM-level performance** and **process-level elasticity**, while maintaining excellent backward compatibility and hardware-enforced isolation.

---

## 🏗️ Architecture Overview

![SKernel Architecture](./SKernel_Architecture.pdf)

### Component Details

| Component | Code Size | Language | Responsibility |
|-----------|-----------|----------|----------------|
| **SKernel-V** | 4,421 LoC | C (Kernel Module) | Lightweight virtualization layer providing resource call channel |
| **SKernel-R** | 44,314 LoC | Go + Rust | Based on gVisor Sentry, responsible for resource management |
| **SKernel-D** | 18,728 LoC | C | High-performance network stack + filesystem |

---

## ✨ Key Features

### 🔹 Fast Syscall Forwarding

- **L1 Syscall Optimization**: ABI-compliant syscall shim enables fast path for system calls; performance-critical I/O operations bypass SKernel-R and are handled directly by SKernel-D
- **L2 Syscall Optimization**: Direct host kernel invocation via vmcall instruction, shortening call path from `GR0→HR0→HR3→HR0` to `GR0→HR0`

### 🔹 High-Performance Network Stack for Cloud-Native

- **Device Passthrough**: Network devices are passed through to SKernel-D (not applications), maintaining compatibility and resource flexibility
- **Hybrid Thread Model**: Integrates inline model and look-aside model to optimize batching efficiency and minimize latency
- **Adaptable I/O Notification**: Dynamic switching between interrupt and polling modes based on traffic patterns

### 🔹 File Descriptor-Based Container Filesystem

- **EROFS Support**: Read-only filesystem based on Enhanced Read-Only File System (EROFS) for efficient compression and fast reads
- **Eliminates Gofer Proxy**: Removes IPC-based Gofer process, reducing host kernel interactions
- **Minimized Attack Surface**: Requires only a few file descriptors instead of entire directory trees, significantly improving security

### 🔹 Dynamic DMA Memory Management

- **Collaborative Paging**: Dynamic DMA memory allocation/reclamation via host APIs (`fallocate()`/`ioctl()`)
- **Memory Overcommitment Support**: Solves memory overcommitment challenges in device passthrough scenarios
- **Elastic Scaling**: Dynamically adjusts DMA memory resources based on workload fluctuations

### 🔹 TLB Sharing Optimization

- **Per-Process PCID + Per-Instance VPID**: Reduces TLB flush frequency, improves TLB entry utilization
- **Microarchitecture Optimizations**: Supports Transparent Huge Pages

---

## 📊 Performance Benefits

### Macro-Benchmark Evaluation

| Application | SKernel vs gVisor | SKernel vs Kata | SKernel vs runc |
|-------------|-------------------|-----------------|-----------------|
| **Microservice (sofaload)** | 1.68× | 1.23× | 1.18× |
| **Redis** | 1.81× - 2.34× | 1.13× - 1.27× | 1.07× - 1.18× |
| **Nginx** | 3.42× - 4.04× | 1.69× - 1.98× | 1.40× - 1.67× |
| **MySQL** | **4.49×** | 1.47× | 1.28× |

### Micro-Benchmark Evaluation

| Operation Type | Specific Operation | runc | Kata | gVisor | SKernel |
|----------------|-------------------|------|------|--------|---------|
| **L1 Syscall** | getpid() | N/A | 0.345μs | 0.849μs | **0.320μs** |
| **L2 Syscall** | getpid() | 0.266μs | N/A | 6μs | **0.289μs** |
| **Network I/O** | ICMP | 130μs | 140μs | 250μs | **110μs** |
| **Network I/O** | TCP | 420μs | 470μs | 640μs | **390μs** |
| **File I/O** | Write BW | 795 MB/s | 1293 MB/s | 359 MB/s | **1182 MB/s** |
| **File I/O** | Read BW | 1725 MB/s | 1463 MB/s | 465 MB/s | **1700 MB/s** |

### Production Environment Performance

In Ant Group's production environment under the same workload:
- **CPU usage reduced by 0.53% - 3.99%** (compared to runc)
- **8.73% performance improvement under peak traffic**
- Supports 600+ TPS high-concurrency scenarios

---

## 🏢 Production Scale

SKernel has been deployed at scale in Ant Group's production environment:

| Metric | Scale |
|--------|-------|
| Physical Nodes | **40,000+** |
| Secure Containers | **100,000+** |
| Microservice Applications | **2,000+** |
| Daily Requests | **Billions** |
| Production Deployment | **Years of validation** |

Supported scenarios:
- ✅ Long-running microservices
- ✅ Serverless computing (AFaaS platform)
- ✅ E-commerce shopping festival peak traffic
- ✅ Multi-tenant public cloud services

---

## 🔓 Open Source Components

Core SKernel technologies have been contributed to the open source community:

### SKernel-V

Based on lightweight hypervisor design, integrated into **SlimVM** project:

🔗 **[github.com/antgroup/slimvm](https://github.com/antgroup/slimvm?tab=readme-ov-file#citation)**

```bibtex
@misc{slimvm,
  title = {SlimVM: Lightweight Virtualization for Cloud-Native Workloads},
  author = {Ant Group},
  year = {2024},
  howpublished = {\url{https://github.com/antgroup/slimvm}}
}
```

### SKernel-D

Core technologies upstreamed to **gVisor** project:

| PR | Feature | Status |
|----|---------|--------|
| [#9308](https://github.com/google/gvisor/pull/9308), [#9648](https://github.com/google/gvisor/pull/9648) | Filesystem support | Merged |
| [#9551](https://github.com/google/gvisor/pull/9551) | Network stack optimization | Merged |
---

## 📚 Citation

If you use SKernel in your research, please cite our paper:

```bibtex
@inproceedings{chai2026skernel,
  author = {Chai, Xiaohu and Hu, Keyang and Tan, Jianfeng and Bie, Tiwei and Tan, Guotao and Zhou, Tianyu and Shen, Anqi and Shen, Dawei and Yang, Xinyao and Chen, Xin and Wang, Xu and Yu, Feng and He, Zhengyu and Du, Dong and Xia, Yubin and Chen, Kang and Chen, Yu},
  title = {SKernel: An Elastic and Efficient Secure Container System at Scale with a Split-Kernel Architecture},
  booktitle = {Proceedings of the European Conference on Computer Systems (EuroSys '26)},
  year = {2026},
  publisher = {ACM},
  doi = {10.1145/3767295.3769332},
  url = {https://doi.org/10.1145/3767295.3769332}
}
```

**Author Affiliations**:
- Tsinghua University
- Ant Group
- Shanghai Jiao Tong University
- Quan Cheng Laboratory

---

## 🔗 Related Projects

| Project | Description | Link |
|---------|-------------|------|
| **gVisor** | Application-level kernel, foundation of SKernel-R | [gvisor.dev](https://gvisor.dev) |
| **Kata Containers** | VM-based secure container solution | [katacontainers.io](https://katacontainers.io) |
| **SlimVM** | Lightweight virtualization platform | [github.com/antgroup/slimvm](https://github.com/antgroup/slimvm) |
| **EROFS** | Enhanced Read-Only File System | [kernel.org](https://www.kernel.org/doc/html/latest/filesystems/erofs.html) |
| **DPDK** | Data Plane Development Kit | [dpdk.org](https://www.dpdk.org) |

---

## 📝 License

SKernel is released under the [Apache License 2.0](LICENSE).
