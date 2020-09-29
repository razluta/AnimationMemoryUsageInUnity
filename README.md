# Animation Memory Usage in Unity [![License](https://img.shields.io/badge/License-MIT-lightgrey.svg?style=flat)](http://mit-license.org)
Best Practices for Optimizing Animation Memeory Usage - This repository attempts to best analyze all possible outcomes that could result in less than optimal memory usage when bringing in different types of 3D animations into Unity.  While animations are not typically the most memory intensive assets in a project, many 3D mobile games are very animation heavy and as such animation represents a significant part of the data that needs optimization.

## Conclusions
This section summarizes all conclusions reached after studying the data.

![](/Screenshots/WalkAnimation.gif)

### Summary
It is best to use complex rigs (as complex they need to be to achieve the animation in the DCC package) to generate simple single hierarchy rig skeletons and animations with the minimum amount of keys possible and import it in _Unity_ using the _Animation Compression_ set to **Optimal**. Use the _Animation Masking_ feature to further reduce the size of the animation in the build.

The delta between the most optimized and least optimized situation presented in this test, for the same animation, was over 500 KB. If the project has a conservative 20 animations per each character and 10 characters loaded at the same time, that could result in almost 100 MB of wasted runtime memory, which on a low end device such as the iPhone 6s represents a large chunk of the usable memory (and this is worrisome because for 3D projects textures typically present an ever bigger threat to excessive memory usage and could exceed the limits if the animations are also bulky).

### Detailed Explanation 
To best optimize and minimize the memory usage of an animation clip, the following are universally recommended:
1. The skeleton/rig used to create/define the [Unity Avatar](https://docs.unity3d.com/Manual/class-Avatar.html) should contain the rig and the mesh, but no animation. The image below showcases a full complex rig with its hierarchy on the left side and a "simplified"/single hierarchy rig (ready for export to Unity on the right side).

![](/Screenshots/RigInDCC.png)

Taken a step forward, the right side of the image above is continued in the image below where it exemplifies a skeleton .fbx file that only contains the model and the single hierarchy chain rig (as seen in _Unity_).

![](/Screenshots/SkeletonContents.png)

2. The _Avatar_ generated from the rig should be used for all animations that are supposed to share that rig, otherwise each animation will end up with its own avatar and will cost additional memory usage. To use an existent _Avatar_ for an animation, the user need to define it in the import options for the animations as demonstrated in the image below.

![](/Screenshots/ReuseAvatar.png)

3.  No single animation file should contain the mesh. Each animation should only contain the rig joints and the animation(s) on them (speaking strictly about the exported / published file, not the animation source file). As demonstrated in the examples in the repository, having the mesh in the animation file does not increase the animation size in memory in the built project (because only the animation data is used to generated the _Unity Animation Clip_), but it considerably increases the animation file size on disk. In this example, the size delta is about 300 KB per file. Multiply that by 50 animations per character, in a project of 100 characters, that is over 1.4 GB of wasted space. Additionally, having the mesh in the animation file presents the added risk of accidentally using that mesh in the game in a scene, at which point _Unity_ would have a duplicate of that mesh in memory and the actual runtime memory will also increase by 300 KB times the number of accidents. The image below exemplifies an animation .fbx file that only contains the single hierarchy chain rig with animation on it. It is important to note that the animation is not visible in the scene preview window when dragged directly in the scene because it needs to be applied on an already created avatar.

![](/Screenshots/AnimationContents.png)

Although not universally applicable, we recommend the following settings (unless advanced users have tested and profiled specialized branching from these standards):
1. _Anim. Compression_ should be set to **Optimal**. The three animation compression options are: "Off", "Optimal" and "Keyframe Reduction". In most cases, "Optimal" is the option best suited for the best animation compression with the least animation quality loss, with default settings. Fine tuning each setting for each animation may provide better results, but it is likely a solution that cannot scale for hundreds of animations. In this demo, almost across the board, using the "Optimal" setting provided significant Memory reduction. Below are a few of the examples:
- the short walk animation using keyed poses only and imported using "Optimal" compression was 17 KB <> using the "Keyframe Reduction" option, the same animation was 43.6 KB <> using the "Off" option (no compression), the same animation was 70 KB
- the long walk animation using all keys baked and imported using "Optimal" compression was 103 KB <> using the "Keyframe Reduction" option, the same animation was 293.6 KB <> using the "Off" option (no compression), the same animation was 614 KB
As observed in the examples, above, the difference can be 500 KB on a long (300 frames) animation, which can quickly add up.

![](/Screenshots/AnimationCompressionOptimal.png)

Additionally, here are some common mistakes when working with animation data coming from DCC packages.
1. Each of the rig and the animation .fbx files should only contain what is commonly referred to as the single hierarchy joint chain. Typically, animation rigs are very complex in the DCC package to allow the animator extensive control over the "base" skeleton. This complex setup usually contains: IK (inverse kinematics) controllers and setup, FK (forward kinematic) controllers and setup, helper nodes, etc. Some of these nodes are nodes that are supported by the .fbx format and when _accidentally_ exported, will create empty _GameObjects_ in the _Unity_ hierarchy, but will still contain the animation data and waste memory resources. We recommend exporting **only** the single hierarchy joint chain for both the skeleton/rig (used to generate the Avatar), while for the animation, we recommend exporting only joints that are skinned to the mesh and used directly in the motion (no helper joints) - this process typically requires a standardization (naming or expectected content/nodes) across rigs and an automated solution that can differentiate between what is needed in the game engine and what is not needed. Below is some data from this analysis that backs up this claim:
- when exported with only the single hierarchy rig and imported as a _Humanoid_, the skeleton was 42.5 KB <> when exported with the full rig and imported as a Humanoid, the skeleton was 46.3 KB
- when exported with only the single hierarchy rig and imported with optimal compression, the short walk animation with all keys baked was 17 KB <> when exported with only the full rig and imported with optimal compression, the short walk animation with all keys baked was 70 KB
2. Animations should not be set to bake all keyframes on export or be pre-baked in the original source animation file. If an animation is pre-baked on all keyframes in the original source file, it is very difficult to update and iterate on the file. Additionally, it automatically means the exported result will be baked as well. Similarly, if an animation is set to be baked on export, all keyframes in range will have an animation key resulting in a large animation data set. Unity has ways of compressing the data that is present in the animation clip (for example the [Animation Compression](https://docs.unity3d.com/Manual/class-AnimationClip.html) import settings), but it is even better if only the needed keyframes are kept. If the artistic quality is not met because of how _Unity_ interprets the animation curves differently than the DCC packages, the user can either selectively bake a problematic section of the animation (as opposed to baking the entire animation) or disable the [_Resample Curves_](https://docs.unity3d.com/Manual/class-AnimationClip.html) option (the option is enabled by default - if disabled it will "keep animation curves as they were originally authored". 

Lastly, even if rigs and animations are imported into Unity with garbage data (for example controllers or helper nodes that do not affect the animation) and the user does not have access to improve the source file or cannot improve the soure file, Unity offers the [Animation Mask](https://docs.unity3d.com/Manual/AnimationMaskOnImportedClips.html) as a way to remove this data during the build process. An _Animation Mask_ is a Unity serialized asset that is linked to a skeleton/avatar and can be used and re-used to occlude any animation that uses the avatar. As seen in the image above, the masking can be done visually on a humanoid or by directly ignoring a hierarchy of transforms. It is important to note that, during the build process, the masked animation will be removed from the clip and will result in an lighter animation clip (memory wise) - but all animation in clip will be lost. 

![](/Screenshots/AnimationMask.PNG)

Most of these rules apply for most projects that need to minimize memory usage for animation clips. For additional information and the technical explanation of these conclusions were reached, please take a look at the **Raw Data** and **Legend** sections below. 

## Special Notes
All the data was captured at runtime from a development build running on an iPhone 6s (released in 2015).

## Raw Data
The data table below contains the raw data of memory usage, as extracted from the Unity Memory Profiler. All animations were applied to two game objects and captured together in a single **Memory Snapshot**. The three columns with numbers represent, in order (left to right):
- the in-memory size of the rig/animation when imported as a Humanoid
- the in-memory size of the rig/animation when imported as Generic
- the .fbx file size on disk

| Animation Clip Name                                                         | Humanoid In Memory Size (KB) | Generic In Memory Size (KB) | On Disk Size (KB) |
|-----------------------------------------------------------------------------|:----------------------------:|:---------------------------:|:-----------------:|
| PolyBod_ShOnly_Skeleton                                                     |             42.5             |             28.5            |        361        |
| PolyBod_ShOnly_Anim_Walk_Short_KeyedPosesOnly_CompressionOptimal            |              17              |             25.9            |        750        |
| PolyBod_ShOnly_Anim_Walk_Short_KeyedPosesOnly_WithMesh_CompressionOptimal   |              17              |              -              |        1033       |
| PolyBod_ShOnly_Anim_Walk_Short_KeyedPosesOnly_CompressionOff                |              70              |            199.4            |        750        |
| PolyBod_ShOnly_Anim_Walk_Short_KeyedPosesOnly_CompressionKeyframeReduction  |             43.6             |             42.5            |        750        |
|                                                                             |                              |                             |                   |
| PolyBod_ShOnly_Anim_Walk_Short_Baked_CompressionOptimal                     |              17              |             26.8            |        1274       |
| PolyBod_ShOnly_Anim_Walk_Short_Baked_CompressionOff                         |              70              |            190.7            |        1274       |
| PolyBod_ShOnly_Anim_Walk_Short_Baked_CompressionKeyframeReduction           |             43.6             |             49.5            |        1274       |
|                                                                             |                              |                             |                   |
| PolyBod_ShOnly_Anim_Walk_Long_KeyedPosesOnly_Optimal                        |             104.7            |            141.4            |        1865       |
| PolyBod_ShOnly_Anim_Walk_Long_KeyedPosesOnly_CompressionOff                 |              614             |             1843            |        1865       |
| PolyBod_ShOnly_Anim_Walk_Long_KeyedPosesOnly_CompressionKeyframeReduction   |             293.6            |            208.6            |        1865       |
|                                                                             |                              |                             |                   |
| PolyBod_ShOnly_Anim_Walk_Long_Baked_CompressionOptimal                      |              103             |            126.4            |       10193       |
| PolyBod_ShOnly_Anim_Walk_Long_Baked_CompressionOff                          |              614             |             1740            |       10193       |
| PolyBod_ShOnly_Anim_Walk_Long_Baked_CompressionKeyframeReduction            |             293.6            |            265.3            |       10193       |
|                                                                             |                              |                             |                   |
| PolyBod_FullRig_Skeleton                                                    |             46.3             |             63.2            |        448        |
| PolyBod_FullRig_Anim_Walk_Short_KeyedPosesOnly_CompressionOptimal           |              17              |             65.7            |        1493       |
| PolyBod_FullRig_Anim_Walk_Short_KeyedPosesOnly_CompressionOff               |             69.4             |            424.4            |        1493       |
| PolyBod_FullRig_Anim_Walk_Short_KeyedPosesOnly_CompressionKeyframeReduction |             39.1             |            111.3            |        1493       |
|                                                                             |                              |                             |                   |
| PolyBod_FullRig_Anim_Walk_Short_Baked_CompressionOptimal                    |              70              |             65.6            |        3135       |
| PolyBod_FullRig_Anim_Walk_Short_Baked_CompressionOff                        |              70              |            411.8            |        3135       |
| PolyBod_FullRig_Anim_Walk_Short_Baked_CompressionKeyframeReduction          |             41.4             |            143.4            |        3135       |
|                                                                             |                              |                             |                   |
| PolyBod_FullRig_Anim_Walk_Long_KeyedPosesOnly_Optimal                       |             104.7            |            449.9            |        4156       |
| PolyBod_FullRig_Anim_Walk_Long_KeyedPosesOnly_CompressionOff                |              614             |             3993            |        4156       |
| PolyBod_FullRig_Anim_Walk_Long_KeyedPosesOnly_CompressionKeyframeReduction  |             293.6            |             700             |        4156       |
|                                                                             |                              |                             |                   |
| PolyBod_FullRig_Anim_Walk_Long_Baked_CompressionOptimal                     |             104.9            |            472.9            |       21453       |
| PolyBod_FullRig_Anim_Walk_Long_Baked_CompressionOff                         |              614             |             3993            |       21453       |
| PolyBod_FullRig_Anim_Walk_Long_Baked_CompressionKeyframeReduction           |             359.4            |             1400            |       21453       |

Below are some of the screenshots of the raw data, directly from the Unity Memory Profiler.
The image below is a screenshot of the raw data in the Memory Profiler when using a Humanoid import.

![](/Screenshots/DeviceMemoryUsage_Humanoid.PNG)

The image below is a screenshot of the raw data in the Memory Profiler when using a Generic import.

![](/Screenshots/DeviceMemoryUsage_Generic.png)

_It is important to note that the Humanoid animations seem to be better optimized for memory than the Generic animations and that is because Unity, at build time, scraps the data that doesn't map to the Humanoid avatar and it the process reduces memory usage. The same optimization result can be achieved on the Generic animations by either improving the import data in the DCC package or by using Animation Masking in Unity._

## Legend
The .fbx rig and animation files were generated using Autodesk Maya 2020, but the same measurements and rules apply for rigs and animations generated from any DCC packages. The rig was generated using Maya's HumanIK System (HIK), but the same concepts apply to any rig system.

Below is a quick explanation of what each file contains:
- **PolyBod_ShOnly_Skeleton** = The single hierarchy joints only rig + mesh
- **PolyBod_ShOnly_Anim_Walk_Short_KeyedPosesOnly_CompressionOptimal** = The single hierarchy joints only rig + no mesh + short walk animation with key poses only
- **PolyBod_ShOnly_Anim_Walk_Short_KeyedPosesOnly_WithMesh_CompressionOptimal** = The single hierarchy joints only rig + no mesh + short walk animation with key poses only - imported with Animation Compression set to Optimal
- **PolyBod_ShOnly_Anim_Walk_Short_KeyedPosesOnly_CompressionOff** = The single hierarchy joints only rig + no mesh + short walk animation with key poses only - imported with Animation Compression set to Off
- **PolyBod_ShOnly_Anim_Walk_Short_KeyedPosesOnly_CompressionKeyframeReduction** = The single hierarchy joints only rig + no mesh + short walk animation with key poses only - imported with Animation Compression set to Keyframe Reduction

- **PolyBod_ShOnly_Anim_Walk_Short_Baked_CompressionOptimal** = The single hierarchy joints only rig + no mesh + short walk animation with all keyframes in range baked - imported with Animation Compression set to Optimal
- **PolyBod_ShOnly_Anim_Walk_Short_Baked_CompressionOff**  = The single hierarchy joints only rig + no mesh + short walk animation with all keyframes in range baked - imported with Animation Compression set to Off
- **PolyBod_ShOnly_Anim_Walk_Short_Baked_CompressionKeyframeReduction** = The single hierarchy joints only rig + no mesh + short walk animation with all keyframes in range baked - imported with Animation Compression set to Keyframe Reduction

- **PolyBod_ShOnly_Anim_Walk_Long_KeyedPosesOnly_Optimal** = The single hierarchy joints only rig + no mesh + long walk animation with key poses only  - imported with Animation Compression set to Optimal
- **PolyBod_ShOnly_Anim_Walk_Long_KeyedPosesOnly_CompressionOff** = The single hierarchy joints only rig + no mesh + long walk animation with key poses only - imported with Animation Compression set to Off
- **PolyBod_ShOnly_Anim_Walk_Long_KeyedPosesOnly_CompressionKeyframeReduction** = The single hierarchy joints only rig + no mesh + long walk animation with key poses only - imported with Animation Compression set to Keyframe Reduction

- **PolyBod_ShOnly_Anim_Walk_Long_Baked_CompressionOptimal** = The single hierarchy joints only rig + no mesh + long walk animation with all keyframes in range baked - imported with Animation Compression set to Optimal
- **PolyBod_ShOnly_Anim_Walk_Long_Baked_CompressionOff**  = The single hierarchy joints only rig + no mesh + long walk animation with all keyframes in range baked - imported with Animation Compression set to Off
- **PolyBod_ShOnly_Anim_Walk_Long_Baked_CompressionKeyframeReduction**  = The single hierarchy joints only rig + no mesh + long walk animation with all keyframes in range baked - imported with Animation Compression set to Keyframe Reduction

- **PolyBod_FullRig_Skeleton** = The full hierarchy joints rig (controllers, dummy nodes, single hierarchy rig) + mesh
- **PolyBod_FullRig_Anim_Walk_Short_KeyedPosesOnly_CompressionOptimal** = The full hierarchy rig + no mesh + short walk animation with key poses only - imported with Animation Compression set to Optimal
- **PolyBod_FullRig_Anim_Walk_Short_KeyedPosesOnly_CompressionOff** = The full hierarchy rig + no mesh + short walk animation with key poses only  - imported with Animation Compression set to Off
-**PolyBod_FullRig_Anim_Walk_Short_KeyedPosesOnly_CompressionKeyframeReduction** = The full hierarchy rig + no mesh + short walk animation with key poses only - imported with Animation Compression set to Keyframe Reduction

- **PolyBod_FullRig_Anim_Walk_Short_Baked_CompressionOptimal** = The full hierarchy rig + no mesh + short walk animation with all keyframes in range baked  - imported with Animation Compression set to Optimal
- **PolyBod_FullRig_Anim_Walk_Short_Baked_CompressionOff** = The full hierarchy rig + no mesh + short walk animation with all keyframes in range baked - imported with Animation Compression set to Off
- **PolyBod_FullRig_Anim_Walk_Short_Baked_CompressionKeyframeReduction**  = The full hierarchy rig + no mesh + short walk animation with all keyframes in range baked - imported with Animation Compression set to KeyframeReduction

- **PolyBod_FullRig_Anim_Walk_Long_KeyedPosesOnly_Optimal**  = The full hierarchy rig + no mesh + long walk animation with only key poses  - imported with Animation Compression set to Optimal
- **PolyBod_FullRig_Anim_Walk_Long_KeyedPosesOnly_CompressionOff**  = The full hierarchy rig + no mesh + long walk animation with only key poses  - imported with Animation Compression set to Off
- **PolyBod_FullRig_Anim_Walk_Long_KeyedPosesOnly_CompressionKeyframeReduction**  = The full hierarchy rig + no mesh + long walk animation with only key poses  - imported with Animation Compression set to Keyframe Reduction

- **PolyBod_FullRig_Anim_Walk_Long_Baked_CompressionOptimal** = The full hierarchy rig + no mesh + long walk animation with all keyframes in range baked - imported with Animation Compression set to Optimal
- **PolyBod_FullRig_Anim_Walk_Long_Baked_CompressionOff** = The full hierarchy rig + no mesh + long walk animation with all keyframes in range baked - imported with Animation Compression set to Off
- **PolyBod_FullRig_Anim_Walk_Long_Baked_CompressionKeyframeReduction**  = The full hierarchy rig + no mesh + long walk animation with all keyframes in range baked - imported with Animation Compression set to Keyframe Reduction

All results and data obtained have been collected from the project generated with the data in this repository and can be replicated by profiling a development build generated from the project provided in the repository.
