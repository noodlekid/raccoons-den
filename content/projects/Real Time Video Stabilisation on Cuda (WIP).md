---
title: Real Time Video Stabilisation on Cuda (WIP)
tags:
  - cuda
  - parallel-processing
  - gpgpu
  - design-team
---
## Problem

*What problem is this project trying to solve?*

The rover's camera produces unstable footage during operation due to mechanical vibrations transmitted through the chassis. This instability severely impairs teleoperation by making precise control and navigation difficult. Operators experience higher cognitive load and reduced effectiveness when working with shaking video feeds.

**TL;DR: Shitty shaky video feed makes it difficult to navigate the rover visually.**

Current mechanical damping proves insufficient across the frequency spectrum encountered during terrain traversal. While the camera hardware performs well statically, the video pipeline lacks software based motion compensation.

## Requirements

### System Constraints

The rover operates on an NVIDIA Jetson platform (Nvidia Jetson Orin AGX Dev kit 32GB) with available GPU capacity for video processing. The current video pipeline uses GStreamer framework for processing and streaming. Any solution must integrate with the rover's existing control and telemetry systems.

![[nvidia-jetson-orin-agx.png]]
*our rovers compute platform (Jetson Orin AGX)*

### Performance Requirements

Maximum acceptable additional latency is 50ms. The solution must maintain real-time processing at 30 FPS minimum for 1280x720 resolution video. No consecutive or bursty frame drops although a few frame drops (<1 drop every 60 frames) is acceptable. 

These requirements are in place for purpose of integration into our video Tx/Rx pipeline. In order to be able get the most of our compression, forward error correction and other features meeting these requirements is crucial. As far as the quality of the frames goes, you gotta be able to make out things in your frame or else it's kind of useless for a human operator.

### Operational Requirements

The solution must function reliably across varying terrain conditions and vibration frequencies during missions up to 1 hour. System failures must not interrupt video feed availability; degraded operation is preferable to complete loss. Diagnostic data is not strictly required but boy would it be helpful for troubleshooting down the line. No one writes perfect code. 

## Goals

### Primary Objectives

Achieve smooth video output by eliminating high frequency vibrations while preserving intentional camera movements. Reduce operator cognitive load to improve rover operation. Integrate with existing systems as a drop-in enhancement.

### Success Metrics

**Darren/Operator does not get a seizure when operating the rover.**

## Design Exploration

| Criteria | Passive Dampening | Active Gimbal (Mech/Elec) | Software EIS (Vision Only) | Hybrid EIS (IMU + Vision) |
|---|---|---|---|---|
| Performance | Low to Medium. Only helps with high-frequency jitter. | Very High. best stabilization against all motion types. | Medium. Effective, but suffers from latency and FoV loss. | High. Near-gimbal performance without mechanical complexity.  |
| BOM Cost | Very Low. A few dollars for mounts/elastomers. | Very High. Adds significant cost per unit. | Zero. Purely a software addition. | Low. Adds the cost of a single IMU component. |
| Dev Effort | Medium. Requires mechanical design and physical testing/tuning. | High. Involves mechanical design, motor control software, and tuning. | Medium. Requires implementing and tuning the vision pipeline. | High. Requires complex sensor fusion (EKF) and precise calibration. |
| Negative Impact on entire Rover | Low. Adds some weight/bulk at the camera mount. | Very High. The heaviest, bulkiest option. | Zero. No physical impact. | Very Low. Only adds a tiny, low-power IMU. |
| Key Risk | Ineffective against larger, low-frequency movements. | High unit cost and development complexity for Elec, Mech and SW. | Added latency to video feed | Calibration complexity. Getting the spatio-temporal alignment right is critical. |

### Pure Software Stabilization

GPU visual processing using NVIDIA VPI libraries within a custom GStreamer element. Features are tracked across frames to estimate motion, which is filtered and compensated through image warping.

**Advantages**: No hardware changes required. Fully configurable. Maximizes available GPU capacity. This shit is **FREE**.  We have memebers on the team that have enough knowledge about GStreamer to review code effectievly. Good GPU based libraries for this application exist.

**Limitations**: Requires 5-10% frame cropping. Cannot handle extreme motion. Increased latency in GStreamer pipeline. 

**Risks:** It's too slow!

### IMU-Assisted Stabilization

Combines inertial measurements with visual processing for predictive compensation. IMU data anticipates motion before it affects the video stream.

Go-Pro and popular action cameras will implement such a strategy, they do not have that much processing power but they are able to pull it off pretty well.

**Advantages**: Better rapid motion handling. Lower latency potential.

**Limitations**: Requires IMU integration. Complex calibration. Extended development time.

**Risks:** Integration takes too long before key deadlines. 

### Mechanical-Software Hybrid

Combines passive dampers with software compensation for vibration reduction.

**Advantages**: Reduced software burden.

**Limitations**: Requires hardware changes. Increases weight and complexity. Added risk of failure for mechanical components.

### Full Gimbal

Uses a few motors and mount to stabilise the camera itself to quickly correct for undesired motion.

Imagine one of those *vlogger-eqsque* gimbals integrated on the rover. An off the shelf might be considered, really depends on the cost, and how easily it integrates. We'd really want to use our own cameras on it, not something built in as you see in the diagram below.

![[gimbal.png]]

**Advantages:** no latency added to GStreamer pipeline. Very fast.

**Limitations:** Good quality components are expensive. Firmware complexity is high. Synchronisation between electrical, mechanical and software teams is not so great. More sophisticated wiring required.

## Prime Candidate - Pure Software Stabilisation

### Selection Rationale

Pure software stabilisation best fits our constraints. It requires no hardware modifications, uses existing libraries and provides immediate (eh hopefully) deployment capability. The approach directly solves the instability problem while maintaining compatibility with existing systems.

### Architecture Overview

A custom GStreamer element encapsulates all stabilization logic as a plugin. The element maintains a short rolling frame buffer in GPU memory, tracks features between frames, estimates motion vectors, applies Kalman filtering for smoothing, and warps frames to compensate for unwanted motion.

The zero-copy NVMM architecture ensures minimal latency. Runtime configuration allows parameter tuning without restart.

**TBD:** Bypass feature in case element shits itself, or if we do not need video stabilisation.

### Implementation Phases

**Phase 1**: Core stabilization pipeline with basic motion compensation.

**Phase 2**: Performance optimization and latency reduction.

**Phase 3**: System integration and field testing.

**Future**: IMU integration as optional upgrade.

### Key Design Elements

<div class="interrupt">

**INTERRUPT! A bit on Zero-Copy Buffers in NVMM**

In a naive approach to designing a video pipeline you might continuously copy frame buffers back and forth between the CPU and GPU. But why would we ever want to do that in the first place anyway?

The answer is that not everything image-related is meant to "just work" on the GPU or hardware accelerators. In optimized video pipelines built around the GStreamer framework you'll often see these things called "frame buffers" being passed around between different processing units - CPU, GPU, VPU, you name it. You may already see the problem with this... **memory transfer is generally slow**

So in an ideal world (the one we're building) the data just stays put in memory the whole time, and we pass around *references* to where the frame lives instead of shuffling the actual pixels around. This is where the name "zero-copy" comes from - we're not copying frames between processors, just updating pointers. The DMA controller in the diagram? That's the hardware doing the heavy lifting here, moving data around without bothering the CPU.

![[DMA.png]]

Now here's the cool part on these Jetson platforms - the CPU and GPU actually share the same DRAM anyway (check out that unified memory architecture in the diagram). So zero-copy really means we're avoiding redundant copies within that *same* memory space. As you may imagine this is **much faster** keeping our video pipelines snappy since we want minimal latency between processing and streaming video between our rover and base station.

But as I alluded to before, not everything is built to run on GPU or these specialized accelerators! So any element we build needs to play nice with this shared memory programming model.

Anyway, back to main routine...

*return;*

</div>


The core design is composed of 4 key functional components and 1 configuration component. These components are then made up of their own parts with clear inputs and outputs (interfaces). These elements allow for swappable algorithms, and more readable code due to its modularity.

![Enter image alt description](0R3_Image_1.png)

#### The Motion
**Detecting all that motion**
![Enter image alt description](9bU_Image_2.png)

The summary is that I want to use [Harris Corners](https://docs.nvidia.com/vpi/algo_harris_corners.html) **because it is rotation invariant and the VPI implementation runs entirely on GPU**

[Lucas-Kanade](https://docs.nvidia.com/vpi/group__VPI__OpticalFlowPyrLK.html) is apparently good at small motion between frames and tracks feature movement. It excels at translational motion which we may expect the most. This may be less effective if we have a lot of rotational motion (i.e. tilt) which with our architecture we could swap out **Lucase-Kanade** with a different algo. 

**RANSAC** can deal with a lot of outliers which seems to be a common strategy to filter outliers in computer vision. A moving object in a frame might classify as an outlier so RANSAC would filter that motion vector out leaving only global motion.



#### Stabilisation Control
**Kalman filter separates intentional movement from vibration noise.**
![Enter image alt description](KGs_Image_3.png)

Raw motion vectors from our motion estimator will take into account almost all types of motion. So the solution to this is to filter out motion that we think is intentional (i.e. operator purposefully rotating the rover or the camera) and keep motion that is undesired (i.e. vibrations) so that we can correct it later on.

The Kalman filter effectively predicts what the motion should be so that we can find these differences in motion to filter out. For example a small difference from the prediction may indicate that the movement was intentional, whereas a large difference from the prediction is likely a vibration so that is something we want to compensate for. This filter would need to be tuned in order to make these predictions correctly. A simple “threshold” would not be sufficient as vibrations/unintended movements often do not have consistent velocity, whereas intentional movements do maintain velocity making it easy to filter and predict that intentional movement (and thereby preserve it)

#### The Transformation
**the part that un-wobbles the stream**
![Enter image alt description](5sI_Image_4.png)

**Configuration Interface**: GStreamer properties for strength, threshold, margin (independent horizontal and vertical), and performance mode (😎)

TBD, but it'll give us a way to control parameters at run time depending on stuff like vibration intensity, large movement, and helpful for debugging and tuning the stabilisation.

---

# Implementation Time (WIP)