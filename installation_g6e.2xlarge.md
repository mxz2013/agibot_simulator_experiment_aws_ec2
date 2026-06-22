# Isaac Sim 5.1.0 Installation Runbook for AWS g6e.2xlarge

Date written: 2026-06-13

Goal: start from a fresh `g6e.2xlarge` EC2 instance and get NVIDIA Isaac Sim 5.1.0 running headless with livestream support for AGIBOT simulation work.

Known-good result from the successful attempt:

- Instance type: `g6e.2xlarge`
- GPU: NVIDIA L40S, about 46 GB VRAM
- OS: Ubuntu 24.04.4 LTS
- Kernel: `6.17.0-1017-aws`
- Docker: `29.5.3`
- Working NVIDIA driver: `580.159.04`
- Isaac Sim image: `nvcr.io/nvidia/isaac-sim:5.1.0`

## 1. Connect to EC2

Use the key for the instance:

```bash
chmod 400 ec2_agibot_frankfurt.pem
ssh -i ec2_agibot_frankfurt.pem ubuntu@EC2_PUBLIC_IP
```

For the successful Frankfurt instance:

```bash
ssh -i ec2_agibot_frankfurt.pem ubuntu@63.179.111.58
```

If SSH times out, check the EC2 Security Group and Network ACL.

Inbound SSH should allow:

```text
TCP 22 from YOUR_MAC_PUBLIC_IP/32
```

Find your Mac public IP:

```bash
curl -s https://checkip.amazonaws.com
```

## 2. Inspect the Instance

Run these on the EC2 instance:

```bash
hostnamectl
uname -a
nvidia-smi
df -h
groups
docker --version
docker info --format '{{json .Runtimes}}'
```

Expected shape:

```text
Hardware Model: g6e.2xlarge
GPU: NVIDIA L40S
Docker runtime includes: nvidia
ubuntu user is in docker group
```

## 3. Verify GPU Access From Docker

Run:

```bash
docker run --rm --runtime=nvidia --gpus all \
  nvcr.io/nvidia/cuda:12.8.0-base-ubuntu24.04 \
  nvidia-smi
```

Expected:

```text
GPU Name: NVIDIA L40S
```

If this fails, fix NVIDIA Container Toolkit / Docker GPU runtime before continuing.

## 4. Replace R595 Driver With R580 If Needed

The fresh AWS image had driver `595.71.05`. Isaac Sim compatibility check passed with R595, but real Isaac Sim runtime crashed in:

```text
librtx.scenedb.plugin.so
libcarb.scenerenderer-rtx.plugin.so
```

The fix was to replace R595 with apt-managed R580.

Check current driver:

```bash
nvidia-smi
cat /proc/driver/nvidia/version
dkms status | grep nvidia || true
command -v nvidia-uninstall || true
```

If driver is `595.71.05`, replace it:

```bash
sudo systemctl stop nvidia-persistenced 2>/dev/null || true
sudo systemctl stop dcgm 2>/dev/null || true

if command -v nvidia-uninstall >/dev/null 2>&1; then
  sudo nvidia-uninstall --silent || true
fi

sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y nvidia-driver-580
sudo reboot
```

Reconnect after reboot:

```bash
ssh -i ec2_agibot_frankfurt.pem ubuntu@EC2_PUBLIC_IP
```

Verify R580:

```bash
nvidia-smi
cat /proc/driver/nvidia/version
dkms status | grep nvidia || true
```

Expected:

```text
Driver Version: 580.159.04
nvidia/580.159.04, 6.17.0-1017-aws, x86_64: installed
```

## 5. Pull Isaac Sim 5.1.0

```bash
docker pull nvcr.io/nvidia/isaac-sim:5.1.0
```

Expected image:

```text
nvcr.io/nvidia/isaac-sim:5.1.0
Digest: sha256:f3563cb2ba0c18af0b2fb321360dcb73a917b899f879e3213623d6bee484fa54
```

## 6. Create Persistent Isaac Sim Directories

```bash
install -d -m 755 \
  ~/docker/isaac-sim/cache/main/ov \
  ~/docker/isaac-sim/cache/main/warp \
  ~/docker/isaac-sim/cache/computecache \
  ~/docker/isaac-sim/config \
  ~/docker/isaac-sim/data/documents \
  ~/docker/isaac-sim/data/Kit \
  ~/docker/isaac-sim/logs \
  ~/docker/isaac-sim/pkg

sudo chown -R 1234:1234 /home/ubuntu/docker/isaac-sim
```

## 7. Common Docker Mounts

Use these mounts for Isaac Sim commands:

```bash
-v /home/ubuntu/docker/isaac-sim/cache/main:/isaac-sim/.cache:rw
-v /home/ubuntu/docker/isaac-sim/cache/computecache:/isaac-sim/.nv/ComputeCache:rw
-v /home/ubuntu/docker/isaac-sim/logs:/isaac-sim/.nvidia-omniverse/logs:rw
-v /home/ubuntu/docker/isaac-sim/config:/isaac-sim/.nvidia-omniverse/config:rw
-v /home/ubuntu/docker/isaac-sim/data:/isaac-sim/.local/share/ov/data:rw
-v /home/ubuntu/docker/isaac-sim/pkg:/isaac-sim/.local/share/ov/pkg:rw
```

Use:

```bash
-e ACCEPT_EULA=Y
-e PRIVACY_CONSENT=Y
--runtime=nvidia --gpus all
--network=host
-u 1234:1234
```

`ACCEPT_EULA=Y` is required for Isaac Sim to run. `PRIVACY_CONSENT=Y` was used in the successful run.

## 8. Run Isaac Sim Compatibility Check

```bash
docker run --entrypoint bash --runtime=nvidia --gpus all --rm --network=host \
  -e ACCEPT_EULA=Y \
  -e PRIVACY_CONSENT=Y \
  -v /home/ubuntu/docker/isaac-sim/cache/main:/isaac-sim/.cache:rw \
  -v /home/ubuntu/docker/isaac-sim/cache/computecache:/isaac-sim/.nv/ComputeCache:rw \
  -v /home/ubuntu/docker/isaac-sim/logs:/isaac-sim/.nvidia-omniverse/logs:rw \
  -v /home/ubuntu/docker/isaac-sim/config:/isaac-sim/.nvidia-omniverse/config:rw \
  -v /home/ubuntu/docker/isaac-sim/data:/isaac-sim/.local/share/ov/data:rw \
  -v /home/ubuntu/docker/isaac-sim/pkg:/isaac-sim/.local/share/ov/pkg:rw \
  -u 1234:1234 \
  nvcr.io/nvidia/isaac-sim:5.1.0 \
  ./isaac-sim.compatibility_check.sh --/app/quitAfter=10 --no-window
```

Expected:

```text
GPU 0: NVIDIA L40S [supported]
GPU 0: VRAM [excellent]
RAM [good]
Storage [good]
System checking result: PASSED
```

Headless warnings are expected:

```text
GLFW initialization failed
failed to open the default display
Display [no display was detected]
```

## 9. Run Python Smoke Test

This test crashed with R595 and worked with R580.

```bash
docker run --runtime=nvidia --gpus all --rm --network=host \
  -e ACCEPT_EULA=Y \
  -e PRIVACY_CONSENT=Y \
  -v /home/ubuntu/docker/isaac-sim/cache/main:/isaac-sim/.cache:rw \
  -v /home/ubuntu/docker/isaac-sim/cache/computecache:/isaac-sim/.nv/ComputeCache:rw \
  -v /home/ubuntu/docker/isaac-sim/logs:/isaac-sim/.nvidia-omniverse/logs:rw \
  -v /home/ubuntu/docker/isaac-sim/config:/isaac-sim/.nvidia-omniverse/config:rw \
  -v /home/ubuntu/docker/isaac-sim/data:/isaac-sim/.local/share/ov/data:rw \
  -v /home/ubuntu/docker/isaac-sim/pkg:/isaac-sim/.local/share/ov/pkg:rw \
  -u 1234:1234 \
  --entrypoint bash \
  nvcr.io/nvidia/isaac-sim:5.1.0 \
  -lc ./python.sh\ standalone_examples/api/isaacsim.core.api/add_cubes.py
```

Expected:

```text
Simulation App Starting
app ready
```

The example may continue running. Stop it with `Ctrl-C`.

If it crashes with:

```text
Segmentation fault
librtx.scenedb.plugin.so
libcarb.scenerenderer-rtx.plugin.so
```

then check that the active host driver is R580, not R595:

```bash
nvidia-smi
```

## 10. Start Isaac Sim Headless Livestream

```bash
docker run -d --name isaac-sim \
  --runtime=nvidia --gpus all \
  -e ACCEPT_EULA=Y \
  -e PRIVACY_CONSENT=Y \
  --rm --network=host \
  -v /home/ubuntu/docker/isaac-sim/cache/main:/isaac-sim/.cache:rw \
  -v /home/ubuntu/docker/isaac-sim/cache/computecache:/isaac-sim/.nv/ComputeCache:rw \
  -v /home/ubuntu/docker/isaac-sim/logs:/isaac-sim/.nvidia-omniverse/logs:rw \
  -v /home/ubuntu/docker/isaac-sim/config:/isaac-sim/.nvidia-omniverse/config:rw \
  -v /home/ubuntu/docker/isaac-sim/data:/isaac-sim/.local/share/ov/data:rw \
  -v /home/ubuntu/docker/isaac-sim/pkg:/isaac-sim/.local/share/ov/pkg:rw \
  -u 1234:1234 \
  --entrypoint bash \
  nvcr.io/nvidia/isaac-sim:5.1.0 \
  -lc ./runheadless.sh\ -v
```

Watch logs:

```bash
docker logs -f isaac-sim
```

Expected readiness line:

```text
Isaac Sim Full Streaming App is loaded.
```

First startup can take several minutes while shader caches compile. These lines are normal:

```text
Waiting for RtPso async group async compilation
```

Verify container and GPU process:

```bash
docker ps --filter name=isaac-sim
nvidia-smi
```

Expected:

```text
container name: isaac-sim
process: /isaac-sim/kit/kit
GPU memory: about 3 GB or more
```

## 11. Check Livestream Ports On EC2

Run on the EC2 instance:

```bash
ss -lntup | grep -E '47998|49100|8211|8899|47995|47996|47999|48000' || true
```

In the successful run, EC2 listened on:

```text
TCP 49100
```

## 12. Open AWS Networking For Mac Visualization

In EC2 Security Group inbound rules, allow from your Mac public IP:

```text
TCP 49100 from YOUR_MAC_PUBLIC_IP/32
UDP 47998 from YOUR_MAC_PUBLIC_IP/32
```

If the subnet Network ACL is restrictive, also allow:

```text
Inbound TCP 49100 from YOUR_MAC_PUBLIC_IP/32
Inbound UDP 47998 from YOUR_MAC_PUBLIC_IP/32
Outbound ephemeral ports 1024-65535 to YOUR_MAC_PUBLIC_IP/32
```

Find your Mac public IP:

```bash
curl -s https://checkip.amazonaws.com
```

Check connectivity from your Mac:

```bash
nc -vz EC2_PUBLIC_IP 49100
```

If this times out while `ss` shows EC2 is listening, the issue is AWS Security Group or Network ACL, not Isaac Sim.

## 13. Useful Operations

SSH:

```bash
ssh -i ec2_agibot_frankfurt.pem ubuntu@EC2_PUBLIC_IP
```

Show Isaac Sim logs:

```bash
docker logs -f isaac-sim
```

Stop Isaac Sim:

```bash
docker stop isaac-sim
```

Show running containers:

```bash
docker ps -a
```

Show GPU state:

```bash
nvidia-smi
```

## 14. Known Lessons

- `g4dn.2xlarge` with Tesla T4 was not reliable for Isaac Sim 5.1.0 runtime.
- `g6e.2xlarge` with L40S is appropriate.
- The AWS image's R595 driver passed compatibility but crashed real Isaac Sim runtime.
- Installing R580 fixed the RTX renderer crash.
- The compatibility checker alone is not enough; always run the Python smoke test and the livestream startup.
- For AWS rules, the inbound source is your Mac public IP, not the EC2 public IP.

## 15. Next Step For AGIBOT

After this runbook succeeds:

1. Keep `isaac-sim` running.
2. Open livestream ports from your Mac.
3. Connect with the Isaac Sim livestream client.
4. Then proceed with AGIBOT repository/setup and the simulation task:
   - robot walks in a room
   - scans items
   - detects persons
   - counts number of persons
   - streams visualization to the Mac

