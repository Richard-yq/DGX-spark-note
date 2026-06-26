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

### Step 5：驗證

```bash
docker run --platform=linux/arm64 \
    --privileged --gpus all --rm \
    nvcr.io/nvidia/aerial/aerial-cuda-accelerated-ran:26-1-cubb \
    echo "ok"
```

接著執行原始腳本：

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