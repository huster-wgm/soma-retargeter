# A3 shoulder/wrist continuity — clip 19

Left: before. Right: after.

![Before and after](before-after.gif)

[Before: original SOMA vs A3 (MP4)](before.mp4) · [After: original SOMA vs A3 (MP4)](after.mp4)

These are the existing renders from the single-clip experiment, not a new dataset-wide validation.
The GIF crops the robot half of each original video for a side-by-side preview;
the MP4 files retain the original SOMA/robot comparison. No raw motion data is included.

- Source: BONES-SEED SOMA Uniform `221129/reach_jump_R_004__A077.bvh`, 424 frames at 120 Hz.
- Videos: 107 frames at 30 fps; first output frame is t=0.
- Model: project-specific Agibot A3 MJCF/calibration, not the bundled upstream A3 URDF.
- Change: add a residual `2 * (q_current - q_previous_base_solution)` on both shoulders, elbows and wrists.
- Existing 24 IK iterations, static smooth filter, limits and contact/post-processing settings were retained.
- Fast arm lowering is smoother but has slight lag and greater target-tracking error. See the linked PR for the numerical audit and scope.

## Credits

**Motion Data by [Bones Studio](https://bones.studio/).**
Dataset: [bones-studio/seed](https://huggingface.co/datasets/bones-studio/seed).
Use of the underlying dataset is subject to the [BONES-SEED Dataset License Agreement](https://bones.studio/info/seed-license).
SOMA model and renderer: [NVIDIA/soma-retargeter](https://github.com/NVIDIA/soma-retargeter).
Robot: Agibot A3, sourced from [AgibotTech/sonic_for_a3](https://github.com/AgibotTech/sonic_for_a3).
The renders do not redistribute BVH/CSV motion files or model/mesh assets.
