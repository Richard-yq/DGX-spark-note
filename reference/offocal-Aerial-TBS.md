# Part 3. Install Aerial Testbed Software Components
## Automated installation of Aerial CUDA-Accelerated RAN
Aerial CUDA-Accelerated RAN can be installed by following the [Aerial CUDA-Accelerated RAN Installation Guide](https://docs.nvidia.com/aerial/cuda-accelerated-ran/latest/install_guide/index.html).

For your convenience, the aerial-cuda-accelerated-ran repository includes the [NVIDIA Aerial SDK Installation script](https://github.com/NVIDIA/aerial-cuda-accelerated-ran/blob/main/cubb_scripts/install/README.md),
which automates the installation process.

From a fresh flash of DGX Spark, or from a fresh Ubuntu Server 22.04.5 installation, after the clone above, install using the scripts under `cubb_scripts/install/`.
It is recommended to run the installation in a tmux session so the installation can continue in the case of network issues.
With the below commands, the session can be reattached with `tmux attach -t aerial-cuda-accelerated-ran`.

```bash
git clone --recurse-submodules https://github.com/NVIDIA/aerial-cuda-accelerated-ran.git ~/aerial-cuda-accelerated-ran
cd ~/aerial-cuda-accelerated-ran/cubb_scripts/install/
sudo apt install make tmux git-lfs && git lfs pull
tmux new-session -s "aerial-cuda-accelerated-ran" -n "installation"
make all && sudo reboot
RU_MAC=<your RU MAC address> make all
```

On Ubuntu Server 22.04.5, the same sequence applies, but perform a **cold power cycle** after the second `make all` if the installation shows:

```bash
INFO[MISC]: NIC Firmware reset is not supported. Host power cycle is required
```

Then continue the installation by running `make all` again.

If errors occur or the host reboots before installation finishes, resume from the completed step by running `make all` again from `~/aerial-cuda-accelerated-ran/cubb_scripts/install/`.

Installation steps can be shown by prepending `DRY_RUN=1` to the `make all` command.

At this point you should have a running Aerial CUDA-Accelerated RAN installation including the L1 and OAI L2 + CN.

## Manual Installation of Aerial CUDA-Accelerated RAN
After following [Aerial CUDA-Accelerated RAN Installation Guide](https://docs.nvidia.com/aerial/cuda-accelerated-ran/latest/install_guide/index.html) these additional steps are required.

### Verify Inbound PTP Packets
Typically, you should see packets with `ethertype 0x88f7` on the selected interface.

```bash
sudo tcpdump -i aerial00 -c 5 | grep ethertype
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on aerial00, link-type EN10MB (Ethernet), capture size 262144 bytes
13:27:41.291503 48:b0:2d:63:83:ac (oui Unknown) > 01:1b:19:00:00:00 (oui Unknown), ethertype Unknown (0x88f7), length 60:
13:27:41.291503 48:b0:2d:63:83:ac (oui Unknown) > 01:1b:19:00:00:00 (oui Unknown), ethertype Unknown (0x88f7), length 60:
13:27:41.296727 c4:5a:b1:14:1a:c6 (oui Unknown) > 01:1b:19:00:00:00 (oui Unknown), ethertype Unknown (0x88f7), length 78:
13:27:41.296784 c4:5a:b1:14:1a:c6 (oui Unknown) > 01:1b:19:00:00:00 (oui Unknown), ethertype Unknown (0x88f7), length 60:
13:27:41.306316 08:c0:eb:71:e7:d5 (oui Unknown) > 01:1b:19:00:00:00 (oui Unknown), ethertype Unknown (0x88f7), length 58:
```

### Configure VLAN and IP Address on the gNB Server
#### Access a Foxconn RU
To be able to access a Foxconn RU over ssh, create a VLAN and IP address on the gNB server:

```bash
sudo ip link add link aerial00 name aerial00.2 type vlan id 2
sudo ip addr add 169.254.1.169/24 dev aerial00.2
sudo ip link set up aerial00.2
```

They can also be appended to (`/usr/local/bin/nvidia.sh`) without `sudo` so they are automatically run on reboot.

#### Access a WNC RU
Create (`/etc/netplan/wnc-access-setup.yaml`) and change the permissions:

```bash
sudo tee /etc/netplan/wnc-access-setup.yaml <<'EOF'
network:
  version: 2
  ethernets:
    aerial00:
      addresses:
      - 192.168.2.169/24
EOF
sudo chmod 600 /etc/netplan/wnc-access-setup.yaml
sudo netplan apply
```

You may need to reconnect to the server after this step.

### Start the Aerial cuBB Container and Build the Source Code
Execute the `/cuPHY-CP/container/run_aerial.sh` script to set up cuBB. This script downloads the image and starts the container; it will
then mount the local cuBB source code in the container and build it. Changes to the L1 script are stored locally.

```bash
# Run on host: start a docker container with the script below
cd ~/aerial-cuda-accelerated-ran
./cuPHY-CP/container/run_aerial.sh
```

Follow the [Aerial cuBB documentation](https://docs.nvidia.com/aerial/cuda-accelerated-ran/latest/quickstart_guide/running_cubb-end-to-end.html)
to build and run the cuphycontroller. The following instructions are for building and setting up the environment
for running cuphycontroller. The following commands must be run from inside the cuBB container.

```bash
aerial@c_aerial_user:/opt/nvidia/cuBB$ ${cuBB_SDK}/testBenches/phase4_test_scripts/build_aerial_sdk.sh --preset 10_02 -- -DSCF_FAPI_10_04_SRS=ON
```

### gNB Configuration Files
OAI L2 configuration used in the default setup is located [here](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb-vnf.sa.band78.273prb.aerial.conf).

L1 `cuphycontroller` configuration files are maintained in `cuPHY-CP/cuphycontroller/config/`.

For ATB 1.0, the setup has been validated with the following configuration files:

* [cuphycontroller_P5G_WNC_GH.yaml](https://github.com/NVIDIA/aerial-cuda-accelerated-ran/blob/main/cuPHY-CP/cuphycontroller/config/cuphycontroller_P5G_WNC_GH.yaml)

* [cuphycontroller_P5G_WNC_DGX.yaml](https://github.com/NVIDIA/aerial-cuda-accelerated-ran/blob/main/cuPHY-CP/cuphycontroller/config/cuphycontroller_P5G_WNC_DGX.yaml)

Configuration files are selected automatically based on server type by `run_l1.sh`.

There is also a configuration file for Foxconn RU on Grace Hopper, which can be specified as the first argument to `run_l1.sh`, eg: `./run_l1.sh P5G_FXN_GH`.

* [cuphycontroller_P5G_FXN_GH.yaml](https://github.com/NVIDIA/aerial-cuda-accelerated-ran/blob/main/cuPHY-CP/cuphycontroller/config/cuphycontroller_P5G_FXN_GH.yaml)

In all configurations `dst_mac_addr` must match your RU’s MAC address.

Before running the cuphycontroller, edit the `cuphycontroller_<xyz>.yaml` configuration file to point to the
correct MAC address of the O-RU and the correct PCIe address of the FH interface on the gNB. Determine whether
the NIC address is correct.

```bash
sed -i "s/ nic:.*/ nic: 0000:b5:00.0/" ${cuBB_SDK}/cuPHY-CP/cuphycontroller/config/cuphycontroller_<xyz>.yaml
sed -i "s/ dst_mac_addr:.*/ dst_mac_addr: 6c:ad:ad:00:02:02/" ${cuBB_SDK}/cuPHY-CP/cuphycontroller/config/cuphycontroller_<xyz>.yaml
sed -i "s/ vlan:.*/ vlan: 2/" ${cuBB_SDK}/cuPHY-CP/cuphycontroller/config/cuphycontroller_<xyz>.yaml
```

When the build is done and the configuration files are updated, exit the container. All changes will be stored locally, so you don’t need to commit the changes to a new docker image.

### Creating the NVIPC Source Code Package
In the latest release, NVIPC is no longer included in the OAI repository. You must copy the source package from
the Aerial cuBB SDK and add it to the OAI build files. Execute the following to create the `nvipc_src.<data>.tar.gz`
file.

```bash
cd ~/aerial-cuda-accelerated-ran
./cuPHY-CP/gt_common_libs/pack_nvipc.sh
cp ./cuPHY-CP/gt_common_libs/nvipc_src.$(date +%Y.%m.%d).tar.gz ~/openairinterface5g
```

This will create and copy the `nvipc_src.<data>.tar.gz` archive that is required to build OAI L2+ for Aerial.
Instructions can also be found in the [Aerial FAPI tutorial](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/Aerial_FAPI_Split_Tutorial.md).

### Build gNB Docker Image
In ATB 1.0, the OAI image is built in two steps. Instructions can be found in the
[Aerial FAPI tutorial](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/Aerial_FAPI_Split_Tutorial.md).

Note

When building a Docker image, the files are copied from the filesystem into the image. After you build
the image, you must make changes to the configuration inside the container.

### Build OAI L2 image
Clone the OpenAirInterface5G repository.

```bash
git clone --branch ATB1.0_integration https://gitlab.eurecom.fr/oai/openairinterface5g.git ~/openairinterface5g
cd openairinterface5g
```

Use the below commands to build the OAI L2 code:

```bash
cd ~/openairinterface5g/
docker build . -f docker/Dockerfile.base.ubuntu --tag ran-base:latest
docker build . -f docker/Dockerfile.gNB.aerial.ubuntu --tag oai-gnb-aerial:latest
```

This will create two images:

* `ran-base:latest` includes the environment to build the OAI source code. This will be used when building the `oai-gnb-aerial:latest` image.

* `oai-gnb-aerial:latest` will be used later in the Docker Compose script to create the OAI L2 container.

### Run gNB with Docker Compose
ATB 1.0 includes a Docker Compose configuration for running gNB software. Refer to the
[OAI instructions](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/Aerial_FAPI_Split_Tutorial.md)
on how to run with Docker Compose. The Docker Compose configuration for SMC-GH servers is located in
the following file path:

```bash
~/openairinterface5g/ci-scripts/yaml_files/sa_gnb_aerial/docker-compose.yaml
```

The Docker compose script will start containers to run cuBB and OAI L2+. The script also includes
an entry-point script for cuBB, which starts cUBB with the `cuphycontroller_P5G_WNC_GH.yaml` default configuration.
To use a custom configuration, you can supply it as an argument in the Docker compose file. This script is located
in the following file path:

```bash
~/aerial-cuda-accelerated-ran/run_l1.sh
```

Before running the Docker Compose script, you need to create a local `~/openairinterface5g/ci-scripts/yaml_files/sa_gnb_aerial/.env` file to set the tag of the local OAI L2 image.

```bash
echo 'REGISTRY=""' > .env
echo 'TAG="latest"' >> .env
```

The tag will set in the following location of the `docker-compose.yaml` file.

```bash
image: ${REGISTRY-oaisoftwarealliance/}${GNB_IMG:-oai-gnb-aerial}:${TAG:-develop}
```

You can now run ATB with the below commands.

```bash
cd ~/openairinterface5g/ci-scripts/yaml_files/sa_gnb_aerial
docker compose up -d
# console of cuBB
docker logs -f nv-cubb
# console of oai
docker logs -f oai-gnb-aerial
```

After following the instructions, you should have the following images:

```bash
$ docker image ls
REPOSITORY                                          TAG         IMAGE ID       CREATED         SIZE
oai-gnb-aerial                                      latest      1f382f5c8962   6 days ago      514MB
ran-base                                            latest      5ed95a3c6ad5   6 days ago      2.43GB
nvcr.io/nvidia/aerial/aerial-cuda-accelerated-ran   26-1-cubb   8e25b9b049d2   7 weeks ago     28.4GB
```

### Example Screenshot of Starting CN5G
```bash
aerial@e200-smc-11:~/openairinterface5g/doc/tutorial_resources/oai-cn5g$ docker compose up -d
[+] Running 11/11
✔ Network oai-cn5g-public-net  Created                 0.1s
✔ Container oai-ext-dn         Started                 0.6s
✔ Container mysql              Started                 0.4s
✔ Container oai-nrf            Started                 0.5s
✔ Container asterisk-ims       Started                 0.5s
✔ Container oai-udr            Started                 0.7s
✔ Container oai-udm            Started                 0.9s
✔ Container oai-ausf           Started                 1.1s
✔ Container oai-amf            Started                 1.3s
✔ Container oai-smf            Started                 1.4s
✔ Container oai-upf            Started                 1.6s
```

### Example Screenshot of Running CN5G
```bash
aerial@e200-smc-11:~$ docker ps
CONTAINER ID   IMAGE                                   COMMAND                  CREATED        STATUS                PORTS                                                   NAMES
21f4d9fc7b7a   oaisoftwarealliance/oai-upf:develop     "sh /openair-upf/bin…"   5 days ago     Up 5 days (healthy)     2152/udp, 8805/udp, 5342-5344/tcp                     oai-upf
dec8c3999718   oaisoftwarealliance/oai-smf:develop     "/openair-smf/bin/oa…"   5 days ago     Up 5 days (healthy)     80/tcp, 5342-5344/tcp, 8080/tcp, 9090/tcp, 8805/udp   oai-smf
5b39b97eca1f   oaisoftwarealliance/oai-amf:develop     "/openair-amf/bin/oa…"   5 days ago     Up 5 days (healthy)     80/tcp, 5342-5344/tcp, 8080/tcp, 9090/tcp, 38412/sctp oai-amf
4395198c8f63   oaisoftwarealliance/oai-ausf:develop    "/openair-ausf/bin/o…"   5 days ago     Up 5 days (healthy)     80/tcp, 5342-5344/tcp, 8080/tcp                       oai-ausf
198740927b35   oaisoftwarealliance/oai-udm:develop     "/openair-udm/bin/oa…"   5 days ago     Up 5 days (healthy)     80/tcp, 5342-5344/tcp, 8080/tcp                       oai-udm
ba7c80dacdfc   oaisoftwarealliance/oai-udr:develop     "/openair-udr/bin/oa…"   5 days ago     Up 5 days (healthy)     80/tcp, 8080/tcp                                      oai-udr
f0b31e825e61   oaisoftwarealliance/oai-nrf:develop     "/openair-nrf/bin/oa…"   5 days ago     Up 5 days (healthy)     80/tcp, 5342-5344/tcp, 8080/tcp                       oai-nrf
27be7c0d2e6a   oaisoftwarealliance/trf-gen-cn5g:latest "/bin/bash -c ' ip r…"   5 days ago     Up 5 days (healthy)                                                           oai-ext-dn
ddbbd35aa2d9   mysql:8.0                               "docker-entrypoint.s…"   5 days ago     Up 5 days (healthy)     3306/tcp, 33060/tcp                                   mysql
c89de994abc6   oaisoftwarealliance/ims:latest          "asterisk -fp"           5 days ago     Up 5 days (healthy)                                                           ims
```
