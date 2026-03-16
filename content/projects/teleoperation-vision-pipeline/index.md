---
title: "Teleoperation Vision Pipeline"
date: 2026-01-15
tags: ["Teleoperation", "Digital-Twin"]
featured: true
weight: 10
summary_line: "Real-time multi-camera data pipeline from RealSense D435 and stereo cameras through CycloneDDS, connecting sensor capture to future scene representation work."
summary: "A real-time image transport and synchronization infrastructure for RealSense D435 and stereo cameras over CycloneDDS, with Linux kernel tuning for deterministic latency. Grounding future 3DGS teleoperation work in production-grade sensor infrastructure."
---

## Motivation

Before a robot can use a 3D scene representation during teleoperation, someone has to reliably get camera images from the robot to the compute node with bounded latency and without significant loss. This is not a research problem in the academic sense, but it is a prerequisite to everything that is a research problem.

My work at ROBROS required building and maintaining exactly this infrastructure. This project documents that work and extends it toward the specific requirements of 3DGS-based scene understanding: synchronized multi-camera streams, accurate timestamps, and the network configuration needed to support real-time reconstruction.

---

## Scope

- Multi-camera image transport for RealSense D435 (RGB + depth) and stereo camera pairs
- Communication layer: CycloneDDS over heterogeneous LAN/WiFi networks
- Linux kernel tuning for reduced network jitter and interrupt latency
- Time synchronization across camera sources (PTP or NTP-based)
- A thin Python consumer that saves synchronized frame sets to disk and optionally feeds a 3DGS reconstruction pipeline

Out of scope for now: the 3DGS reconstruction itself and any online scene update logic.

---

## System Structure

```
[RealSense D435]  --USB--> [Camera Node]  --DDS topic: /camera/realsense/rgb, /depth-->  [Compute Node]
[Stereo Camera L] --USB--> [Camera Node]  --DDS topic: /camera/stereo/left-->             [Compute Node]
[Stereo Camera R] --USB--> [Camera Node]  --DDS topic: /camera/stereo/right-->            [Compute Node]

[Compute Node]: subscriber + synchronizer + frame buffer + writer
```

The Camera Node runs on a lightweight ARM or x86 embedded host attached to the robot. The Compute Node runs on a workstation or server with GPU for downstream processing.

CycloneDDS is configured with:
- Reliable QoS for depth images (occasional drops are unacceptable for reconstruction)
- Best-effort QoS for RGB images during high-load conditions (latency matters more than completeness)
- Custom network interface selection to route DDS traffic away from the WiFi channel used for robot command communication

---

## What I Already Understand

From the ROBROS work:

- CycloneDDS QoS parameter tuning (history depth, reliability, deadline policies)
- Linux kernel parameters for network performance: `net.core.rmem_max`, `net.ipv4.tcp_rmem`, interrupt coalescing settings
- RealSense D435 SDK integration (librealsense2): frame alignment, depth scale calibration, intrinsic export
- Synchronization strategies: message-based timestamping vs. hardware trigger

The WiFi latency problem was the most persistent issue: standard 802.11 WiFi introduces variable delays (10–100ms jitter is typical) that break naive synchronization. The fix involved: explicit QoS deadline settings so that the subscriber knows when a frame is late, and a frame-drop policy that discards frames older than one control cycle.

---

## What Remains Difficult

- **Accurate multi-camera time synchronization over WiFi.** PTP requires hardware support and a deterministic network. Over WiFi, the best I achieved was ~5ms synchronization accuracy — acceptable for 3DGS (which can tolerate moderate timestamp errors) but not for high-speed manipulation tracking.
- **Depth stream alignment to stereo.** RealSense D435 has a known depth accuracy limitation at close range (< 0.3m). For manipulation at close range, the stereo pair may be more reliable, but fusing the two requires careful extrinsic calibration.
- **Bandwidth ceilings on WiFi.** Streaming RGB + depth at 30fps from a RealSense requires approximately 40–60 Mbps raw (before any compression). On a shared WiFi channel, this leaves little headroom. Compression (H.264, JPEG) reduces bandwidth but adds latency and artifacts in depth data.

---

## Why It Matters for Future Research

A 3DGS-based teleoperation display (as reviewed in the human-in-the-loop Gaussian splatting review) requires exactly this kind of infrastructure to work at real-time rates. The quality of the 3DGS model depends on the quality and synchronization of the input frames. A reconstruction pipeline that receives unsynchronized, jittery frames will produce artifacts that affect operator performance.

Building this infrastructure now means that when I work on online 3DGS reconstruction, the sensor layer is not a bottleneck. The software architecture is also designed to be modular: the DDS topic structure and the frame synchronizer are independent of the downstream consumer, so they can feed a reconstruction module, a segmentation module, or a policy training data recorder without changes.
