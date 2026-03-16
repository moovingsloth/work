---
title: "RoboSplat-lite: Single-Demonstration Augmentation via 3DGS"
date: 2026-02-15
tags: ["3DGS", "Manipulation", "Teleoperation", "Digital-Twin"]
featured: true
weight: 30
summary_line: "A minimal implementation of the RoboSplat pipeline: one recorded pick-and-place demonstration augmented into 50+ synthetic training examples via Gaussian scene manipulation."
summary: "A scoped implementation of the RoboSplat data augmentation pipeline for single pick-and-place tasks. One demonstration, re-rendered from 20 novel viewpoints and 3 object placements, producing a synthetic training set for behavior cloning."
---

## Motivation

RoboSplat's central claim — that a 3DGS scene model can multiply the value of a single demonstration — is compelling but requires validation in a concrete setup. The paper works in a specific robotics environment with specific hardware. I want to understand whether the pipeline holds up in a simpler, more controllable setting: a tabletop pick-and-place task with one object and one clear grasp point.

RoboSplat-lite tests the pipeline at minimum complexity, which is the right starting point for understanding where the hard problems actually are.

---

## Scope

**In scope:**
- A single tabletop pick-and-place demonstration (recorded as a sequence of end-effector poses + gripper state)
- 3DGS reconstruction of the workspace from a static multi-view capture
- Object group extraction (using the POGS-lite grouping approach)
- Synthetic data generation: 20 novel viewpoints + 3 novel object placements = 60 synthetic demonstration variants
- Behavior cloning baseline trained on the synthetic dataset

**Out of scope:**
- Real robot deployment (simulation only for this phase)
- Multi-object scenes
- Contact-rich manipulation (push, slide, assembly)

---

## Planned System Structure

```
Phase 1: Scene Reconstruction
  [Multi-view static capture]  -->  [3DGS model]  -->  [POGS-lite object groups]

Phase 2: Demonstration Grounding
  [Demo trajectory: {t, EEF_pose, gripper_state}]
  [Camera calibration: K, [R|t] for each view]
  -->  [Grounded trajectory in world frame]

Phase 3: Augmented Data Generation
  For each (viewpoint V, object placement P):
    1. Transform object group Gaussians by P (SE(3) transform in table plane)
    2. Retarget trajectory: apply same P to end-effector poses
    3. Render scene from V using transformed 3DGS model
    4. Output: image sequence at V + retargeted action sequence

Phase 4: Policy Training
  [60 augmented demonstrations]  -->  [Behavior cloning (MLP or simple diffusion policy)]
  [Evaluate: task success rate in simulation on held-out object placements]
```

---

## What I Already Understand

**3DGS rendering from novel viewpoints** is the most mature component. Once a 3DGS model is optimized, rendering from an arbitrary camera pose takes milliseconds on a GPU. This is where 3DGS has a clear advantage over NeRF or mesh-based representations for this use case.

**Rigid body trajectory retargeting** for pick-and-place is straightforward: if the object moves by a transform T in the table plane, the end-effector trajectory moves by the same T (the grasp approach vector and gripper orientation are defined relative to the object). This breaks for tasks where the approach trajectory is constrained by the environment (e.g., approaching from above because of a shelf), but for open tabletop tasks it holds.

**Background hole-filling** is the expected weak point. When the object moves to a new position, the background under the object's original position is revealed. In the current scope, I will handle this by limiting object placement to regions within the training capture volume, where the background reconstruction is reliable.

---

## What Remains Difficult

**Demonstration quality sensitivity.** If the single demonstration is unusual (awkward grasp angle, non-minimal trajectory), the augmented demonstrations inherit those artifacts. There is no averaging over multiple demonstrations to smooth out idiosyncrasies.

**Sim-to-real visual gap.** 3DGS renders look good but not identical to real images. For behavior cloning, the policy network learns appearance features from the training distribution. If train and test images have different visual characteristics, performance degrades. This is manageable in simulation (where train and test are both synthetic) but would be a problem for real robot deployment.

**Articulated gripper rendering.** The demonstration trajectory includes gripper state, but the 3DGS model of the scene does not include the robot arm. Rendering the arm consistently in synthetic images requires either: (a) a mesh model of the arm overlaid on the 3DGS render, (b) masking the arm region and training the policy to work without arm visibility, or (c) a robot-aware 3DGS extension. For this lite implementation, I will use approach (b).

---

## Why It Matters for Future Research

RoboSplat-lite establishes the full data generation pipeline from scene capture through policy training. Even in a minimal form, this demonstrates:

1. That I can work end-to-end with 3DGS as a data augmentation engine.
2. That the pipeline's failure modes are visible and diagnosable (not a black box).
3. A working baseline for extending to harder tasks: multi-object, contact-rich, or bi-manual.

The simulation setup will also serve as a test bed for evaluating future improvements: better object segmentation, contact-aware trajectory retargeting, or domain adaptation for real robot deployment.
