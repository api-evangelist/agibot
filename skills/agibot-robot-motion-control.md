---
generated: '2026-08-06'
method: generated
name: Bring an AgiBot robot up and move it under the AimDK protocol
description: Enable a robot, set its work mode and gait, issue joint/linear/planned motion, poll the task to completion, and clear alerts — over the published AimDK protobuf contract.
api: grpc/agibot-aimdk-protocol-index.yml
operations: [SetWorkMode, GetWorkMode, EnableRobot, GetWorkState, SetLocomotionGait, Stand, JointMove, LinearMove, PlanningMove, GetTaskState, SafeStop, GoHomePose, DisableRobot, GetAlertList, ClearAlert, CollisionRecover]
source: >-
  Every RPC name is verified verbatim in the .proto files saved under
  grpc/aimdk/protocol/, harvested from https://github.com/Link-U-OS/aimrt_protocol
  at commit 03263afbbe05b224e2525c70c7ec74afb119cecc.
---

# Bring an AgiBot robot up and move it under the AimDK protocol

AgiBot publishes the AimDK protocol — 136 proto3 files, 33 gRPC services, 175 RPCs — as the
contract for its robots. The services below are served **by the robot itself** over the AimRT
runtime on the local network. There is no public cloud endpoint, and there is no
authentication in the contract: reaching the robot's network *is* the access control. Treat
every call here as a physically consequential action.

## Envelope
- Requests carry `RequestHeader{timestamp}`. Requests that may block carry
  `BlockableRequestHeader{timestamp, blocked}`.
- Responses carry `ResponseHeader{code, msg, timestamp}` — **`code == 0` is success, any
  non-zero code is a failure** described by `msg`.
- Motion RPCs return `CommonTaskResponse{header, task_id, state}` where `state` is a
  `CommonState`: `CREATED`, `PENDING`, `RUNNING`, `SUCCESS`, `FAILURE`, `ABORTED`, `TIMEOUT`,
  `NOT_READY`, `IN_MANUAL`, `INVALID`. See `conventions/agibot-conventions.yml`.

## Steps
1. **Check you are clear to move** — `HalEmergencyService.GetEmergencyState` and
   `HalOperationModeService.GetOperationModeState`. Do not proceed while the robot is in
   emergency stop or a manual operation mode.
2. **Set the work mode** — `McBaseService.SetWorkMode`, then confirm with
   `McBaseService.GetWorkMode`.
3. **Enable the robot** — `McBaseService.EnableRobot`. Read back
   `McBaseService.GetWorkState` and `McBaseService.GetState` before commanding motion.
4. **Choose a gait and stand** — `McMotionService.SetLocomotionGait` (confirm with
   `GetLocomotionGait`), then `McMotionService.Stand`.
5. **Command the motion** — pick one:
   - `McMotionService.JointMove` — joint-space move.
   - `McMotionService.LinearMove` — Cartesian linear move.
   - `McMotionService.TrajectoryMove` — follow a trajectory.
   - `McMotionService.PlanningMove` — planned move; inspect the plan first with
     `GetTempPlanningResult`.
   - `McKinematicsService.KineForward` / `KineInverse` — solve poses before committing.
6. **Poll to completion** — `McService.GetTaskState` (or `McDataService.GetTaskState`) with
   the `task_id` until `CommonState` reaches `SUCCESS`, `FAILURE`, `ABORTED` or `TIMEOUT`.
   Never assume completion from the initial response.
7. **Stop and park** — `McMotionService.SafeStop` on any anomaly;
   `McMotionService.GoHomePose` to return to the home pose; `McBaseService.DisableRobot` when
   finished.

## End effectors and interaction
- `HalHandService.GetHandState` / `SetHandCommand`, or `McMotionService.SetHandCommand` for
  the OmniHand, AgiClaw, Inspire and Jodell hands.
- `McMotionService.SetNeckCommand`, `HalAudioService.PlayFile` / `SetAudioVolume`, and the
  `HalRgbLightService` / `HalFillLightService` / `HalLightStripService` trio for interaction.

## Errors and health
- Poll `HDSService.GetAlertList` and `GetExceptionEvent` while running. Alerts carry an
  `AlertLevel` (`FATAL`, `SERIOUS`, `WARNING`, `HIDDEN_DANGERS`, `STATUS`) and an
  `AlertSolution`.
- `HDSService.ClearAlert` clears a specific alert; `HalFaultService.ClearMcuFault` clears an
  MCU fault by `module_id` + `event_id`; `McSafetyService.CollisionRecover` recovers from a
  collision stop. Full catalogue: `errors/agibot-problem-types.yml`.

## Notes
- **No idempotency.** The contract defines no idempotency key. A retried `Set*` or move RPC
  is a second physical command. Confirm state with the paired `Get*` RPC instead of retrying
  blind.
- **List RPCs are unpaginated** — `GetAlertList`, `GetAvailableActions`, `GetAllAppInfos` and
  `GetBlockedExceptionCodeList` return whole collections.
- Exercise this against `Genie Sim` or `aimrt_mujoco_sim` before a physical robot — see
  `sandbox/agibot-sandbox.yml`.
