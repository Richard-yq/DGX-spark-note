# User Guide: Integrating NVIDIA Aerial L1 and OAI gNB (DGX Spark)

This document is an end-to-end SOP to build and run NVIDIA Aerial L1 with OAI gNB (Aerial FAPI split) using Docker Compose. It combines prerequisites, manual compilation steps, and the final Docker Compose integration, ensuring no setup steps are missed.

---

## 1. Environment Paths & Prerequisites

Define your base environment variables to keep paths consistent throughout the tutorial.

```bash
export AERIAL_HOME=/home/admin/aerial-cuda-accelerated-ran_dgx-spark_source
export OAI_HOME=/home/admin/Downloads/openairinterface5g-1-11
```

Ensure you have the Aerial SDK source tarball and the cloned OAI 5G source repository ready on your host machine.

---

## 2. Extract Aerial Source

*If you have already extracted the source, you can skip this step.*

```bash
cd ~
tar -xzf ~/Downloads/aerial-cuda-accelerated-ran_dgx-spark_source.tar.gz
cd "$AERIAL_HOME"
```

---

## 3. Launch Aerial Container and Build L1

Start the Aerial container and compile the L1 (cuBB) binaries.

```bash
cd "$AERIAL_HOME"
sudo ./cuPHY-CP/container/run_aerial.sh
```

**Inside the container:**
```bash
cd /opt/nvidia/cuBB
cmake -Bbuild -GNinja \
  -DCMAKE_TOOLCHAIN_FILE=cuPHY/cmake/toolchains/grace-cross \
  -DCMAKE_CUDA_ARCHITECTURES="121-real" \
  -DENABLE_20C=OFF \
  -DENABLE_CUMAC=OFF

cmake --build build -j"$(nproc --all)"
```

---

## 4. Test L1 Execution (Optional)

It is highly recommended to verify that L1 runs properly in isolation before integrating with OAI.

**Inside the container:**
```bash
cd /opt/nvidia/cuBB
sudo -E ./run-l1.sh
```

### L1 Success Markers
L1 is considered successfully started when the logs include:
- `gRPC Server listening on 0.0.0.0:50051`
- `====> PhyDriver initialized!`
- `cuPHYController configured for 1 cells`
- `====> cuPHYController initialized, L1 is ready!`
- `UlPhyDriver... initialized` and `DlPhyDriver... initialized`

*(Note: Stop the `run-l1.sh` script execution after verifying it works).*

---

## 5. Generate and Copy nvIPC Source Tarball

To allow OAI (L2) to communicate with Aerial L1, the `nvipc` package must be generated and transferred to the OAI repository.

**Inside the Aerial container:**
```bash
cd /opt/nvidia/cuBB/cuPHY-CP/gt_common_libs
./pack_nvipc.sh
ls -lh nvipc_src.*.tar.gz
```
*(Expected output example: `nvipc_src.2026.03.16.tar.gz`)*

**On the host machine:**
Identify the running Aerial container and copy the packaged tarball to the OAI repository:
```bash
# Check your container name
docker ps --format 'table {{.Names}}\t{{.Image}}'

# Assuming the container is named `c_aerial_root`
docker cp c_aerial_root:/opt/nvidia/cuBB/cuPHY-CP/gt_common_libs/nvipc_src.2026.03.16.tar.gz "$OAI_HOME"/
```

Verify the file transferred successfully:
```bash
ls -lh "$OAI_HOME"/nvipc_src.2026.03.16.tar.gz
```

---

## 6. Build OAI Docker Images

With `nvipc_src.*.tar.gz` placed in the root of `$OAI_HOME`, build the OAI Docker images.

```bash
cd "$OAI_HOME"
docker build . -f docker/Dockerfile.base.ubuntu --tag ran-base:latest
docker build . -f docker/Dockerfile.gNB.aerial.ubuntu --tag oai-gnb-aerial:latest
```

Verify images are built successfully:
```bash
docker images | grep -E 'ran-base|oai-gnb-aerial'
```

---

## 7. Configuration Adjustments

Before establishing the Docker Compose cluster, align multiple configurations across L1 and OAI.

### Step 7.1: Configure L1 (Aerial) Settings
Edit `$AERIAL_HOME/cuPHY-CP/cuphycontroller/config/cuphycontroller_P5G_WNC_DGX.yaml`.
Confirm that these values correctly match your RU/NIC hardware setup:
- `dst_mac_addr`
- `nic`
- `vlan`

### Step 7.2: Configure OAI gNB IP Settings
Edit your OAI gNB config (e.g., config under `ci-scripts/conf_files/` or `targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-vnf.sa.band78.273prb.aerial.conf`).
Ensure the IP configurations match your Core Network/DU setup:
- `amf_ip_address`
- `GNB_IPV4_ADDRESS_FOR_NG_AMF`
- `GNB_IPV4_ADDRESS_FOR_NGU`
- `tr_s_preference = "aerial";`
- `tr_s_shm_prefix = "nvipc";`

### Step 7.3: Fix the L1 Entrypoint Script
By default, the script that launches the L1 stack (`aerial_l1_entrypoint.sh`) assumes binaries are placed in an architecture-specific build directory like `build.aarch64/` or `build.x86_64/`. Because we built directly into `build/` (in Step 3), the container will throw a `command not found` error if this isn't corrected.

**File to Modify:** 
`$OAI_HOME/ci-scripts/yaml_files/sa_gnb_aerial/aerial_l1_entrypoint.sh`

**Action:** 
Open the file and modify the lines executing `cuphycontroller_scf` and `pcap_collect` by changing `build.$(uname -m)` to `build`.

**Before:**
```bash
export AERIAL_LOG_PATH=/var/log/aerial
sudo -E "$cuBB_Path"/build.$(uname -m)/cuPHY-CP/cuphycontroller/examples/cuphycontroller_scf $argument
sudo -E ./build.$(uname -m)/cuPHY-CP/gt_common_libs/nvIPC/tests/pcap/pcap_collect nvipc /tmp
```

**After:**
```bash
export AERIAL_LOG_PATH=/var/log/aerial
sudo -E "$cuBB_Path"/build/cuPHY-CP/cuphycontroller/examples/cuphycontroller_scf $argument
sudo -E ./build/cuPHY-CP/gt_common_libs/nvIPC/tests/pcap/pcap_collect nvipc /tmp
```

### Step 7.4: Adjust the Docker Compose YAML Configuration
The example Docker Compose YAML files provided by OAI contain hardcoded host paths (e.g., `/home/jixu/`) and varying image tags.

**File to Modify:** 
`$OAI_HOME/ci-scripts/yaml_files/sa_gnb_aerial/docker-compose.yaml` (You can also apply this to `docker-compose-dgx.yaml`).

**Action:**
1. Update the `volumes` section of the `nv-cubb` block to match your local **Aerial SDK Source** directory.
2. Update the `image` to match your local **Aerial L1 Docker image** (e.g., `aerial/aerial-cuda-accelerated-ran:dgx-spark`).

**Example Configuration for `nv-cubb`:**
```yaml
services:
  nv-cubb:
    container_name: nv-cubb
    # ... 
    volumes:
      # Use your actual Aerial SDK directory
      - /home/admin/aerial-cuda-accelerated-ran_dgx-spark_source:/opt/nvidia/cuBB/
      - /lib/modules:/lib/modules
      - /dev/hugepages:/dev/hugepages
      - /usr/src:/usr/src
      - ./aerial_l1_entrypoint.sh:/opt/nvidia/cuBB/aerial_l1_entrypoint.sh
      - /var/log/aerial:/var/log/aerial
      - ../../../cmake_targets/share:/opt/cuBB/share
    
    # Use your compiled L1 container image
    image: aerial/aerial-cuda-accelerated-ran:dgx-spark
```
Ensure the `oai-gnb-aerial` service `image` points to `oai-gnb-aerial:latest`. *(Note: if there is a single-file override mount from old config attempts, remove it).*

---

## 8. Running the Integrated Setup

### Step 8.1 Stop Standalone Containers
Docker Compose automatically manages both `nv-cubb` and `oai-gnb-aerial`. Since you manually ran the L1 container in Step 3/4 (`c_aerial_root`), you **must** shut it down first to avoid IPC shared memory and GPU resource conflicts.

```bash
docker stop c_aerial_root
docker rm c_aerial_root
```

### Step 8.2 Launch Docker Compose
```bash
cd "$OAI_HOME"/ci-scripts/yaml_files/sa_gnb_aerial
docker compose up -d
```
*Expected Output:*
```text
[+] up 2/2
 ✔ Container nv-cubb        Created                                     0.1s 
 ✔ Container oai-gnb-aerial Created                                     0.0s
```

---

## 9. Verification and Monitoring

Once started, `oai-gnb-aerial` will automatically initialize and wait for `nv-cubb`'s `nvipc` server over shared memory.

### Checking L1 (Aerial / cuBB) Logs
```bash
docker logs -f nv-cubb
```
You should observe the `cuphycontroller_scf` application bootstrapping, reporting throughput slots, and listing `[AERIAL_CUPHY_API_EVENT]` indicating the PHY layer is handling slot mapped events.

### Checking L2 (OAI gNB) Logs
```bash
docker logs -f oai-gnb-aerial
```
If the integration is successful, OAI will connect to L1 via `nvipc`. It will successfully step through SFN (System Frame Number) and Slot loops, continually printing lines similar to:
```text
[NR_MAC] I Frame.Slot 256.0
[NR_MAC] I Frame.Slot 384.0
[NR_MAC] I Frame.Slot 512.0
[NR_MAC] I Frame.Slot 640.0
```
This confirms that the OAI MAC layer is actively synchronized with the Aerial Physical layer. 

---

## 10. Teardown

To safely shut down the environment, navigate to the docker-compose folder and use the `down` command. This gracefully removes both containers and frees up the GPU and hugepages resources.

```bash
cd "$OAI_HOME"/ci-scripts/yaml_files/sa_gnb_aerial
docker compose down
```

---

## Troubleshooting

### A. `docker cp` says file not found in container
Use the absolute path inside the container, not `~`:
```bash
docker cp c_aerial_root:/opt/nvidia/cuBB/cuPHY-CP/gt_common_libs/nvipc_src.2026.03.16.tar.gz "$OAI_HOME"/
```

### B. `OAI_HOME` is empty
Ensure your environment variable is set:
```bash
export OAI_HOME=/home/admin/Downloads/openairinterface5g-1-11
```

### C. `Dockerfile.gNB.aerial.ubuntu` fails at `tar -xvzf nvipc_src.*.tar.gz`
**Cause:** There are multiple `nvipc_src.*.tar.gz` files in `$OAI_HOME` causing the tarball extraction wildcard to break.
**Fix:** Keep only one tarball in the build context.
```bash
cd "$OAI_HOME"
mkdir -p .nvipc_backup
mv nvipc_src.2025.12.23.tar.gz .nvipc_backup/
```
Then rebuild:
```bash
docker build . -f docker/Dockerfile.gNB.aerial.ubuntu --tag oai-gnb-aerial:latest
```

### D. `docker compose up -d` fails with `pull access denied` for `aerial-253-dgx-spark...`
**Cause:** `docker-compose.yaml` is using CI/private image tags that are not available on your host.
**Fix:** Point compose to local images you already built/pulled in Step 6 and Step 7.4.
Check available local images:
```bash
docker images | grep -E 'aerial|oai-gnb-aerial|ran-base'
```
Update `$OAI_HOME/ci-scripts/yaml_files/sa_gnb_aerial/docker-compose.yaml` accordingly.
