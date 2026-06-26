# Aerial + OAI Integration Guide (DGX Spark)

This document is an end-to-end SOP to build and run NVIDIA Aerial L1 with OAI gNB (Aerial FAPI split) on your current environment.

## 1. Environment Paths

```bash
export AERIAL_HOME=/home/admin/aerial-cuda-accelerated-ran_dgx-spark_source
export OAI_HOME=/home/admin/Downloads/openairinterface5g-1-11
```

## 2. Extract Aerial Source

```bash
cd ~
tar -xzf ~/Downloads/aerial-cuda-accelerated-ran_dgx-spark_source.tar.gz
cd "$AERIAL_HOME"
```

## 3. Launch Aerial Container and Build L1

```bash
cd "$AERIAL_HOME"
sudo ./cuPHY-CP/container/run_aerial.sh
```

Inside the container:

```bash
cd /opt/nvidia/cuBB
cmake -Bbuild -GNinja \
  -DCMAKE_TOOLCHAIN_FILE=cuPHY/cmake/toolchains/grace-cross \
  -DCMAKE_CUDA_ARCHITECTURES="121-real" \
  -DENABLE_20C=OFF \
  -DENABLE_CUMAC=OFF

cmake --build build -j"$(nproc --all)"
```

## 4. Start L1

Inside the container:

```bash
cd /opt/nvidia/cuBB
sudo -E ./run-l1.sh
```

### L1 Success Markers

L1 is considered started when logs include:

- `gRPC Server listening on 0.0.0.0:50051`
- `====> PhyDriver initialized!`
- `cuPHYController configured for 1 cells`
- `====> cuPHYController initialized, L1 is ready!`
- `UlPhyDriver... initialized` and `DlPhyDriver... initialized`

### Common Non-Fatal Warnings

- `Cannot find MPS control daemon process`
- `YAML invalid key ... Using default value`
- `eCPRI parser not supported on NIC ... retrying without eCPRI`

## 5. Generate nvIPC Source Tarball

Inside the container:

```bash
cd /opt/nvidia/cuBB/cuPHY-CP/gt_common_libs
./pack_nvipc.sh
ls -lh nvipc_src.*.tar.gz
```

Expected output example:

- `nvipc_src.2026.03.16.tar.gz`

## 6. Copy nvIPC Tarball to OAI Project

On host:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}'
```

Use the container name shown above (example: `c_aerial_root`):

```bash
docker cp c_aerial_root:/opt/nvidia/cuBB/cuPHY-CP/gt_common_libs/nvipc_src.2026.03.16.tar.gz "$OAI_HOME"/
```

Verify:

```bash
ls -lh "$OAI_HOME"/nvipc_src.2026.03.16.tar.gz
```

## 7. Build OAI Docker Images

```bash
cd "$OAI_HOME"
docker build . -f docker/Dockerfile.base.ubuntu --tag ran-base:latest
docker build . -f docker/Dockerfile.gNB.aerial.ubuntu --tag oai-gnb-aerial:latest
```

Verify images:

```bash
docker images | grep -E 'ran-base|oai-gnb-aerial'
```

![alt text](image.png)
## 8. Configure L1 (Aerial)

Edit:

- `$AERIAL_HOME/cuPHY-CP/cuphycontroller/config/cuphycontroller_P5G_WNC_DGX.yaml`

Confirm these values match your RU/NIC setup:

- `dst_mac_addr`
- `nic`
- `vlan`

## 9. Configure OAI gNB

Edit your OAI gNB config (for example under `ci-scripts/conf_files/`).

Required checks:

- `amf_ip_address`
- `GNB_IPV4_ADDRESS_FOR_NG_AMF`
- `GNB_IPV4_ADDRESS_FOR_NGU`
- `tr_s_preference = "aerial";`
- `tr_s_shm_prefix = "nvipc";`

## 10. Configure Docker Compose (Aerial + OAI)

Edit:

- `$OAI_HOME/ci-scripts/yaml_files/sa_gnb_aerial/docker-compose.yaml`

In `nv-cubb` service `volumes`, mount the full Aerial source directory:

```yaml
- /home/admin/aerial-cuda-accelerated-ran_dgx-spark_source:/opt/nvidia/cuBB/
```

If present, remove the single-file override mount for old YAML to avoid confusion.

## 11. Start the Integrated Setup

```bash
cd "$OAI_HOME"/ci-scripts/yaml_files/sa_gnb_aerial
docker compose up -d
```

Follow logs:

```bash
docker logs -f nv-cubb
docker logs -f oai-gnb-aerial
```

## 12. Stop the Setup

```bash
cd "$OAI_HOME"/ci-scripts/yaml_files/sa_gnb_aerial
docker compose down
```

## Troubleshooting

### A. `docker cp` says file not found in container

Use absolute path inside container, not `~`:

```bash
docker cp c_aerial_root:/opt/nvidia/cuBB/cuPHY-CP/gt_common_libs/nvipc_src.2026.03.16.tar.gz "$OAI_HOME"/
```

### B. `OAI_HOME` is empty

Set it again:

```bash
export OAI_HOME=/home/admin/Downloads/openairinterface5g-1-11
```

### C. `Dockerfile.gNB.aerial.ubuntu` fails at `tar -xvzf nvipc_src.*.tar.gz`

Cause: multiple `nvipc_src.*.tar.gz` files in `$OAI_HOME`.

Fix: keep only one tarball in build context.

```bash
cd "$OAI_HOME"
mkdir -p .nvipc_backup
mv nvipc_src.2025.12.23.tar.gz .nvipc_backup/
```

Then rebuild:

```bash
docker build . -f docker/Dockerfile.gNB.aerial.ubuntu --tag oai-gnb-aerial:latest
```

### D. `pull access denied` during `run_aerial.sh`

This happens when registry auth/repo access is not available. If local image exists, script may still continue and run. For clean setup, configure registry login/access first.

### E. `docker compose up -d` fails with `pull access denied` for `aerial-253-dgx-spark...`

Cause: `docker-compose.yaml` is using CI/private image tags that are not available on your host.

Fix: point compose to local images you already built/pulled.

Check available local images:

```bash
docker images | grep -E 'aerial|oai-gnb-aerial|ran-base'
```

Edit `$OAI_HOME/ci-scripts/yaml_files/sa_gnb_aerial/docker-compose.yaml`:

- `nv-cubb`:
  - set `image: aerial/aerial-cuda-accelerated-ran:dgx-spark`
- `oai-gnb-aerial`:
  - set `image: oai-gnb-aerial:latest`

Then restart compose:

```bash
cd "$OAI_HOME"/ci-scripts/yaml_files/sa_gnb_aerial
docker compose up -d
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}' | grep -E 'nv-cubb|oai-gnb-aerial'
```



---
Aerial + OAI Integration Setup Complete
The Docker Compose files have been successfully updated to match your local environment.

Changes Made
Modified 
docker-compose.yaml
 to use path /home/admin/aerial-cuda-accelerated-ran_dgx-spark_source instead of the hardcoded jixu path.
Updated the image tag to aerial/aerial-cuda-accelerated-ran:dgx-spark.
Replicated the volume path changes in docker-compose-dgx.yaml.
Next Steps to Start the Setup
Stop the manual L1 container: Because docker-compose will manage the L1 (nv-cubb) container, you'll need to stop the one you started manually.

bash
docker stop c_aerial_root
docker rm c_aerial_root
Verify gNB IPs (Optional but Recommended): If you have a customized Core Network IP or DU NIC IP, ensure they are updated in: /home/admin/Downloads/openairinterface5g-1-11/targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-vnf.sa.band78.273prb.aerial.conf

amf_ip_address
GNB_IPV4_ADDRESS_FOR_NG_AMF
GNB_IPV4_ADDRESS_FOR_NGU
Start the Integrated Setup:

bash
cd /home/admin/Downloads/openairinterface5g-1-11/ci-scripts/yaml_files/sa_gnb_aerial
docker compose up -d
Monitor the Setup:

bash
docker logs -f nv-cubb
docker logs -f oai-gnb-aerial
To stop everything later, just run docker compose down in that same folder.

----