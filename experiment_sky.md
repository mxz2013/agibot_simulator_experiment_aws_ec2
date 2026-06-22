# AGIBOT GenieSim on AWS EC2 Experiment

This document summarizes the current state of the AGIBOT GenieSim experiment on AWS EC2. It is intended as a handoff for people who want to reproduce the setup, continue the simulation work, or extend it into a complete benchmark run.

## Objective

Run an AGIBOT GenieSim simulation on an AWS EC2 GPU instance and view the Isaac Sim scene from a local machine through Isaac Sim WebRTC Streaming Client.

Reference tutorial:

https://agibot-world.com/sim-evaluation/docs/#/v3?id=_312-run-a-simulation-task

## Current Result

The working setup reached this state:

- Isaac Sim 5.1.0 runs headless in Docker on an AWS `g6e.2xlarge` instance.
- Docker can access the NVIDIA L40S GPU through the NVIDIA runtime.
- GenieSim can start inside the Docker environment.
- The Isaac Sim / GenieSim scene can be viewed from a MacBook, or any local machine that supports Isaac Sim WebRTC Streaming Client:
  https://docs.isaacsim.omniverse.nvidia.com/6.0.0/installation/manual_livestream_clients.html
- The viewer is useful for inspecting the scene, but it is not the robot control interface.

The main remaining gap is policy execution. The tested GenieSim configuration expects an inference server at `localhost:8999`. Without that policy server, the scene can load, but the robot does not perform useful benchmark actions.

## Repository

The public GitHub repository for these notes is:

https://github.com/mxz2013/agibot_simulator_experiment_aws_ec2

It currently contains the two detailed runbooks:

- `installation_g6e.2xlarge.md`
- `configure_genie_sim.md`

Do not commit private keys, PEM files, AWS credentials, tokens, or account-specific secrets to this repository.

## Tested Environment

The known-good setup used:

- AWS EC2 instance type: `g6e.2xlarge`
- GPU: NVIDIA L40S, about 46 GB VRAM
- OS: Ubuntu 24.04.4 LTS
- Kernel: `6.17.0-1017-aws`
- Docker: `29.5.3`
- NVIDIA driver: `580.159.04`
- Isaac Sim image: `nvcr.io/nvidia/isaac-sim:5.1.0`
- GenieSim image: `registry.agibot.com/genie-sim/open_source:latest`
- Local viewer: Isaac Sim WebRTC Streaming Client on macOS, or another supported local platform

## How to Reproduce

Follow the runbooks in this order.

### 1. Install and Validate Isaac Sim

Use `installation_g6e.2xlarge.md`.

This runbook covers:

- launching and connecting to the EC2 instance
- checking GPU, Docker, and NVIDIA runtime access
- replacing the default NVIDIA driver if Isaac Sim crashes with the newer driver
- pulling the Isaac Sim 5.1.0 Docker image
- creating persistent Isaac Sim cache/config/data directories
- running Isaac Sim compatibility checks
- starting Isaac Sim in livestream mode
- connecting from Isaac Sim WebRTC Streaming Client

Success criterion:

- `nvidia-smi` works on the host and inside Docker
- Isaac Sim starts headless without RTX renderer crashes
- the WebRTC client can connect to the EC2 instance

### 2. Configure and Run GenieSim

Use `configure_genie_sim.md`.

In the tested workflow, I connected VS Code to the EC2 instance over SSH, installed the Codex extension in VS Code, and completed the configuration with Codex's help. In principle, a future maintainer can give `configure_genie_sim.md` to Codex as context and ask it to reproduce the configuration on a new instance.

This runbook covers:

- preparing GenieSim assets with Git LFS
- opening the required EC2 network ports
- starting the GenieSim WebRTC container
- running the WebRTC demo script
- connecting from the MacBook viewer
- distinguishing viewer access from robot control
- troubleshooting empty stages, stale streams, missing assets, and policy-server issues

Success criterion:

- the WebRTC client connects to the GenieSim stream
- the scene loads with real USD assets, not Git LFS pointer files
- the logs show the streaming server and task scene starting correctly

## Important Notes

- `g6e.2xlarge` has limited capacity in some AWS regions. On-demand GPU instances may not be immediately available every time.
- To avoid repeating setup work, create an AMI from a stopped working instance. If one region has no available capacity, launch a new instance from the AMI in another region.
- EC2 public IP addresses can change after stopping and restarting an instance. Update the `PUBLIC_IP` value and WebRTC client target whenever this happens.
- Security group rules should allow SSH and WebRTC ports only from the developer's current public IP, not from the whole internet.
- GenieSim assets must be pulled through Git LFS. If the USD files are tiny text pointer files, the scene will not load correctly.

## Known Issues and Workarounds

### AWS GPU Capacity

Problem: `g6e.2xlarge` instances may be unavailable in a selected region.

Workaround: keep a configured AMI and try another region with available GPU capacity.

### Isaac Sim Driver Compatibility

Problem: the fresh AWS image used an NVIDIA driver that passed compatibility checks but crashed at runtime in RTX renderer libraries.

Workaround: use the apt-managed NVIDIA `580.159.04` driver recorded in `installation_g6e.2xlarge.md`.

### Empty or Stale WebRTC Stage

Problem: the WebRTC client may connect to an old Isaac Sim stream instead of the current GenieSim scene.

Workaround: stop stale containers, restart the GenieSim streaming container, and fully close/reopen the WebRTC client before reconnecting.

### No Robot Benchmark Action

Problem: the tutorial configuration expects a policy/inference server at `localhost:8999`.

Workaround: implement or connect the required inference server, or investigate GenieSim's teleoperation path separately.

## Suggested Next Steps

1. Add or connect the policy/inference server expected by GenieSim at `localhost:8999`.
2. Confirm whether the benchmark task can run end-to-end after the policy server is available.
3. Record the exact command sequence and logs for a successful complete benchmark episode.
4. Evaluate the teleoperation path in `source/geniesim/config/teleop.yaml` and `scripts/autoteleop.sh`.
5. Convert the working EC2 setup into an AMI and document the AMI launch procedure.
6. Replace account-specific IPs and key names in the runbooks with placeholders before broader public sharing.

## Files

- `installation_g6e.2xlarge.md`: detailed Isaac Sim and EC2 installation runbook.
- `configure_genie_sim.md`: detailed GenieSim setup, streaming, and troubleshooting runbook.
- `experiment_sky.md`: this high-level handoff summary.
