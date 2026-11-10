# Nova5 RGB pick and return

Robonix Pilot observes the configured black box through a wrist RGB camera, selects separate `pick`, `place` and `home` capabilities to lift the box 30 mm, replace it, and return the arm home. Each action must succeed before the next one starts. The `nova5_arm` primitive owns the arm and gripper connections; the camera and skill use its capabilities. SDK commands use Tool0 flange `MovL` at 50 mm/s with a 174 mm TCP offset: the measured 165 mm gripper assembly plus the 9 mm white adapter plate. This does not guarantee constant fingertip speed during rotation. No depth image is used.

## Install

Development and offline validation were performed on Ubuntu 22.04.5 LTS (x86_64, native Linux). This does not establish hardware acceptance of the split deployment.

Use Linux, Python 3.12 with `venv`, Git, and the [Robonix build dependencies](https://github.com/syswonder/robonix#prerequisites). This deployment uses a contributor integration commit containing optional reasoning-effort forwarding and the Executor lifecycle Driver changes proposed in upstream PRs #321 and #322. It is not an official Robonix release.

```bash
git clone --branch nova5/core-integration-v0.1.0 https://github.com/mikequ95/robonix.git robonix-nova5
cd robonix-nova5
git checkout --detach 29b7459d149a4fc1c9315dcad4a817f8f14bebbe
git submodule update --init --recursive
make install
rbnx setup "$PWD"
```

Keep this source checkout available to `rbnx`. The five repositories are this deployment, `nova5-support`, `primitive-dobot-nova5-arm-rbnx`, `primitive-dobot-nova5-camera-rbnx`, and `skill-nova5-pick-return-rbnx`, all version 0.1.0. Their builds install declared dependencies, including the pinned official Dobot and Orbbec SDKs. The support, arm, camera, and skill repositories have published `v0.1.0` tags.

## Configure

The hardware is a Nova5, an Orbbec Gemini 336L wrist camera, and a PGI Modbus RTU gripper. Set up USB/serial permissions and controller networking; ports 29999 and 30004 must be reachable. Follow `nova5-support`'s calibration instructions to obtain RGB intrinsics, the hand-eye transform, and the table plane. Existing calibration is valid only for the same unchanged mounting and base.

From this robot repository:

```bash
cp settings.example.json settings.json
```

Set the calibration configuration path, hardware and camera provider IDs, measured object height, allowed XY bounds, and home flange pose. Use `allowed_xy_mm: [[x_min, x_max], [y_min, y_max]]` and `home_flange_mm_deg: [x, y, z, rx, ry, rz]`, in base/User0 coordinates with Tool0 and mm/degrees. Paths resolve relative to `settings.json`. Generate calibration files under `calibration/`, or set their actual location. The hardware, camera and skill must use the same calibration configuration. Keep device settings, calibration results and credentials out of Git.

Before executing a grasp, set `motion.enabled=true`, `motion.speed_mm_s=50`, and verified finite XYZ workspace bounds in that calibration configuration; see the [skill configuration](https://github.com/mikequ95/skill-nova5-pick-return-rbnx#configure-and-run). The template deliberately disables motion. This deployment explicitly sets `nova5_arm.config.enable_opening=true` so the skill can release the box after placement; activation does not open the gripper. The 174 mm TCP already includes the adapter; do not add another 9 mm.

The URDF includes the nominal six-axis [Dobot Nova5 model](https://github.com/Dobot-Arm/DOBOT_6Axis_ROS2_V4/tree/7807ef8fb31e8f86a76cd474a4a9cddffef6eca9), the white adapter plate, and the TCP at 174 mm from the flange. Dobot meshes retain their MIT license. Arm joint limits are copied from upstream, including zero effort/velocity values; the SDK performs inverse kinematics and motion control.

The adapter is uniformly 9 mm thick. Its stem is 64 mm wide, with a 32 mm radius end centred on the flange; the distance from the round tip to the crossbar's near edge is 115 mm. The centred crossbar is 160 x 24 mm, giving an overall length of 139 mm. It extends along flange -X toward the camera. Mounting holes are not modelled.

The gripper is represented only by the `tcp` frame at `xyz="0 0 0.174"` relative to `flange`. The offset uses the measured 165 mm assembly length (including the black mounting plate) plus the 9 mm adapter. No gripper meshes or finger joints are distributed, so this URDF cannot model gripper collisions or contact. Verify the TCP against the actual mounting before motion.

The camera has an optical frame but no housing mesh. `wrist_rgb_optical_fixed` uses this installation's hand-eye calibration; update it from `T_flange_camera` after remounting (translation in metres, URDF RPY in radians). This transform is relative to the robot flange and must not receive another 9 mm offset.

## Run

```bash
export NOVA5_SUPPORT_SOURCE='git+https://github.com/mikequ95/nova5-support.git@v0.1.0'
export NOVA5_HARDWARE_SOURCE='git+https://github.com/mikequ95/primitive-dobot-nova5-arm-rbnx.git@v0.1.0'
export NOVA5_CALIBRATION_CONFIG="$PWD/calibration/config.json"
export NOVA5_GRIPPER_PORT='/dev/serial/by-id/<gripper-serial-id>'
export NOVA5_SETTINGS="$PWD/settings.json"
export VLM_BASE_URL='https://your-openai-compatible-endpoint/v1'
read -rsp 'API key: ' VLM_API_KEY; echo
export VLM_API_KEY
export VLM_REASONING_EFFORT=high
rbnx build -f robonix_manifest.yaml
rbnx boot -f robonix_manifest.yaml --verbose
```

The source variables also accept absolute paths to local checkouts. Keep all three providers on this host; feedback windows use its monotonic clock. The model endpoint must support `gpt-6-astra`, images, tool calls, structured responses, and `high` reasoning effort.

Before first real motion, verify that an independent hardware emergency stop is functional. Before motion, the arm must be enabled and idle in TCP control mode with zero User0/Tool0 offsets. The gripper must be initialized, idle (state 1), in IO mode 0, with force 20%, speed 1%, and equal target/actual opening. Start with at least 200 mm TCP clearance above the table and a clear path to the box.

In another terminal using the same `ROBONIX_HOME`, check registration and observation:

```bash
rbnx caps -v
rbnx ask 'Only call nova5_wrist_rgb to capture one wrist RGB image. Do not invoke any motion or grasp skill.'
```

Then request one cycle:

```bash
rbnx ask 'Observe the black box on the checkerboard with the Nova5 wrist camera and read the skill instructions. If clearly visible, pick it up, put it back at its recorded position, and return the arm home using the available Nova5 capabilities. Wait for each action to complete before continuing. Report the final result; stop on any failure and do not retry motion.'
```

The three actions share a persistent task journal. `pick` leaves the object held, `place` releases it at the recorded position and retreats, and `home` returns the empty arm. A failed or canceled action prevents continuation even after restart. Inspect the hardware and reported log; follow the skill README for journal recovery only after manual physical recovery. Restarting does not release the gripper or move the arm. Stop with:

```bash
rbnx shutdown -f robonix_manifest.yaml
```

For local development, use an ignored `robonix_manifest.local.yaml` copy: replace each package's `url` and `branch` with a `path` to its absolute source directory, then pass that manifest to `rbnx build` and `rbnx boot`.
