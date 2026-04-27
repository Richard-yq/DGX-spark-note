# DGX-spark-note

This repository contains essential documentation and SOPs for integrating **NVIDIA Aerial L1** with **OpenAirInterface (OAI) gNB** on an **NVIDIA DGX Spark** system.

---

## 📑 Table of Contents

- [**Quick Start Guide**](DGX-spark-quick-start.md) - High-level overview and links to all deployment paths.
- [**Kernel & Environment Setup**](guide-kernel-setup-troubleshooting.md) - Vital host-level optimizations (Hugepages, GRUB parameters, RT Kernel).
- [**Aerial cuBB Build Guide**](guide-build-cubb.md) - Detailed steps for source compilation in the Aerial development container.
- [**OAI-Aerial Integration Guide**](Aerial_OAI_Integration_Guide.md) - *Primary Guide:* Step-by-step FAPI split integration using Docker Compose.
- [**Docker & Troubleshooting**](guide-docker-troubleshooting.md) - Common fixes for container networking, image registry, and compilation bugs.

---

## 🔍 Latest Reference Material

The following documentation is from the **April 2026 update** of the NVIDIA Aerial docs and should be used as the primary technical ground truth:

> [!TIP]
> [**Official NVIDIA Aerial Installation Guide (DGX Spark)**](https://docs.nvidia.com/aerial/cuda-accelerated-ran/latest/install_guide/installing_tools_spark.html)

### Key Specifications (Latest)
- **Kernel Version:** `6.17.0-1014-nvidia`
- **Hugepage Size:** `1G` (default_hugepagesz=1G)
- **CUDA Version:** `13.1`
- **DOCA OFED:** `3.2.1`

---

## 🛠 Useful Commands

### Check Kernel & Environment
```bash
# Check loaded kernel
uname -r

# Check hugepages (Free count should be > 0)
grep -i huge /proc/meminfo

# Check command line parameters
cat /proc/cmdline
```

### Check GPU & NIC
```bash
# Verify GPU (GB10) detection
nvidia-smi

# Verify ConnectX-7 NIC
lspci | grep Mellanox
```

---

*Last Updated: April 27, 2026*
