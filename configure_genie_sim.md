# Configure GenieSim on EC2 With Isaac Sim WebRTC

This runbook starts after Isaac Sim/Docker/NVIDIA GPU support are already working on the EC2 instance. It records the working setup used to run GenieSim in Docker and view it from a MacBook with Isaac Sim WebRTC Streaming Client.

## 1. Prerequisites

- EC2 GPU instance with NVIDIA driver working.
- Docker with NVIDIA runtime working.
- Isaac Sim / GenieSim Docker image available:

```bash
docker images | grep -E "genie-sim|isaac-sim"
```

Expected useful image:

```text
registry.agibot.com/genie-sim/open_source:latest
```

Verify GPU:

```bash
nvidia-smi
```

The tested instance used an NVIDIA L40S GPU.

## 2. Get GenieSim Assets Correctly

The asset repo uses Git LFS. If you skip LFS, Isaac will show prim names but fail to load real robot/object USD payloads.

Install Git LFS:

```bash
sudo apt-get update
sudo apt-get install -y git-lfs
git lfs install
```

Clone assets into the required location. The old tutorial branch `rolling` did not exist when tested. Use Hugging Face `main` or ModelScope `master`.

```bash
cd ~/genie_sim/source/geniesim

git clone https://huggingface.co/datasets/agibot-world/GenieSimAssets --branch main assets
# or:
# git clone https://www.modelscope.cn/datasets/agibot_world/GenieSimAssets.git --branch master assets
```

Pull real LFS files:

```bash
cd ~/genie_sim/source/geniesim/assets
git lfs pull
```

Sanity check: these files must be real files, not ~130 byte LFS pointers:

```bash
ls -lh   ~/genie_sim/source/geniesim/assets/robot/G1_omnipicker/configuration/robot_physics.usd   ~/genie_sim/source/geniesim/assets/objects/benchmark/table/benchmark_table_019/Aligned.usd
```

Expected approximate sizes after LFS pull:

```text
robot_physics.usd: ~13K
Aligned.usd: hundreds of KB or larger
```

If a file starts with this text, LFS was not pulled:

```text
version https://git-lfs.github.com/spec/v1
```

## 3. EC2 Network Ports

Open these inbound ports from your MacBook public IP only:

```text
TCP 49100
UDP 47998
```

The tested EC2 public IP was:

```text
16.171.20.220
```

If the EC2 public IP changes, update `PUBLIC_IP` when starting the container.

## 4. Start GenieSim WebRTC Container

From repo root:

```bash
cd ~/genie_sim
PUBLIC_IP=16.171.20.220 LIVESTREAM_PORT=49100 ./scripts/start_streaming.sh
```

This starts a detached Docker container named:

```text
genie_sim_benchmark
```

The script is intentionally different from `scripts/start_gui.sh`:

- `start_gui.sh` is X11/GUI-display oriented.
- `start_streaming.sh` is WebRTC/headless-viewer oriented.
- It uses `--network=host`, `--gpus all`, and exports `PUBLIC_IP` / `LIVESTREAM_PORT`.

If the script says the container already exists, enter it with:

```bash
./scripts/into.sh
```

Or restart it:

```bash
docker stop genie_sim_benchmark
docker rm genie_sim_benchmark
PUBLIC_IP=16.171.20.220 LIVESTREAM_PORT=49100 ./scripts/start_streaming.sh
```

If port `49100` is already in use, stop any stale Isaac Sim stream first:

```bash
docker ps
# If an old container named isaac-sim owns the stream:
docker stop isaac-sim
```

## 5. Run the WebRTC Demo

Inside the container, use the helper script so ROS/Jazzy/Isaac env vars are set without relying on interactive shell aliases:

```bash
cd ~/genie_sim
./scripts/into.sh

# inside container:
cd /geniesim/main
./scripts/run_webrtc_demo.sh
```

The helper runs:

```bash
/isaac-sim/python.sh /geniesim/main/source/geniesim/app/app.py   --config source/geniesim/config/s2r_select_color.yaml   --app.headless true   --app.livestream 2
```

Useful log:

```bash
tail -f /tmp/geniesim_webrtc.log
```

Good signs in the log:

```text
Streaming server started.
Saved task json ... table_task_g1_0.json
Added new Workspace ... pick_block_color/0/scene.usda
EPISODE FILE: ... table_task_g1_0.json
```

## 6. Connect From MacBook

Open Isaac Sim WebRTC Streaming Client on the Mac and connect to:

```text
16.171.20.220:49100
```

If you previously connected to a stale stream, fully close and reopen the client before reconnecting.

If the viewport angle is awkward:

- Select `Workspace` or `G1` in the Stage tree.
- Press `F` to frame selection.
- Use the camera dropdown and choose a perspective/free camera if needed.

## 7. What This Demo Does and Does Not Do

The WebRTC client is mainly a viewer/inspector. It is not the robot command interface. UI operations from the client are slow because Isaac UI and RTX rendering are streamed over the network.

The tutorial config uses:

```yaml
benchmark:
  model_arc: pi
  infer_host: localhost:8999
```

That means GenieSim expects a policy/inference server at:

```text
localhost:8999
```

Without that policy server, the scene can load, but the robot will not perform useful benchmark actions.

For manual control, investigate the teleop stack:

```bash
source/geniesim/config/teleop.yaml
scripts/autoteleop.sh
```

That path launches multiple ROS/teleop processes and is separate from the WebRTC viewer.

## 8. Troubleshooting

### WebRTC shows empty default Isaac stage

Likely connected to a stale Isaac Sim stream. Check port ownership:

```bash
ss -ltnup | grep -E '49100|47998|8211|8899'
docker ps
```

Stop stale container if needed:

```bash
docker stop isaac-sim
```

Restart GenieSim WebRTC run.

### Stage tree has prims but viewport is black

Common causes:

- Active camera is a robot sensor camera with a bad/limited view.
- Assets are still LFS pointers.
- Scene is still loading.

Check LFS files:

```bash
ls -lh ~/genie_sim/source/geniesim/assets/robot/G1_omnipicker/configuration/robot_physics.usd
```

Check logs:

```bash
docker exec genie_sim_benchmark tail -n 200 /tmp/geniesim_webrtc.log
```

### Logs show `Could not open asset ... Aligned.usd` or robot USD files

Run:

```bash
cd ~/genie_sim/source/geniesim/assets
git lfs pull
```

### Logs show `rclpy still not available`

Do not launch with a bare non-interactive alias. Use:

```bash
./scripts/run_webrtc_demo.sh
```

The helper exports the needed ROS/Jazzy/Isaac variables.

### Robot does not move

The scene is working, but `s2r_select_color.yaml` needs a policy server at `localhost:8999` because it uses `model_arc: pi`. Start the intended inference service, or use a teleop/manual control path.

## 9. Files Added/Modified for This Setup

- `scripts/start_streaming.sh`: starts the GenieSim Docker container for WebRTC, without X11/DISPLAY.
- `scripts/run_webrtc_demo.sh`: launches GenieSim with ROS/Jazzy/Isaac env vars and WebRTC mode.
- `source/geniesim/app/workflow/app_launcher.py`: makes `app.livestream: 2` enable Isaac Sim WebRTC.
- `source/geniesim/app/controllers/api_core.py`: keeps a human-friendly perspective camera in WebRTC mode instead of forcing the robot head camera.
