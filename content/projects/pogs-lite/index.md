---
title: "POGS-lite: Minimal Object-Centric Gaussian Reproduction"
date: 2026-02-01
tags: ["3DGS", "Manipulation", "Segmentation"]
featured: true
weight: 20
summary_line: "A constrained reproduction of POGS: object-level Gaussian grouping via DINO feature distillation on a tabletop scene with 2–4 objects."
summary: "A scoped reproduction effort targeting the core contribution of POGS: partitioning a 3DGS scene into object-level Gaussian groups that can be tracked and queried independently. Focused on a controlled tabletop environment."
---

## Motivation

Reading POGS raises a question I cannot answer without implementing it: how cleanly do the Gaussian groups actually separate in practice? The paper reports results on curated scenes. I want to understand the failure modes — specifically, when does the feature distillation produce ambiguous group assignments, and how does group quality affect downstream manipulation queries?

POGS-lite is a minimal reproduction targeted at this question, not at reproducing the paper's full quantitative results.

---

## Scope

**In scope:**
- Standard 3DGS reconstruction on a tabletop scene with 3–5 distinct objects
- DINO ViT-S/8 feature distillation into Gaussian attributes
- K-means clustering over distilled features to produce Gaussian groups
- Visualization: per-group color rendering, per-group isolation rendering
- A simple contact-region query: given a group label, extract the 50th percentile depth Gaussians as a contact candidate set

**Out of scope (for now):**
- Rigid body tracking across dynamic frames
- Integration with a grasp planner
- Real-time performance

---

## Planned System Structure

```
[Multi-view images]
       |
       v
[3DGS optimization]  -->  [Gaussian parameters: mu, R, s, alpha, sh_coeffs]
       |
       v
[Per-view DINO feature extraction]
       |
       v
[Feature distillation optimization]  -->  [Per-Gaussian feature vector (64-dim)]
       |
       v
[K-means clustering]  -->  [Per-Gaussian group label]
       |
       v
[Visualization + query interface]
```

Dependencies:
- A differentiable Gaussian rasterizer (using the original 3DGS CUDA implementation as base)
- `transformers` (for DINO ViT)
- `open3d` (for point cloud and mesh extraction utilities)
- `scikit-learn` (K-means)
- `nerfstudio` or a standalone 3DGS training script

---

## What I Already Understand

- The 3DGS optimization loop: initialization from COLMAP sparse point cloud, adaptive densification and pruning, photometric loss with the differentiable rasterizer.
- DINO feature properties: ViT-S/8 features at patch level (14x14 patches for a 112x112 input) capture semantic and structural information that generalizes well across object categories. This is why they work for segmentation-by-clustering.
- The distillation step is a second optimization pass: freeze the Gaussian parameters, add a learned feature attribute per Gaussian, minimize the reprojection error between rendered feature maps and DINO feature maps.
- K-means in high-dimensional feature space (64-dim): initialization matters. Use K-means++ initialization; run with 5 seeds and select by inertia.

---

## What Remains Difficult

**Feature normalization.** DINO features have varying magnitude across patches. Normalizing per-image vs. globally changes the clustering result. I need to establish a consistent normalization strategy empirically.

**Choosing K.** The number of object groups is not known a priori. For the controlled tabletop setup I plan to use, I will set K manually. In a general system, this requires either a learned K or a hierarchical clustering approach.

**Thin objects and transparent objects.** Standard 3DGS performs poorly on thin structures (cables, tool handles) and transparent surfaces (glass containers). These objects are common in manipulation scenarios. POGS-lite will simply avoid these object types in its initial evaluation.

**Evaluation without ground truth part labels.** Since I am not working with a labeled dataset, I will evaluate grouping quality qualitatively (do the isolated groups visually correspond to single objects?) and functionally (does the contact-region query return plausible contact candidates?).

---

## Why It Matters for Future Research

Implementing POGS-lite gives me direct familiarity with the 3DGS optimization codebase, the feature distillation step, and the practical limitations of Gaussian-based object segmentation. This is foundational for:

- Extending the work to dynamic scenes (where groups must be tracked frame to frame)
- Connecting to RoboSplat-lite (where object groups need to be placed at novel positions)
- Understanding what geometric assumptions break when scaling to more complex environments

The goal is not to publish a result. It is to have a working system that I understand end-to-end — a prerequisite for contributing to this line of research in a lab setting.
