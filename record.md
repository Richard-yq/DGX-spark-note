Docker Pull 失敗： 指令腳本 run_aerial.sh 嘗試從 Docker Hub 下拉 aerial/aerial-cuda-accelerated-ran:dgx-spark 鏡像，但該儲存庫（Repository）在公開的 Docker Hub 上不存在（或者是私有倉庫，需要先進行 docker login），因此出現 pull access denied。

---

```
git clone --recurse-submodules https://github.com/NVIDIA/aerial-cuda-accelerated-ran.git ~/aerial-cuda-accelerated-ran
cd ~/aerial-cuda-accelerated-ran/cubb_scripts/install/
sudo apt install make tmux git-lfs && git lfs pull
tmux new-session -s "aerial-cuda-accelerated-ran" -n "installation"
make all && sudo reboot
RU_MAC=48:21:0b:4b:93:8e make all
```

![alt text](image.png)

```
admin@spark-d957:~/aerial-cuda-accelerated-ran/cubb_scripts/install$ make all && sudo reboot
========================================
Installing drivers
========================================

IMPORTANT: Have you configured the following?
  - PTP settings
  - VLAN on RU and gNB interfaces
  - Peer MAC address on RU
  - Docker login if necessary
Without these the gNB will not deploy properly.

Continue with driver installation? (Y/n): 
```

## Clone Repository from Github
```
git clone https://github.com/NVIDIA/aerial-cuda-accelerated-ran.git --recurse-submodules
cd aerial-cuda-accelerated-ran

# Enable git LFS (if needed for large files)
git lfs install
git lfs pull
```
```
docker pull nvcr.io/nvidia/aerial/aerial-cuda-accelerated-ran:26-1-cubb
```

![alt text](image-1.png)

```
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
docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: failed to fulfil mount request: open /usr/bin/nvidia-cuda-mps-control: no such file or directory

Run 'docker run --help' for more information
```

[Reference](https://github.com/bmw-ece-ntust/ming-note/tree/ming-v1.0.0/notes/Handover):
```
2.2 RU：Old Pegatron
Plane / Interface	VLAN / ID	IP Address	MAC Address	Note
cuplane	103	N/A	48:21:0b:4b:93:8e	C/U-plane
fm1-mac9 SSH	N/A	192.168.109.9	48:21:0b:4b:93:8e	SSH management
mplane	104	192.168.110.9	48:21:0b:4b:93:8e	M-plane
splane	2	N/A	48:21:0b:4b:93:8e	S-plane
```


---

0626

經過系統檢查，您的設備環境部分符合規格要求。以下是各項規格的詳細比對結果與調整建議：

系統規格比對清單
項目	需求規格	系統目前狀態	是否符合	說明與建議
Host OS	DGX OS 7.2.3 or above	7.3.1	符合	核心系統版本符合要求。
Kernel	6.17.0-1014-nvidia	已安裝但未啟用
(目前開機核心: 6.14.0-1015-nvidia-64k)	需調整	6.17.0-1014-nvidia 已安裝，但目前開機進入的是 6.14-64k 核心。需要重新啟動並在 GRUB 中選取對應的核心。
CUDA Toolkit	13.1.1	13.0.88	不符合	目前安裝版本為 13.0。APT 軟體源中已有 13.1.1-1 或 13.1.2-1 可供安裝。
GB10 GPU Driver	590.48.01	590.48.01	符合	驅動程式版本完全一致。
CX7 NIC FW	28.47.1088	28.47.1088	符合	所有 Mellanox ConnectX-7 網卡韌體均為此版本。
DOCA OFED	25.10-1.7.1	25.10-1.7.1	符合	OFED 版本符合要求。
GDRCopy	N/A	N/A	符合	無需額外配置。
DOCA	3.2.1	N/A	符合	包含於 cuBB 容器中，主機端無需特別設定。
DPDK	22.11	N/A	符合	已整合於 Mellanox DOCA。
Linuxptp	4.2	未安裝	不符合	系統尚未安裝 linuxptp。且 APT 預設版本為 4.0，需編譯原始碼或透過安裝腳本安裝 4.2 版。
NV Container Toolkit	1.17.4 or above	1.19.0	符合	容器工具包版本符合要求。




如何調整不符合的項目
1. 切換至 6.17.0-1014-nvidia 核心
由於 linux-image-6.17.0-1014-nvidia 已經安裝在您的系統中，您只需重啟系統並在 GRUB 開機選單中選擇該核心。 若要將其設定為預設開機核心，可以參考以下步驟：

編輯 /etc/default/grub。
更新 GRUB 配置：
bash
sudo update-grub
重啟系統：
bash
sudo reboot
2. 升級 CUDA Toolkit 至 13.1.1
您可以使用 APT 升級至軟體源中的 13.1 版本：

bash
sudo apt-get update
sudo apt-get install -y cuda-toolkit-13-1
3. 安裝 Linuxptp 4.2
如 

official-DGX-setup.md
 中所述，為了支援雙埠 PTP，需要下載並編譯 v4.2 版本：

bash
# 確保移除舊版
sudo apt remove -y linuxptp
# 下載並編譯 4.2 版
wget https://github.com/richardcochran/linuxptp/archive/refs/tags/v4.2.tar.gz
tar -zxvf v4.2.tar.gz
cd linuxptp-4.2/
make
sudo make install