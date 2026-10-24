# Optional joint continuity

Add `joint_continuity_weights` to a robot's retargeter configuration to penalize
large changes in selected scalar joint coordinates between input frames:

```json
{
  "joint_continuity_weights": {
    "left_shoulder_pitch_joint": 2.0,
    "left_shoulder_roll_joint": 2.0,
    "left_shoulder_yaw_joint": 2.0,
    "left_elbow_joint": 2.0,
    "left_wrist_roll_joint": 2.0,
    "left_wrist_pitch_joint": 2.0,
    "left_wrist_yaw_joint": 2.0,
    "right_shoulder_pitch_joint": 2.0,
    "right_shoulder_roll_joint": 2.0,
    "right_shoulder_yaw_joint": 2.0,
    "right_elbow_joint": 2.0,
    "right_wrist_roll_joint": 2.0,
    "right_wrist_pitch_joint": 2.0,
    "right_wrist_yaw_joint": 2.0
  }
}
```

Names must match the target's revolute/prismatic joint names. Omit this field or
use `{}` to keep the existing behavior. No bundled robot preset is changed.
Nonpositive/nonfinite weights, ambiguous/unknown names, and non-scalar joints
are rejected when the objective is constructed.

For each selected joint, the additional residual is
`weight * (q_current - q_previous_base_solution)`. The least-squares cost squares
this residual. The reference is copied once before each frame's base IK solve
and held fixed throughout all solver iterations and candidate evaluations.
Each animation in the batch has its own reference; expanded candidate batches
use Newton's `problem_idx` mapping. Initialization/stabilization and the first
output frame run without this prior. Every `execute()` builds a new objective,
so state is not carried from an earlier animation batch.

This is a soft **per-frame** prior, not a hard speed limit. Its effect depends
on the input sample rate: retune the weights when resampling. Coordinates are
in radians (revolute) or metres (prismatic); this API intentionally excludes
floating and quaternion joints. Bounded joint coordinates are not wrapped at
2π. The existing joint-limit, contact and limb-stabilization stages remain in
place; the new prior does not add collision avoidance or dynamic feasibility.
It references the previous **base-pass** solution, not the final post-processed
or clamped output. Select joints unaffected by downstream corrections (such as
arms in the current A3 setup); it is not an output-continuity guarantee for feet
or other coordinates subsequently changed by post-processing.

Larger weights reduce abrupt changes but can introduce lag and tracking error.
Validate these tradeoffs on representative motions before changing a preset.

## A3 single-clip experiment

The motivating experiment used BONES-SEED SOMA Uniform
`221129/reach_jump_R_004__A077.bvh`, 424 source frames at 120 Hz, upstream
`cbdb2b4`, Newton 1.3.0 and Warp 1.16.0. It used a project-specific A3 MJCF
from Agibot's `sonic_for_a3` assets, with calibration adapted to that model,
**not the bundled upstream A3 URDF**. Its XML SHA256 was
`797f80caf29970a6d37fc7022f03dd1d78d0d5e8898b37a806e926b2528882e2`.
The paired runs used the same model, targets, 24 IK iterations, static smooth
filter, and contact/post-processing settings. The only change was adding the
above 14 arm residuals with weight 2. These are measurements from the original
experimental implementation of the same prior, not a claim that every upstream
asset or dataset exhibits this behavior.

Peak absolute adjacent-frame joint change in **0.90–1.05 seconds**:

| Right joint | Baseline | Continuity |
|---|---:|---:|
| Shoulder pitch | 11.20° | 2.93° |
| Shoulder roll | 14.72° | 7.07° |
| Shoulder yaw | 17.46° | 6.15° |
| Wrist roll | 12.49° | 4.26° |
| Wrist pitch | 10.50° | 2.53° |
| Wrist yaw | 6.24° | 1.17° |

At source frame 115 the shoulder pitch/yaw world axes had dot product 0.995,
and the 3-axis rotational Jacobian condition number was about 20. The jump
was already present after base IK; the post-processing stage changed these
arm coordinates by zero. Increasing IK iterations from 24 to 96 increased
peak shoulder-yaw change to 50.32° in the same window. These observations are
consistent with sensitivity near a shoulder singularity; they do not establish
that singularity is the only cause of retargeting error. The existing
`smooth_joint_filter` is a static near-limit penalty, not a temporal prior.

The tradeoff is visible: full-clip mean right-wrist error relative to the
calibrated IK target changed from **16.53 to 17.24 cm** for position and
**0.27 to 2.32°** for orientation. In the event window those means were
37.59 → 40.34 cm and 1.13 → 7.48°. The original tracking error is substantial;
reducing joint discontinuity does not resolve target calibration or reachability.
All 424 output frames were finite and joint limits were respected within
floating-point precision. A MuJoCo FK contact query using the original XML
masks found no self-penetration, but existing ground penetration up to about
1.19 cm remained. This is not a dynamics rollout or proof of collision freedom.

[Before/after videos and credits](EVIDENCE_LINK)

The videos are the previously recorded comparison, 107 frames at 30 fps
(3.57 seconds); each shows original SOMA on the left and the robot on the right.
The numerical audit uses the 120 Hz output, not the downsampled video.

**Motion Data by [Bones Studio](https://bones.studio/).** The underlying data
is subject to the [BONES-SEED license](https://bones.studio/info/seed-license).
Raw BVH, motion CSV and human/robot asset files are not redistributed here.
SOMA/renderer: NVIDIA soma-retargeter; robot: Agibot A3.

## Tests

```bash
python -m unittest discover -s tests -v
```

Tests exercise a floating-root model (different coordinate/DoF indices),
expanded candidate batches, finite-difference Jacobian agreement, reference
snapshot isolation, warmup/frame lifecycle, invalid joint selections, and an
actual Newton IK solve with a known tracking/continuity optimum. CPU coverage
is provided; the objective uses device arrays and in-place reference updates
so its interface is compatible with the existing CUDA-graph pass.
