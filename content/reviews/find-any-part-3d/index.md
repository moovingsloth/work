---
title: "Find Any Part in 3D: Part-Level Grounding for Interaction-Aware Scene Understanding"
date: 2026-03-10
tags: ["Geometric-Vision", "Segmentation", "Manipulation", "3DGS"]
featured: true
weight: 40
insight: "Lifting open-vocabulary part segmentation to 3D: if a robot can localize semantic parts (handle, hinge, button) in 3D geometry, it can plan contact without task-specific annotation."
summary: "A review examining part-level 3D grounding — how open-vocabulary language queries can identify physically meaningful regions in 3D scenes for manipulation-aware scene understanding."
---

## Why This Paper Matters

Whole-object recognition is not enough for manipulation. Grasping a mug requires knowing where the handle is. Opening a drawer requires knowing where the pull is. Turning a valve requires knowing where the grip surfaces are. Part-level scene understanding bridges the gap between object recognition and manipulation planning.

Find Any Part in 3D addresses the problem of grounding open-vocabulary part queries into 3D geometry, without requiring part-specific training data. This makes it broadly applicable: you describe the part you care about in natural language, and the system localizes it in the 3D representation.

---

## Core Technical Idea

The approach distills 2D part segmentation capabilities (from models like SAM and open-vocabulary detectors) into a 3D representation. The pipeline:

1. **Multi-view 2D part segmentation.** For each captured view, a 2D model produces part masks given a text query.
2. **3D lifting.** Part mask predictions across views are aggregated into the 3D Gaussian representation using the Gaussian's projection correspondence. Each Gaussian accumulates votes from its visible views.
3. **3D part grounding.** The aggregated votes produce a per-Gaussian part probability, which can then be thresholded to extract 3D part regions.

The result is a 3D scene where Gaussian subsets are labeled by part semantics — queryable at runtime with natural language.

---

## What Is Geometric Here

The lifting step is a 3D aggregation problem. Each Gaussian projects to a pixel region in each view where it is visible. The part mask probability at those pixels is back-projected to the Gaussian. With multiple views, a Gaussian accumulates evidence from different angles.

This is related to the general problem of multi-view feature aggregation — the same structure appears in feature distillation for POGS and in NeRF-based semantic field methods. The geometric insight is that 3D consistency constrains the semantics: a part that appears consistently across views at the same 3D location has stronger evidence than one that appears only from a single viewpoint.

---

## What Is Optimization-Relevant Here

The aggregation is typically done without gradient-based optimization — it is a counting or averaging operation per Gaussian. However, variants that include a learned aggregation (attention-weighted, for example) would involve optimization.

The 2D part segmentation quality is the dominant source of noise. SAM-based segmentation can produce inconsistent part boundaries across views due to: varying occlusion, view-dependent appearance, and ambiguity in part definitions. These inconsistencies propagate into noisy 3D part regions.

---

## Why It Matters for Manipulation

The downstream application is direct: given a 3D scene with Gaussian part labels, a grasp planner can:

1. Query for the relevant part (e.g., "handle").
2. Extract the 3D point set corresponding to that part.
3. Plan a grasp in the part's local coordinate frame.

This removes the need for task-specific grasp annotation databases for each object category. The representation generalizes to novel objects as long as the 2D part model generalizes.

For teleoperation, this also supports operator interfaces where the user queries a part by name and the system highlights it in the display — a direct interaction affordance.

---

## Implementation Notes

For a reproduction targeting manipulation-relevant part detection:

- Focus on a small set of objects with well-defined parts (drawers, mugs, bottles, tools).
- Use SAM with point prompts derived from the text query (via a detector like Grounding DINO to get initial bounding boxes).
- For the aggregation, simple per-Gaussian vote counting with a consistency filter (require > N views to agree) is a reasonable baseline.
- Evaluate by whether the extracted 3D region supports a successful grasp, not by mIoU against a part annotation — the downstream task is what matters.

---

## My Follow-up Questions

1. How does part localization quality scale with the number of views? Is there a diminishing returns curve, and where is the practical sweet spot?
2. For deformable parts (cable, cloth flap), how should the representation handle the fact that the part's 3D position changes frame by frame?
3. Can part representations be shared across object instances of the same category? A "handle" representation learned from a mug should transfer to a pitcher — what structural assumption enables or limits this?
4. What is the failure mode when the text query is ambiguous (e.g., "grip area" on an object with multiple grip regions)?
