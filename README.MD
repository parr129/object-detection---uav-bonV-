# GPU-Accelerated Tracking of an Unregistered Object Across Video Frames

## Problem Statement

The objective of this project is to design a system capable of tracking a user-selected object throughout a video sequence without relying on predefined object classes such as cars, people, or animals.

The system receives:

* A video file
* An initial bounding box identifying the target object in the first frame

For every subsequent frame, the system outputs:

* Estimated bounding box `(x, y, width, height)`
* Tracker confidence
* Motion model confidence
* Trusted output source

The challenge is maintaining stable tracking even when the visual tracker becomes uncertain due to changes in appearance, lighting, scale, motion blur, or partial occlusion.

---

# Solution Overview

The proposed solution combines two independent tracking components:

### 1. Generic Visual Object Tracker

Tracks the object based on its visual appearance.

### 2. Constant Velocity Motion Model

Predicts where the object should appear in the next frame based on previous motion.

The outputs from both components are fused to generate a more stable and reliable estimate.

This hybrid design mimics how modern tracking systems operate: combining visual observations with motion predictions to reduce drift and improve robustness.

---

# System Architecture

```text
Input Video
     │
     ▼
Initial Bounding Box
     │
     ▼
Template Extraction
     │
     ▼
┌──────────────────────┐
│  Visual Object       │
│      Tracker         │
└──────────────────────┘
            │
            ▼
      Tracker Estimate
            │
            │
            ▼
┌──────────────────────┐
│ Constant Velocity    │
│   Motion Model       │
└──────────────────────┘
            │
            ▼
      Motion Estimate
            │
            ▼
┌──────────────────────┐
│  Fusion Module       │
└──────────────────────┘
            │
            ▼
 Final Tracking Output
```

---

# Approach

## Step 1: User Initialization

The user manually selects an object in the first frame using a bounding box.

Unlike traditional object detectors, the system does not need to know whether the object is a car, bicycle, person, drone, or any other category.

The selected region becomes the tracking template.

---

## Step 2: Template-Based Visual Tracking

The first-frame object crop is stored as a template.

For every new frame:

1. A search region is generated around the predicted object location.
2. Template matching is performed inside the search region.
3. The location with the highest similarity score is selected.
4. The similarity score becomes the tracker confidence.

### Why Template Matching?

Advantages:

* Works on arbitrary objects
* Does not require training data
* Lightweight implementation
* Easy GPU acceleration

Limitation:

* Sensitive to large appearance changes
* Can drift when background textures resemble the target

---

## Step 3: Constant Velocity Motion Model

The motion model assumes:

> The object continues moving at approximately the same velocity as observed in previous frames.

The model maintains:

* Current position
* Estimated velocity

For each frame:

```text
Predicted Position
=
Current Position
+
Velocity
```

The motion estimate provides continuity even when visual tracking becomes unreliable.

### Benefits

* Handles temporary tracking failures
* Smooths noisy measurements
* Reduces sudden jumps

---

## Step 4: Tracker-Motion Fusion

Both systems generate independent estimates.

### Visual Tracker

Provides:

* Bounding box
* Similarity confidence

### Motion Model

Provides:

* Predicted location
* Motion confidence

The fusion logic determines which estimate should be trusted more.

#### High Tracker Confidence

When visual confidence is strong:

```text
Tracker dominates
```

The final result follows the visual observation closely.

#### Low Tracker Confidence

When confidence drops:

```text
Motion Model dominates
```

The system temporarily relies on prediction until the tracker recovers.

This prevents sudden tracking failures.

---

# GPU Acceleration

The implementation automatically checks whether CUDA-enabled OpenCV is available.

If CUDA is detected:

* Template matching is executed on the GPU.
* Search operations become significantly faster.

Otherwise:

* The system gracefully falls back to CPU execution.

No code changes are required by the user.

---

# Output Format

For each frame, the system generates:

```json
{
  "frame_index": 25,
  "bbox": {
    "x": 2205.5,
    "y": 1020.3,
    "width": 240,
    "height": 270
  },
  "tracker_confidence": 0.92,
  "motion_model_confidence": 0.88,
  "trusted_output_source": "object_tracker"
}
```

---

# Design Decisions

### Why Not Use YOLO?

The problem statement assumes the user already provides the initial bounding box.

Therefore, object detection is not required.

The objective is tracking, not detection.

---

### Why Use a Motion Model?

Visual tracking alone often fails when:

* The object becomes partially occluded
* Motion blur occurs
* Similar background patterns appear

The motion model acts as a fallback mechanism.

---

### Why Hybrid Tracking?

A single tracker can drift.

A motion model alone cannot recover from accumulated errors.

Combining both approaches provides:

* Better robustness
* Smoother trajectories
* Reduced tracking loss

---

# Future Improvements

Several improvements can further enhance performance:

### Kalman Filter

Replace the basic constant velocity model with a Kalman Filter for more principled state estimation.

### Deep Feature Tracking

Replace template matching with deep visual embeddings.

### Multi-Scale Search

Improve handling of object size changes.

### Automatic Initialization

Integrate YOLO-based detection to automatically generate the initial bounding box.

### Multi-Object Tracking

Extend the architecture to support simultaneous tracking of multiple targets.

---

# Conclusion

This project demonstrates a complete hybrid tracking pipeline that combines appearance-based tracking with motion prediction. The system satisfies the requirements of tracking an arbitrary user-selected object while producing smooth and continuous trajectories across video frames. By incorporating GPU acceleration, confidence-based fusion, and motion prediction, the tracker remains reliable even in challenging scenarios where visual tracking alone may fail.
