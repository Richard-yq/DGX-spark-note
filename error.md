0626

Searched web: ""Module already installed, override by specifying --force" dkms dpkg"

這個問題發生的原因是：在執行 `sudo apt install` 時，系統同時在進行核心（Kernel）與 NVIDIA 驅動程式的更新，但在設定 `linux-image-6.17.0-1014-nvidia` 與 `linux-headers-6.17.0-1014-nvidia` 時，**DKMS（Dynamic Kernel Module Support）自動安裝程式失敗了**。

### 具體原因分析：
1. **模組衝突/已存在**：
   DKMS 嘗試為新核心 `6.17.0-1014-nvidia` 編譯並安裝 `gdrdrv/2.5.1` 與 `nvidia/590.48.01` 驅動模組，但發現這些模組的 `.ko.zst` 檔案已經存在於 `/lib/modules/6.17.0-1014-nvidia/updates/dkms/` 目錄下：
   ```text
   Module /lib/modules/6.17.0-1014-nvidia/updates/dkms/gdrdrv.ko.zst already installed at version 2.5, override by specifying --force
   Error! Installation aborted.
   ```
2. **安裝中斷**：
   因為預設情況下 DKMS 檢測到重複檔案會直接中斷（Aborted），導致 `dpkg` 配置核心套件失敗，並回傳錯誤碼。
3. **指令未完成**：
   由於 `sudo apt install` 發生錯誤中斷，後續的 `&& git lfs pull` 就沒有被執行。

---

### 解決方法：

您可以透過以下任一方法解決此問題，**推薦使用方法一（修改設定檔）**，這樣可以防止未來更新核心時再次遇到相同錯誤。

#### 方法一：開啟 DKMS 的自動覆寫（推薦）
1. **修改 DKMS 設定檔**，允許自動覆寫已存在的模組（以強製覆寫完成安裝）：
   ```bash
   echo "FORCE_MODULE_INSTALL=yes" | sudo tee -a /etc/dkms/framework.conf
   ```
2. **修復並重新配置未完成的套件**：
   ```bash
   sudo dpkg --configure -a
   ```
3. **確保所有相依性修復完成，並繼續執行原先的 `git lfs pull`**：
   ```bash
   sudo apt install -f
   cd ~/aerial-cuda-accelerated-ran/cubb_scripts/install/
   git lfs pull
   ```

#### 方法二：手動強制安裝出錯的模組
如果您不想修改全域設定，也可以手動對出錯的模組帶上 `--force` 參數：
1. **手動編譯並安裝模組**：
   ```bash
   sudo dkms install -m gdrdrv -v 2.5.1 -k 6.17.0-1014-nvidia --force
   sudo dkms install -m nvidia -v 590.48.01 -k 6.17.0-1014-nvidia --force
   ```
2. **重新配置套件並拉取 LFS 檔案**：
   ```bash
   sudo dpkg --configure -a
   cd ~/aerial-cuda-accelerated-ran/cubb_scripts/install/
   git lfs pull
   ```

---


# Troubleshooting: Aerial Container GPU Launch Failure on DGX Spark

## 環境

| 項目 | 內容 |
|------|------|
| 硬體 | NVIDIA DGX Spark (P4242) |
| GPU | NVIDIA GB10 (Architecture 12.1) |
| Driver | 590.48.01 |
| Container Runtime | nvidia-container-runtime 1.19.0 |
| Image | `nvcr.io/nvidia/aerial/aerial-cuda-accelerated-ran:26-1-cubb` |

---

## 症狀

執行 `run_aerial.sh` 時出現以下錯誤：

```
docker: Error response from daemon: failed to create task for container:
failed to create shim task: OCI runtime create failed: runc create failed:
unable to start container process: error during container init:
failed to fulfil mount request: open /usr/bin/nvidia-cuda-mps-control:
no such file or directory
```

---

## 根本原因

問題由兩個因素疊加造成：

1. **CDI spec 包含 MPS 路徑**：`/var/run/cdi/nvidia.yaml` 內記載了需要掛載 `/usr/bin/nvidia-cuda-mps-control` 和 `/usr/bin/nvidia-cuda-mps-server` 進容器，但容器 rootfs 內找不到這些路徑。

2. **nvidia-container-runtime mode 為 `auto`**：在 `--gpus all` flag 觸發下，runtime 自動選擇 CDI mode，CDI spec 裡的 MPS 掛載項目因此被強制執行。

---

## 解決步驟

### Step 1：移除 CDI spec 中的 MPS 條目

```bash
# 備份原始 CDI spec
sudo cp /var/run/cdi/nvidia.yaml /var/run/cdi/nvidia.yaml.bak

# 確認 MPS 相關行號
grep -n mps /var/run/cdi/nvidia.yaml

# 移除所有 MPS 相關行
sudo sed -i '/nvidia-cuda-mps/d' /var/run/cdi/nvidia.yaml

# 確認已移除
grep -n mps /var/run/cdi/nvidia.yaml
```

### Step 2：將 nvidia-container-runtime mode 改為 legacy

```bash
sudo sed -i 's/^mode = "auto"/mode = "legacy"/' /etc/nvidia-container-runtime/config.toml

# 確認修改結果
grep "^mode" /etc/nvidia-container-runtime/config.toml
# 預期輸出：mode = "legacy"
```

### Step 3：將 Docker 預設 runtime 設為 nvidia

編輯 `/etc/docker/daemon.json`：

```bash
sudo nano /etc/docker/daemon.json
```

內容改為：

```json
{
    "runtimes": {
        "nvidia": {
            "args": [],
            "path": "nvidia-container-runtime"
        }
    },
    "default-runtime": "nvidia"
}
```

### Step 4：重啟 Docker daemon

```bash
sudo systemctl restart docker
```

### Step 5：修改 [run_aerial.sh](file:///home/admin/aerial-cuda-accelerated-ran/cuPHY-CP/container/run_aerial.sh) 中的 GPU_FLAG

由於 `nvidia-container-runtime` 改為 `legacy` 模式，使用 `--gpus all` 直接呼叫 Container Hook 可能會報錯：
```
docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: error running prestart hook #0: exit status 1, stdout: , stderr: Using requested mode 'cdi'
invoking the NVIDIA Container Runtime Hook directly (e.g. specifying the docker --gpus flag) is not supported. Please use the NVIDIA Container Runtime (e.g. specify the --runtime=nvidia flag) instead
```

因此需要將 `cuPHY-CP/container/run_aerial.sh` 中的 GPU flag 改為使用 `--runtime=nvidia`：

```bash
# 修改前
GPU_FLAG="--gpus all"

# 修改後
# GPU_FLAG="--gpus all"
GPU_FLAG="--runtime=nvidia -e NVIDIA_VISIBLE_DEVICES=all"
```

### Step 6：驗證與執行

執行原始腳本：

```bash
./cuPHY-CP/container/run_aerial.sh
```

成功進入容器後應看到：

```
aerial@c_aerial_admin:/opt/nvidia/cuBB$
```

---

## 補充說明

### 為何 DGX Spark 特別會遇到此問題

- DGX Spark 搭載 GB10 GPU，驅動版本 590.x 在安裝時會自動生成 CDI spec
- CDI spec 將 MPS 二進位檔列為掛載項目，但在某些 DGX Spark 系統配置下，容器 rootfs 內無法解析這些路徑
- `run_aerial.sh` 已針對 DGX Spark 跳過 gdrdrv 檢查（`AERIAL_CHECK_GDRDRV=0`），但未處理 CDI mode 問題

### 若 CDI spec 被系統更新覆蓋

系統更新驅動後，`/var/run/cdi/nvidia.yaml` 可能被重新生成並帶回 MPS 條目，需重新執行 Step 1。

可建立 post-install hook 或定期檢查：

```bash
grep -c "nvidia-cuda-mps" /var/run/cdi/nvidia.yaml && \
    sudo sed -i '/nvidia-cuda-mps/d' /var/run/cdi/nvidia.yaml
```

---

## 修改檔案清單

| 檔案 | 修改內容 |
|------|----------|
| `/var/run/cdi/nvidia.yaml` | 移除 `nvidia-cuda-mps-control` 及 `nvidia-cuda-mps-server` 掛載條目 |
| `/etc/nvidia-container-runtime/config.toml` | `mode = "auto"` → `mode = "legacy"` |
| `/etc/docker/daemon.json` | 新增 `"default-runtime": "nvidia"` |
| [run_aerial.sh](file:///home/admin/aerial-cuda-accelerated-ran/cuPHY-CP/container/run_aerial.sh) | `GPU_FLAG="--gpus all"` → `GPU_FLAG="--runtime=nvidia -e NVIDIA_VISIBLE_DEVICES=all"` |



---
admin@spark-d957:~/aerial-cuda-accelerated-ran$ ./cuPHY-CP/container/run_aerial.sh
/home/admin/aerial-cuda-accelerated-ran/cuPHY-CP/container/run_aerial.sh starting...
Start container instance at bash prompt
server_id: NVIDIA-P4242
Skipping gdrdrv check for DGX Spark
Command: /bin/bash
TTY detected - running in interactive mode
26-1-cubb: Pulling from nvidia/aerial/aerial-cuda-accelerated-ran
Digest: sha256:5773d96a2188372cb39e57e85148501836cf397bcebeb80861a30674e0ccaa0c
Status: Image is up to date for nvcr.io/nvidia/aerial/aerial-cuda-accelerated-ran:26-1-cubb
nvcr.io/nvidia/aerial/aerial-cuda-accelerated-ran:26-1-cubb

==========
== CUDA ==
==========

CUDA Version 13.1.1

Container image Copyright (c) 2016-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.

This container image and its contents are governed by the NVIDIA Deep Learning Container License.
By pulling and using the container, you accept the terms and conditions of this license:
https://developer.nvidia.com/ngc/nvidia-deep-learning-container-license

A copy of this license is made available in this container at /NGC-DL-CONTAINER-LICENSE for your convenience.

fixuid: fixuid should only ever be used on development systems. DO NOT USE IN PRODUCTION
fixuid: runtime UID '1000' already matches container user 'aerial' UID
fixuid: runtime GID '1000' already matches container group 'aerial' GID
aerial@c_aerial_admin:/opt/nvidia/cuBB$ cd /opt/nvidia/cuBB
aerial@c_aerial_admin:/opt/nvidia/cuBB$ sudo -E ./run_l1.sh
Cannot find MPS control daemon process
Started cuphycontroller on CPU core 3
09:33:23.531450 CON phy_init 0 [CTL.SCF] Config file: /opt/nvidia/cuBB/cuPHY-CP/cuphycontroller/config/cuphycontroller_P5G_WNC_DGX.yaml
09:33:23.532387 CON phy_init 0 [CTL.SCF] low_priority_core=10
09:33:23.532425 CON phy_init 0 [APP.CONFIG] Current TAI offset: 0s
09:33:23.532748 CON phy_init 0 [NVLOG.CPP] Using /opt/nvidia/cuBB/cuPHY/nvlog/config/nvlog_config.yaml for nvlog configuration
09:33:23.535191 CON 113 0 [NVLOG.EXIT_HANDLER] [exit_watchdog] thread started on core 10
09:33:23.544480 CON phy_init 0 [NVLOG.CPP] Output log file path /tmp/phy.log
Aerial metrics backend address: 127.0.0.1:8081
09:33:23.556436 WRN phy_init 0 [CTL.YAML] cuphycontroller config. yaml does not have use_gc_workqueues key (experimental feature); defaulting to 0.
09:33:23.556535 WRN phy_init 0 [CTL.YAML] YAML invalid key: cupti_enable_tracing Using default value of 0 to CUPTI_ENABLE_TRACING
09:33:23.556560 WRN phy_init 0 [CTL.YAML] YAML invalid key: cupti_buffer_size Using default value of 2GB for cupti_buffer_size
09:33:23.556566 WRN phy_init 0 [CTL.YAML] YAML invalid key: cupti_num_buffers Using default value of 2 for cupti_num_buffers
09:33:23.556579 WRN phy_init 0 [CTL.YAML] YAML invalid key: ul_warmup_frame_count Using default value of 8 to YAML_PARAM_UL_WARMUP_FRAME_COUNT
09:33:23.556590 WRN phy_init 0 [CTL.YAML] YAML invalid key: enable_dl_core_affinity Using default value of 1 to YAML_PARAM_ENABLE_DL_CORE_AFFINITY
09:33:23.556596 WRN phy_init 0 [CTL.YAML] YAML invalid key: dlc_core_packing_scheme Using default value of 0 to YAML_PARAM_DLC_CORE_PACKING_SCHEME
09:33:23.556614 WRN phy_init 0 [CTL.YAML] YAML invalid key: pusch_weighted_average_cfo Using default value of 0 to PUSCH-WEIGHTED-AVERAGE-CFO
09:33:23.556652 WRN phy_init 0 [CTL.YAML] YAML invalid key: enable_tx_notification Using default value of 0 for YAML_PARAM_ENABLE_TX_NOTIFICATION
09:33:23.556777 WRN phy_init 0 [CTL.YAML] YAML invalid key: pusch_nMaxTbPerNode Using default value of 32 to PUSCH-N-MAX-TB-PER-NODE
09:33:23.556800 CON phy_init 0 [CTL.YAML] cell_id 1 nic_index :0
09:33:23.556871 CON phy_init 0 [CTL.YAML] ta4_min_ns_srs/ta4_max_ns_srs not set in config file, using default ta4_min_ns_srs = 621 us and ta4_max_ns_srs = 1831 us
09:33:23.556939 CON phy_init 0 [CTL.YAML] Num Slots: 8
09:33:23.556940 CON phy_init 0 [CTL.YAML] Enable UL cuPHY Graphs: 1
09:33:23.556940 CON phy_init 0 [CTL.YAML] Enable DL cuPHY Graphs: 1
09:33:23.556940 CON phy_init 0 [CTL.YAML] Accurate TX scheduling clock resolution (ns): 500
09:33:23.556940 CON phy_init 0 [CTL.YAML] DPDK core: 10
09:33:23.556940 CON phy_init 0 [CTL.YAML] Prometheus core: -1
09:33:23.556940 CON phy_init 0 [CTL.YAML] UL cores: 
09:33:23.556941 CON phy_init 0 [CTL.YAML]       - 5
09:33:23.556941 CON phy_init 0 [CTL.YAML]       - 6
09:33:23.556941 CON phy_init 0 [CTL.YAML] DL cores: 
09:33:23.556941 CON phy_init 0 [CTL.YAML]       - 7
09:33:23.556941 CON phy_init 0 [CTL.YAML]       - 8
09:33:23.556941 CON phy_init 0 [CTL.YAML]       - 9
09:33:23.556941 CON phy_init 0 [CTL.YAML] Debug worker: -1
09:33:23.556941 CON phy_init 0 [CTL.YAML] Data Lake core: -1
09:33:23.556942 CON phy_init 0 [CTL.YAML] SRS starting Section ID: 3072
09:33:23.556942 CON phy_init 0 [CTL.YAML] PRACH starting Section ID: 2048
09:33:23.556942 CON phy_init 0 [CTL.YAML] USE GREEN CONTEXTS: 0
09:33:23.556942 CON phy_init 0 [CTL.YAML] USE GC WORKQUEUES: 0
09:33:23.556942 CON phy_init 0 [CTL.YAML] USE BATCHED MEMCPY: 1
09:33:23.556942 CON phy_init 0 [CTL.YAML] MPS SM PUSCH: 40
09:33:23.556942 CON phy_init 0 [CTL.YAML] MPS SM PUCCH: 4
09:33:23.556942 CON phy_init 0 [CTL.YAML] MPS SM PRACH: 4
09:33:23.556942 CON phy_init 0 [CTL.YAML] MPS SM UL ORDER: 12
09:33:23.556944 CON phy_init 0 [CTL.YAML] MPS SM PDSCH: 46
09:33:23.556944 CON phy_init 0 [CTL.YAML] MPS SM PDCCH: 12
09:33:23.556944 CON phy_init 0 [CTL.YAML] MPS SM PBCH: 4
09:33:23.556944 CON phy_init 0 [CTL.YAML] MPS SM GPU_COMMS: 16
09:33:23.556945 CON phy_init 0 [CTL.YAML] MPS SM SRS: 16
09:33:23.556945 CON phy_init 0 [CTL.YAML] UL Order Kernel Mode: 0
09:33:23.556945 CON phy_init 0 [CTL.YAML] PDSCH fallback: 0
09:33:23.556945 CON phy_init 0 [CTL.YAML] Massive MIMO enable: 0
09:33:23.556945 CON phy_init 0 [CTL.YAML] mMIMO_enable feature 0
09:33:23.556945 CON phy_init 0 [CTL.YAML] Enable SRS : 1
09:33:23.556946 CON phy_init 0 [CTL.YAML] ul_order_timeout_gpu_log_enable: 1
09:33:23.556946 CON phy_init 0 [CTL.YAML] ue_mode: 0
09:33:23.556946 CON phy_init 0 [CTL.YAML] Aggr Obj Non-availability threshold: 5
09:33:23.556946 CON phy_init 0 [CTL.YAML] sendCPlane_timing_error_th_ns: 0
09:33:23.556946 CON phy_init 0 [CTL.YAML] pusch_aggr_per_ctx: 3
09:33:23.556946 CON phy_init 0 [CTL.YAML] prach_aggr_per_ctx: 2
09:33:23.556946 CON phy_init 0 [CTL.YAML] pucch_aggr_per_ctx: 4
09:33:23.556946 CON phy_init 0 [CTL.YAML] srs_aggr_per_ctx: 3
09:33:23.556946 CON phy_init 0 [CTL.YAML] max_harq_pools: 384
09:33:23.556946 CON phy_init 0 [CTL.YAML] max_harq_tx_count_bundled: 10
09:33:23.556946 CON phy_init 0 [CTL.YAML] max_harq_tx_count_non_bundled: 4
09:33:23.556946 CON phy_init 0 [CTL.YAML] ul_input_buffer_per_cell: 10
09:33:23.556946 CON phy_init 0 [CTL.YAML] ul_input_buffer_per_cell_srs: 6
09:33:23.556946 CON phy_init 0 [CTL.YAML] max_ru_unhealthy_ul_slots: 100
09:33:23.556946 CON phy_init 0 [CTL.YAML] srs_chest_algo_type: 0
09:33:23.556947 CON phy_init 0 [CTL.YAML] srs_chest_tol2_normalization_algo_type: 1
09:33:23.556947 CON phy_init 0 [CTL.YAML] srs_chest_tol2_constant_scaler: 32768
09:33:23.556947 CON phy_init 0 [CTL.YAML] bfw_power_normalization_alg_selector: 1
09:33:23.556947 CON phy_init 0 [CTL.YAML] bfw_beta_prescaler: 16384
09:33:23.556947 CON phy_init 0 [CTL.YAML] total_num_srs_chest_buffers: 6144
09:33:23.556947 CON phy_init 0 [CTL.YAML] send_static_bfw_wt_all_cplane: 1
09:33:23.556947 CON phy_init 0 [CTL.YAML] ul_pcap_capture_enable: 0
09:33:23.556947 CON phy_init 0 [CTL.YAML] ul_pcap_capture_thread_cpu_affinity: 255
09:33:23.556948 CON phy_init 0 [CTL.YAML] ul_pcap_capture_thread_sched_priority: 0
09:33:23.556948 CON phy_init 0 [CTL.YAML] pcap_logger_ul_cplane_enable: 0
09:33:23.556948 CON phy_init 0 [CTL.YAML] pcap_logger_dl_cplane_enable: 0
09:33:23.556948 CON phy_init 0 [CTL.YAML] pcap_logger_thread_cpu_affinity: 10
09:33:23.556948 CON phy_init 0 [CTL.YAML] pcap_logger_thread_sched_prio: 95
09:33:23.556948 CON phy_init 0 [CTL.YAML] pcap_logger_file_save_dir: ./
09:33:23.556948 CON phy_init 0 [CTL.YAML] static_beam_id_start: 1
09:33:23.556949 CON phy_init 0 [CTL.YAML] static_beam_id_end: 16527
09:33:23.556949 CON phy_init 0 [CTL.YAML] dynamic_beam_id_start: 16528
09:33:23.556949 CON phy_init 0 [CTL.YAML] dynamic_beam_id_end: 32767
09:33:23.556949 CON phy_init 0 [CTL.YAML] ul_order_timeout_gpu_log_enable: 1
09:33:23.556949 CON phy_init 0 [CTL.YAML] pusch_workCancelMode: 2
09:33:23.556949 CON phy_init 0 [CTL.YAML] GPU-initiated comms DL: 1
09:33:23.556949 CON phy_init 0 [CTL.YAML] GPU-initiated comms (via CPU): 1
09:33:23.556949 CON phy_init 0 [CTL.YAML] CPU-initiated comms : 0
09:33:23.556949 CON phy_init 0 [CTL.YAML] Cell group: 1
09:33:23.556950 CON phy_init 0 [CTL.YAML] Cell group num: 1
09:33:23.556950 CON phy_init 0 [CTL.YAML] puxchPolarDcdrListSz: 8
09:33:23.556950 CON phy_init 0 [CTL.YAML] split_ul_cuda_streams: 0
09:33:23.556950 CON phy_init 0 [CTL.YAML] serialize_pucch_pusch: 0
09:33:23.556950 CON phy_init 0 [CTL.YAML] bfw_c_plane_chaining_mode: 0
09:33:23.556951 CON phy_init 0 [CTL.YAML] fh_stats_dump_cpu_core: -1
09:33:23.556951 CON phy_init 0 [CTL.YAML] fix_beta_dl: 0
09:33:23.556951 CON phy_init 0 [CTL.YAML] pdump_client_thread: -1
09:33:23.556951 CON phy_init 0 [CTL.YAML] profiler_sec: 0
09:33:23.556951 CON phy_init 0 [CTL.YAML] datalake_address: localhost
09:33:23.556951 CON phy_init 0 [CTL.YAML] dpdk_file_prefix: cuphycontroller
09:33:23.556951 CON phy_init 0 [CTL.YAML] puschrxChestFactorySettingsFilename: 
09:33:23.556951 CON phy_init 0 [CTL.YAML] accu_tx_sched_disable: 0
09:33:23.556952 CON phy_init 0 [CTL.YAML] cplane_disable: 0
09:33:23.556952 CON phy_init 0 [CTL.YAML] disable_empw: 0
09:33:23.556952 CON phy_init 0 [CTL.YAML] dlc_alloc_cplane_bfw_txq: 0
09:33:23.556952 CON phy_init 0 [CTL.YAML] dlc_bfw_enable_divide_per_cell: 0
09:33:23.556952 CON phy_init 0 [CTL.YAML] dpdk_verbose_logs: 0
09:33:23.556952 CON phy_init 0 [CTL.YAML] enable_cpu_task_tracing: 0
09:33:23.556953 CON phy_init 0 [CTL.YAML] enable_dl_cqe_tracing: 0
09:33:23.556953 CON phy_init 0 [CTL.YAML] enable_l1_param_sanity_check: 0
09:33:23.556953 CON phy_init 0 [CTL.YAML] enable_ok_tb: 0
09:33:23.556953 CON phy_init 0 [CTL.YAML] enable_prepare_tracing: 0
09:33:23.556953 CON phy_init 0 [CTL.YAML] cupti_enable_tracing: 0
09:33:23.556953 CON phy_init 0 [CTL.YAML] cupti_buffer_size: 2147483648
09:33:23.556953 CON phy_init 0 [CTL.YAML] cupti_num_buffers: 2
09:33:23.556954 CON phy_init 0 [CTL.YAML] mCh_segment_proc_enable: 0
09:33:23.556954 CON phy_init 0 [CTL.YAML] pmu_metrics: 0
09:33:23.556954 CON phy_init 0 [CTL.YAML] puschCfo: 1
09:33:23.556954 CON phy_init 0 [CTL.YAML] puschDftSOfdm: 0
09:33:23.556954 CON phy_init 0 [CTL.YAML] puschEnablePerPrgChEst: 0
09:33:23.556954 CON phy_init 0 [CTL.YAML] puschRssi: 1
09:33:23.556955 CON phy_init 0 [CTL.YAML] puschSelectChEstAlgo: 1
09:33:23.556955 CON phy_init 0 [CTL.YAML] puschSelectEqCoeffAlgo: 3
09:33:23.556955 CON phy_init 0 [CTL.YAML] puschSinr: 2
09:33:23.556955 CON phy_init 0 [CTL.YAML] puschTbSizeCheck: 1
09:33:23.556955 CON phy_init 0 [CTL.YAML] puschTdi: 1
09:33:23.556955 CON phy_init 0 [CTL.YAML] puschTo: 1
09:33:23.556955 CON phy_init 0 [CTL.YAML] pusch_deviceGraphLaunchEn: 1
09:33:23.556955 CON phy_init 0 [CTL.YAML] ulc_alloc_cplane_bfw_txq: 1
09:33:23.556956 CON phy_init 0 [CTL.YAML] ulc_bfw_enable_divide_per_cell: 0
09:33:23.556956 CON phy_init 0 [CTL.YAML] ul_rx_pkt_tracing_level: 0
09:33:23.556956 CON phy_init 0 [CTL.YAML] ul_rx_pkt_tracing_level_srs: 0
09:33:23.556956 CON phy_init 0 [CTL.YAML] ul_warmup_frame_count: 8
09:33:23.556956 CON phy_init 0 [CTL.YAML] forcedNumCsi2Bits: 0
09:33:23.556956 CON phy_init 0 [CTL.YAML] pusch_waitTimeOutPostEarlyHarqUs: 3000
09:33:23.556956 CON phy_init 0 [CTL.YAML] pusch_waitTimeOutPreEarlyHarqUs: 3000
09:33:23.556958 CON phy_init 0 [CTL.YAML] validation: 0
09:33:23.556958 CON phy_init 0 [CTL.YAML] cqe_trace_slot_mask: 196800
09:33:23.556958 CON phy_init 0 [CTL.YAML] datalake_samples: 1000000
09:33:23.556959 CON phy_init 0 [CTL.YAML] num_ok_tb_slot: 0
09:33:23.556959 CON phy_init 0 [CTL.YAML] pusch_nMaxLdpcHetConfigs: 32
09:33:23.556959 CON phy_init 0 [CTL.YAML] pusch_nMaxTbPerNode: 32
09:33:23.556959 CON phy_init 0 [CTL.YAML] sendCPlane_dlbfw_backoff_th_ns: 300000
09:33:23.556959 CON phy_init 0 [CTL.YAML] sendCPlane_ulbfw_backoff_th_ns: 300000
09:33:23.556960 CON phy_init 0 [CTL.YAML] ul_order_max_rx_pkts: 512
09:33:23.556960 CON phy_init 0 [CTL.YAML] ul_order_rx_pkts_timeout_ns: 100000
09:33:23.556960 CON phy_init 0 [CTL.YAML] ul_order_timeout_cpu_ns: 8000000
09:33:23.556960 CON phy_init 0 [CTL.YAML] ul_order_timeout_gpu_ns: 3000000
09:33:23.556960 CON phy_init 0 [CTL.YAML] ul_order_timeout_gpu_srs_ns: 5200000
09:33:23.556961 CON phy_init 0 [CTL.YAML] ul_order_timeout_log_interval_ns: 1000000000
09:33:23.556961 CON phy_init 0 [CTL.YAML] ul_srs_aggr3_task_launch_offset_ns: 500000
09:33:23.556961 CON phy_init 0 [CTL.YAML] workers_sched_priority: 95
09:33:23.556961 CON phy_init 0 [CTL.YAML] cqe_trace_cell_mask: 1
09:33:23.556962 CON phy_init 0 [CTL.YAML] Number of Cell Configs: 1
09:33:23.556962 CON phy_init 0 [CTL.YAML] L2Adapter config file: /opt/nvidia/cuBB/cuPHY-CP/cuphycontroller/config/l2_adapter_config_P5G_DGX.yaml
09:33:23.556962 CON phy_init 0 [CTL.YAML] Cell name: O-RU 0
09:33:23.556962 CON phy_init 0 [CTL.YAML]       MU: 1
09:33:23.556962 CON phy_init 0 [CTL.YAML]       ID: 1
09:33:23.556962 CON phy_init 0 [CTL.YAML] Number of MPlane Configs: 1
09:33:23.556963 CON phy_init 0 [CTL.YAML]       Mplane ID: 1
09:33:23.556963 CON phy_init 0 [CTL.YAML]       VLAN ID: 564
09:33:23.556963 CON phy_init 0 [CTL.YAML]       Source Eth Address: 00:00:00:00:00:00
09:33:23.556963 CON phy_init 0 [CTL.YAML]       Destination Eth Address: e8:c7:cf:ac:58:20
09:33:23.556963 CON phy_init 0 [CTL.YAML]       NIC port: 0000:01:00.0
09:33:23.556963 CON phy_init 0 [CTL.YAML]       RU Type: 1
09:33:23.556963 CON phy_init 0 [CTL.YAML]       U-plane TXQs: 1
09:33:23.556964 CON phy_init 0 [CTL.YAML]       DL compression method: 1
09:33:23.556964 CON phy_init 0 [CTL.YAML]       DL iq bit width: 9
09:33:23.556964 CON phy_init 0 [CTL.YAML]       UL compression method: 1
09:33:23.556964 CON phy_init 0 [CTL.YAML]       UL iq bit width: 9
09:33:23.556964 CON phy_init 0 [CTL.YAML] 
09:33:23.556964 CON phy_init 0 [CTL.YAML]       Flow list SSB/PBCH: 
09:33:23.556965 CON phy_init 0 [CTL.YAML]               0
09:33:23.556965 CON phy_init 0 [CTL.YAML]               1
09:33:23.556965 CON phy_init 0 [CTL.YAML]               2
09:33:23.556965 CON phy_init 0 [CTL.YAML]               3
09:33:23.556965 CON phy_init 0 [CTL.YAML]       Flow list PDCCH: 
09:33:23.556965 CON phy_init 0 [CTL.YAML]               0
09:33:23.556965 CON phy_init 0 [CTL.YAML]               1
09:33:23.556965 CON phy_init 0 [CTL.YAML]               2
09:33:23.556966 CON phy_init 0 [CTL.YAML]               3
09:33:23.556966 CON phy_init 0 [CTL.YAML]       Flow list PDSCH: 
09:33:23.556966 CON phy_init 0 [CTL.YAML]               0
09:33:23.556966 CON phy_init 0 [CTL.YAML]               1
09:33:23.556966 CON phy_init 0 [CTL.YAML]               2
09:33:23.556966 CON phy_init 0 [CTL.YAML]               3
09:33:23.556966 CON phy_init 0 [CTL.YAML]       Flow list CSIRS: 
09:33:23.556967 CON phy_init 0 [CTL.YAML]               0
09:33:23.556967 CON phy_init 0 [CTL.YAML]               1
09:33:23.556967 CON phy_init 0 [CTL.YAML]               2
09:33:23.556967 CON phy_init 0 [CTL.YAML]               3
09:33:23.556967 CON phy_init 0 [CTL.YAML]       Flow list PUSCH: 
09:33:23.556967 CON phy_init 0 [CTL.YAML]               0
09:33:23.556967 CON phy_init 0 [CTL.YAML]               1
09:33:23.556967 CON phy_init 0 [CTL.YAML]               2
09:33:23.556967 CON phy_init 0 [CTL.YAML]               3
09:33:23.556968 CON phy_init 0 [CTL.YAML]       Flow list PUCCH: 
09:33:23.556968 CON phy_init 0 [CTL.YAML]               0
09:33:23.556968 CON phy_init 0 [CTL.YAML]               1
09:33:23.556968 CON phy_init 0 [CTL.YAML]               2
09:33:23.556968 CON phy_init 0 [CTL.YAML]               3
09:33:23.556968 CON phy_init 0 [CTL.YAML]       Flow list SRS: 
09:33:23.556968 CON phy_init 0 [CTL.YAML]               8
09:33:23.556968 CON phy_init 0 [CTL.YAML]               9
09:33:23.556969 CON phy_init 0 [CTL.YAML]               10
09:33:23.556969 CON phy_init 0 [CTL.YAML]               11
09:33:23.556969 CON phy_init 0 [CTL.YAML]       Flow list PRACH: 
09:33:23.556969 CON phy_init 0 [CTL.YAML]               4
09:33:23.556969 CON phy_init 0 [CTL.YAML]               5
09:33:23.556969 CON phy_init 0 [CTL.YAML]               6
09:33:23.556969 CON phy_init 0 [CTL.YAML]               7
09:33:23.556970 CON phy_init 0 [CTL.YAML]       PUSCH TV: /opt/nvidia/cuBB/testVectors/cuPhyChEstCoeffs.h5
09:33:23.556970 CON phy_init 0 [CTL.YAML]       SRS TV: /opt/nvidia/cuBB/testVectors/cuPhyChEstCoeffs.h5
09:33:23.556970 CON phy_init 0 [CTL.YAML]       Section_3 time offset: 58369
09:33:23.556970 CON phy_init 0 [CTL.YAML]       nMaxRxAnt: 4
09:33:23.556970 CON phy_init 0 [CTL.YAML]       PUSCH PRBs Stride: 273
09:33:23.556970 CON phy_init 0 [CTL.YAML]       PRACH PRBs Stride: 12
09:33:23.556970 CON phy_init 0 [CTL.YAML]       SRS PRBs Stride: 273
09:33:23.556971 CON phy_init 0 [CTL.YAML]       PUSCH nMaxPrb: 273
09:33:23.556971 CON phy_init 0 [CTL.YAML]       PUSCH nMaxRx: 4
09:33:23.556971 CON phy_init 0 [CTL.YAML]       UL Gain Calibration: 78.68
09:33:23.556971 CON phy_init 0 [CTL.YAML]       Lower guard bw: 845
09:33:23.833540 CON phy_init 0 [CTL.SCF] Network interface for PCIe address 0000:01:00.0 : aerial02
09:33:23.833616 CON phy_init 0 [APP.UTILS]           PHC clock: 1782466403.139867700
09:33:23.833631 CON phy_init 0 [APP.UTILS]           CLOCK_TAI: 1782466403.833622617
09:33:23.833631 CON phy_init 0 [APP.UTILS]      CLOCK_REALTIME: 1782466403.833622585
09:33:23.833631 CON phy_init 0 [APP.UTILS] TAI/REALTIME offset: 0 seconds
09:33:23.836854 CON phy_init 0 [DRV.CTX] CUDA_DEVICE_MAX_CONNECTIONS 8
09:33:23.836860 CON phy_init 0 [DRV.CTX] CUDA runtime version from cudaRuntimeGetVersion(): 13010
09:33:23.836861 CON phy_init 0 [DRV.CTX] CUDA driver version from cudaDriverGetVersion(): 13010
09:33:23.836862 CON phy_init 0 [DRV.CTX] use_green_contexts 0
09:33:23.836862 CON phy_init 0 [DRV.CTX] use_gc_workqueues 0
09:33:24.065773 CON phy_init 0 [DRV.CTX] PUSCH MPS context with max. SM count of 40.
09:33:24.315902 CON phy_init 0 [DRV.CTX] PUCCH MPS context with max. SM count of 4.
09:33:24.560424 CON phy_init 0 [DRV.CTX] PRACH MPS context with max. SM count of 4.
09:33:24.820771 CON phy_init 0 [DRV.CTX] PDSCH MPS context with max. SM count of 46.
09:33:25.096603 CON phy_init 0 [DRV.CTX] DL Ctrl MPS context with max. SM count of 24.
09:33:25.379591 CON phy_init 0 [DRV.CTX] UL Order MPS context with max. SM count of 12.
09:33:25.670852 CON phy_init 0 [DRV.CTX] SRS MPS context with max. SM count of 16.
EAL: Detected CPU lcores: 20
EAL: Detected NUMA nodes: 1
EAL: Detected shared linkage of DPDK
EAL: Multi-process socket /var/run/dpdk/cuphycontroller/mp_socket
EAL: Selected IOVA mode 'VA'
EAL: No free 524288 kB hugepages reported on node 0
EAL: FATAL: Cannot get hugepage information.
EAL: Cannot get hugepage information.
09:33:26.041174 FATAL exit: Thread [phy_drv_init] on core 10 file /opt/nvidia/cuBB/cuPHY-CP/cuphycontroller/examples/cuphycontroller_scf.cpp line 468: additional info: NULL
Stack trace (most recent call last):
#7    Object "/usr/lib/aarch64-linux-gnu/ld-linux-aarch64.so.1", at 0xffffffffffffffff, in 
#6    Object "/opt/nvidia/cuBB/build.aarch64/cuPHY-CP/cuphycontroller/examples/cuphycontroller_scf", at 0x420baf, in _start
#5    Object "/usr/lib/aarch64-linux-gnu/libc.so.6", at 0xe439e6b774d7, in __libc_start_main
#4    Object "/usr/lib/aarch64-linux-gnu/libc.so.6", at 0xe439e6b773ff, in 
#3    Object "/opt/nvidia/cuBB/build.aarch64/cuPHY-CP/cuphycontroller/examples/cuphycontroller_scf", at 0x413ddf, in main
#2    Object "/opt/nvidia/cuBB/build.aarch64/cuPHY/nvlog/libnvlog.so", at 0xe43a19f82b57, in exit_handler::test_trigger_exit(char const*, int, char const*)
#1    Object "/opt/nvidia/cuBB/build.aarch64/cuPHY-CP/cuphycontroller/examples/cuphycontroller_scf", at 0x4214c3, in nvlog_exit_handler()
#0    Source "/opt/nvidia/cuBB/cuPHY-CP/cuphydriver/src/common/cuphydriver_api.cpp", line 3331, in l1_exit_handler
       3328:     //PhyDriver initialization failure
       3329:     if(l1_getPhydriverHandle() == nullptr)
       3330:     {
      >3331:         AERIAL_PRINT_BACKTRACE(32ULL);
       3332:         NVLOGW_FMT(TAG, "L1 exit handler: PhyDriver handle is null, cleanup may be incomplete");
       3333:         return;
       3334:     }
09:33:25.968009 CON phy_init 0 [DRV.CTX] GPU comm MPS context with max. SM count of 16.
09:33:26.022352 ERR phy_init 0 [AERIAL_ORAN_FH_EVENT] [FH.LIB] Exception! DPDK initialization failed: Success
09:33:26.041138 ERR phy_init 0 [AERIAL_CUPHYDRV_API_EVENT] [DRV.EXCP] /opt/nvidia/cuBB/cuPHY-CP/cuphydriver/src/common/cuphydriver_api.cpp l1_init line 165 exception: aerial_fh::open returned error
09:33:26.041157 ERR phy_init 0 [AERIAL_CUPHYDRV_API_EVENT] [CTL.DRV] Error l1_init
09:33:26.041157 ERR phy_init 0 [AERIAL_CUPHYDRV_API_EVENT] [CTL.SCF] pc_init_phydriver error -1
09:33:26.041189 FAT phy_init 0 [AERIAL_SYSTEM_API_EVENT] [NVLOG.EXIT_HANDLER] FATAL exit: Thread [phy_drv_init] on core 10 file /opt/nvidia/cuBB/cuPHY-CP/cuphycontroller/examples/cuphycontroller_scf.cpp line 468: additional info: NULL
09:33:26.041204 CON phy_init 0 [CTL.SCF] nvlog_exit_handler: nvlog exit handler called in thread [phy_drv_init]
09:33:26.041213 CON phy_init 0 [DRV.API] Triggering L1 exit handler
09:33:26.105452 WRN phy_init 0 [DRV.API] L1 exit handler: PhyDriver handle is null, cleanup may be incomplete
09:33:26.105454 CON phy_init 0 [NVLOG.EXIT_HANDLER] test_trigger_exit: exit_cb called
09:33:26.105768 CON 113 0 [NVLOG.EXIT_HANDLER] [exit_watchdog] thread received exiting notification, wait for 5 seconds to force exit
09:33:31.106240 CON 113 0 [NVLOG.EXIT_HANDLER] [exit_watchdog] thread closed FMT log after and force exit
09:33:31.106278 CON 113 0 [NVLOG.CPP] nvlog_fmtlog_close: closing FMT log from [exit_watchdog] thread on core 10, last file path is /tmp/phy.log
Exiting bg_fmtlog_collector - log queue ever was full: 0
nvlog_fmtlog_close: FMT log closed
09:33:31.133796 EXIT from watchdog after 5 seconds waiting
aerial@c_aerial_admin:/opt/nvidia/cuBB$ 


解決步驟
根據 

official-DGX-setup.md
 的最佳實踐，DGX Spark 系統建議配置 1G 的大頁記憶體（數量為 24）。

您可以透過以下兩種方式之一來分配大頁記憶體：

方案一：暫時（動態）配置大頁（免重啟，可快速測試）
在**主機（Host）**的終端機執行以下指令，直接動態分配大頁記憶體給系統（此方法在系統重啟後會失效）：

若目前系統預設大頁為 512MB (524288 kB)：

bash
echo 24 | sudo tee /sys/devices/system/node/node0/hugepages/hugepages-524288kB/nr_hugepages
若欲嘗試分配 1GB (1048576 kB) 大頁：

bash
echo 24 | sudo tee /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages
方案二：永久配置大頁（需要重啟主機，推薦）
編輯主機的 /etc/default/grub 檔案：
bash
sudo nano /etc/default/grub
在 GRUB_CMDLINE_LINUX 參數中，加入以下大頁配置：
text
default_hugepagesz=1G hugepagesz=1G hugepages=24
更新 GRUB 設定並重啟主機：
bash
sudo update-grub
sudo reboot
驗證方法
主機啟動後，在主機上執行以下指令：

bash
grep -i huge /proc/meminfo
若看到 HugePages_Total 與 HugePages_Free 皆為 24（且 Hugepagesize 為 1048576 kB），即代表大頁記憶體已成功啟用。此時再啟動容器執行 ./run_l1.sh 就不會再遇到這個錯誤。


---


這個錯誤表示 L1 驅動（DPDK）檢測到實體網卡 0000:01:00.0（對應 aerial02 網口）沒有任何實體連線。

text
NIC 0000:01:00.0 has no physical Ethernet link (cable unplugged or peer port down)
從您主機的網口現狀來看：

aerial00, aerial01, aerial02, aerial03 的狀態皆為 NO-CARRIER (DOWN)。
透過 Mellanox 診斷工具 sudo mlxlink -d 0000:01:00.0 可以確認：
State: Disable
Physical state: ETH_AN_FSM_ENABLE
Recommendation: Cable is unplugged （光纖線/DAC 線未插上）
解決方法：
方案一：確認實體連線（適用於有對接 O-RU/測試儀的場景）
請檢查 DGX Spark 後方的 ConnectX-7 網卡，確保已插上光模組與光纖線（或 DAC 線），且線路的另一端（例如實體 O-RU、DU 或 Switch）已正常開機且網口處於啟用（Up）狀態。

方案二：啟用內部環回（Internal Loopback）模式（適用於單機開發與模擬測試）
如果您要在沒有實體連線的狀態下進行本機測試，可以使用 Mellanox 工具將網卡強制設定為內部環回（Loopback）模式。請在**主機（Host）**端執行以下指令：

先將網卡 Port 狀態設為 Down（ConnectX-7 在 Polling 狀態下無法更改 Loopback 模式）：
bash
sudo mlxlink -d 0000:01:00.0 --port_state DN
設定大頁內部的環回模式為 PH (PHY loopback)：
bash
sudo mlxlink -d 0000:01:00.0 --loopback PH
重新將網卡 Port 狀態設為 Up：
bash
sudo mlxlink -d 0000:01:00.0 --port_state UP
設定完成後，再次執行 sudo mlxlink -d 0000:01:00.0 檢查，如果 Link 顯示為 Up 且 Loopback 顯示 PH，即可重新進入容器並執行 ./run_l1.sh！