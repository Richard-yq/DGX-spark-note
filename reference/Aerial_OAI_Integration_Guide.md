# Comprehensive Guide: Integrating NVIDIA Aerial L1 and OAI gNB

This tutorial provides a complete, step-by-step guide to successfully integrating the **NVIDIA Aerial SDK L1 (cuBB)** with the **OpenAirInterface (OAI) 5G gNB L2** using Docker Compose. By following these steps, anyone can reproduce the environment and establish IPC communication (`nvipc`) between the L1 and L2 containers.

---

## 1. Prerequisites and Environment Setup

Before you begin, ensure you have the following components prepared on your host machine:

1. **NVIDIA Aerial SDK Source Code:** 
   The source directory containing the compiled `cuPHY-CP` and other L1 components. 
   *Example path:* `/home/admin/aerial-cuda-accelerated-ran_dgx-spark_source`

2. **OAI 5G Source Repository:**
   The cloned OpenAirInterface 5G repository. 
   *Example path:* `/home/admin/Downloads/openairinterface5g-1-11`

3. **Compiled Docker Images:**
   * **Aerial L1 Image:** Make sure you have the Aerial container built or committed (e.g., `aerial/aerial-cuda-accelerated-ran:dgx-spark`).
   * **OAI gNB Aerial Image:** The pre-built OAI L2 image compatible with Aerial (e.g., built using `Dockerfile.gNB.aerial.ubuntu`).

---

## 2. Step-by-Step Configuration

### Step 2.1: Fix the L1 Entrypoint Script
By default, the script that launches the L1 stack (`aerial_l1_entrypoint.sh`) assumes the compiled binaries are placed in an architecture-specific build directory (e.g., `build.aarch64/` or `build.x86_64/`). If your source code was compiled directly into the `build/` folder, the container will fail to start L1 and throw a `command not found` error.

**File to Modify:** 
`<OAI_DIR>/ci-scripts/yaml_files/sa_gnb_aerial/aerial_l1_entrypoint.sh`

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

### Step 2.2: Adjust the Docker Compose YAML Configuration
The example Docker Compose YAML files provided by OAI contain hardcoded host paths (e.g., `/home/jixu/cuBB_dgx_arc`) and specific image tags that need to be adapted to your local environment.

**File to Modify:** 
`<OAI_DIR>/ci-scripts/yaml_files/sa_gnb_aerial/docker-compose.yaml` (You can also apply this to `docker-compose-dgx.yaml`).

**Action:**
1. Focus on the `nv-cubb` service block.
2. Update the `volumes` section so the host path points exactly to your local **Aerial SDK Source** directory.
3. Update the `image` to match your local **Aerial L1 Docker image**.

**Example Configuration:**
```yaml
services:
  nv-cubb:
    container_name: nv-cubb
    # ... (other configs)
    volumes:
      # REPLACE the /home/jixu/... path with your actual Aerial SDK directory
      - /home/admin/aerial-cuda-accelerated-ran_dgx-spark_source:/opt/nvidia/cuBB/
      - /lib/modules:/lib/modules
      - /dev/hugepages:/dev/hugepages
      - /usr/src:/usr/src
      - ./aerial_l1_entrypoint.sh:/opt/nvidia/cuBB/aerial_l1_entrypoint.sh
      - /var/log/aerial:/var/log/aerial
      - ../../../cmake_targets/share:/opt/cuBB/share
    
    # REPLACE the image tag with your compiled L1 container image
    image: aerial/aerial-cuda-accelerated-ran:dgx-spark
```

*(Note: The `oai-gnb-aerial` service block generally does not need path changes because its volume mounts use relative paths inside the OAI repository. Just ensure `gnb.conf` network IPs configuration matches your AMF/UPF setup.)*

---

## 3. Running the Integrated Setup

### 3.1: Stop Standalone Containers
Docker Compose will automatically manage both the `nv-cubb` and `oai-gnb-aerial` containers. If you previously ran an L1 container manually using `run_aerial.sh` (resulting in a container named `c_aerial_root` or similar), you **must** shut it down first to avoid IPC shared memory and GPU resource conflicts.

```bash
# Stop and remove any manual Aerial container
docker stop c_aerial_root
docker rm c_aerial_root
```

### 3.2: Launch Docker Compose
Navigate to the directory containing the modified YAML file and launch the cluster in detached mode:

```bash
cd /home/admin/Downloads/openairinterface5g-1-11/ci-scripts/yaml_files/sa_gnb_aerial
docker compose up -d
```
*Expected Output:*
```text
[+] up 2/2
 ✔ Container nv-cubb        Created                                     0.1s 
 ✔ Container oai-gnb-aerial Created                                     0.0s
```

---

## 4. Verification and Monitoring

Once started, `oai-gnb-aerial` will automatically initialize and wait for `nv-cubb`'s `nvipc` server to start. You can monitor their progress by tracking the respective logs.

### Checking L1 (Aerial / cuBB) Logs
```bash
docker logs -f nv-cubb
```
You should observe the `cuphycontroller_scf` application bootstrapping, reporting DL/UL throughput slots, and listing `[AERIAL_CUPHY_API_EVENT]` indicating the PHY layer is handling slot mapped events.

### Checking L2 (OAI gNB) Logs
```bash
docker logs -f oai-gnb-aerial
```
If the integration is successful, OAI will connect to L1 via `nvipc`. You will see it successfully step through SFN (System Frame Number) and Slot loops, continually printing lines similar to:
```text
[NR_MAC] I Frame.Slot 256.0
[NR_MAC] I Frame.Slot 384.0
[NR_MAC] I Frame.Slot 512.0
[NR_MAC] I Frame.Slot 640.0
```
This confirms that the OAI MAC layer is actively synchronized with the Aerial Physical layer. 

---

## 5. Teardown
To safely shut down the environment, use the compose down command. This gracefully removes both containers and frees up the GPU and hugepages resources.

```bash
docker compose down
```
