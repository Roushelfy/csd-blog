+++
title = "VR-Doh: Hands-On 3D Modeling in VR with Physics-Based Simulations"
date = 2025-11-24

[taxonomies]
areas = ["Graphics"]
tags = ["virtual-reality", "3D-modeling", "physics-simulation", "HCI", "gaussian-splatting"]

[extra]
author = {name = "Rouslan Fikhou", url = "https://github.com/Roushelfy" }
committee = [
    {name = "Committee Member 1", url = "https://www.cs.cmu.edu/"},
    {name = "Committee Member 2", url = "https://www.cs.cmu.edu/"},
    {name = "Committee Member 3", url = "https://www.cs.cmu.edu/"}
]
+++

Have you ever wished you could sculpt a 3D model as naturally as shaping clay with your bare hands? Traditional 3D modeling software like Blender and Maya, while powerful, often require significant expertise---users must master complex interfaces, learn specialized techniques for vertex manipulation, and spend considerable time fine-tuning their creations. This steep learning curve has long been a barrier preventing creative individuals from expressing their ideas in three dimensions.

In this blog post, I introduce **VR-Doh**, an immersive virtual reality system that transforms 3D modeling into an intuitive, hands-on experience. By combining physics-based simulation with natural hand interactions, VR-Doh allows users to create and edit 3D content as if they were molding real clay---no specialized training required.

![VR-Doh Teaser showing hands-on 3D modeling in virtual reality](./VR-Doh-Teaser.png)
**Figure 1:** *VR-Doh enables intuitive 3D modeling through physics-based simulations and natural hand interactions in virtual reality. Users can deform objects, edit realistic 3D Gaussian Splatting representations, and create new models from scratch.*

# The Challenge: Making 3D Modeling Accessible

The demand for 3D digital content has never been higher. From video games and movies to product design and virtual environments, 3D models are everywhere. Yet creating these models remains a task largely reserved for professionals.

Traditional 3D modeling tools impose significant barriers for novice users:

- **Steep learning curves**: Users must learn complex interfaces and specialized techniques
- **Unintuitive interactions**: Manipulating 3D objects through a 2D screen with a mouse feels disconnected from real-world creation
- **Iterative fine-tuning**: Achieving desired results often requires careful observation and repeated adjustments

Meanwhile, Virtual Reality (VR) has emerged as a promising platform for 3D content creation. Commercial tools like Shapelab and Gravity Sketch have made strides in VR modeling, but they still rely primarily on geometric approaches---drawing curves to form surfaces, manipulating vertices, and using procedural operations. These methods, while effective, still demand significant skills and don't fully leverage what VR does best: immersive, natural interactions.

> What if modeling in VR could feel as intuitive as working with physical clay?

<p></p>

This question led us to develop VR-Doh, a system that bridges the gap between physical intuition and digital creation.

# Our Approach: Physics Meets Virtual Reality

VR-Doh takes a fundamentally different approach to 3D modeling. Instead of treating virtual objects as collections of polygons to be manipulated geometrically, we simulate them as **physically deformable materials** that respond to your hands just like real clay would.

The core insight is simple: humans have a lifetime of experience manipulating physical objects. We intuitively understand how materials deform when we squeeze, stretch, or poke them. By bringing realistic physics into VR, we allow users to leverage this existing knowledge rather than learning entirely new skills.

## System Overview

VR-Doh integrates three key components:

1. **Real-time physics simulation** using the Material Point Method (MPM)
2. **High-quality rendering** through 3D Gaussian Splatting and mesh reconstruction
3. **Natural hand-based interactions** combining contact and gesture inputs

![VR-Doh Pipeline](./Technical_Framework.png)
**Figure 2:** *VR-Doh Pipeline. Our interactive system enables hands-on 3D modeling in VR through contact- and gesture-based inputs. The pipeline integrates MPM to simulate realistic elastoplastic deformations and supports rendering using both 3D Gaussian Splatting and meshes.*

The system runs in a continuous loop: it captures hand positions and gestures from the VR headset, performs physics simulation to compute how objects deform in response to user input, and renders the updated scene at interactive frame rates.

# Simulation: The Material Point Method

At the heart of VR-Doh lies the **Material Point Method (MPM)**, a hybrid simulation technique that elegantly handles large deformations and complex material behaviors. Originally developed for computational mechanics, MPM has gained popularity in computer graphics for simulating challenging phenomena like snow, sand, and elastoplastic materials.

## Why MPM?

Traditional simulation methods face trade-offs that make them unsuitable for interactive clay-like modeling:

- **Mesh-based methods** (like finite element analysis) struggle with large deformations because their mesh elements become distorted
- **Pure particle methods** can be noisy and may not accurately capture material stiffness
- **Grid-based methods** require very fine resolutions to capture detailed geometries

MPM combines the best of both worlds. It represents materials as particles that carry mass, velocity, and deformation information, while using a background grid to efficiently compute forces and handle collisions. This hybrid approach naturally handles:

- **Large elastoplastic deformations** (like squeezing a sphere into a pancake)
- **Material self-collision** (like folding clay onto itself)
- **Contact between multiple objects** (like stacking components)

## Making MPM Real-Time

Standard MPM implementations are computationally expensive---far too slow for the responsive feedback required in VR. We introduce several key optimizations to achieve real-time performance.

### Particle-Level Collision Handling

One challenge in interactive simulation is ensuring that virtual objects don't interpenetrate with the user's hands. Standard MPM applies boundary conditions on the simulation grid, but this can miss fine geometric details when hands make intricate contact with objects.

We solve this with a post-processing step that operates directly on particles. After the standard grid-based update, we check each particle against the hand geometry. Any particle that ends up inside a hand is projected back to the surface, and its velocity is adjusted to match the hand's motion. This ensures clean contact even at moderate grid resolutions.

![Particle-Level Collision Handling](./Adjust_Particle.png)
**Figure 3:** *Comparison of MPM boundary conditions with and without particle-level collision handling. The top row shows hand imprints, while the bottom row shows sphere compression. Combining both methods (right column) achieves realistic deformations with enhanced detail while avoiding volume loss.*

### Localized Simulation

Real-time simulation is constrained by hardware capabilities---typical systems can handle approximately 100K particles at interactive rates. However, photorealistic 3D scenes may contain over 500K particles, making full-scene simulation infeasible.

Our insight is that users only interact with a small portion of the scene at any given time. We confine the simulation to a cubic region centered around the user's hands, leaving particles outside this region stationary. This dramatically reduces computational cost while maintaining full responsiveness in the active area.

Experiments show that localized simulation achieves an average **3.3× improvement in frame rate** across typical scenes, enabling smooth interaction even with complex environments.

| Scene | Gaussian Count | FPS without Localization | FPS with Localization |
|:------|:--------------|:-----------------------:|:--------------------:|
| Garden | 660K | 12.2 | **39.8** |
| Bicycle | 480K | 13.1 | **43.5** |
| Room | 382K | 13.6 | **44.5** |

**Table 1:** *Localized simulation significantly improves performance in large 3D Gaussian Splatting scenes.*

# Rendering: From Particles to Pixels

VR-Doh supports two rendering paradigms to serve different use cases.

## Mesh Rendering for Creation

When users create models from scratch, we reconstruct surface meshes from MPM particles using the Marching Cubes algorithm. The process transfers particle mass onto a uniform density grid and extracts an isosurface to construct the mesh. Laplacian smoothing is applied to improve surface quality.

To prevent unwanted visual artifacts when different objects are close together, we assign a "category" attribute to each particle. This ensures that each object is reconstructed and rendered independently.

## 3D Gaussian Splatting for Photorealistic Editing

For editing existing photorealistic 3D content, VR-Doh leverages 3D Gaussian Splatting---a recent breakthrough in neural scene representation. This representation captures complex appearances using millions of oriented 3D Gaussians, enabling high-fidelity rendering of real-world objects.

We build upon PhysGaussian to enable physics-based editing of these representations. However, direct application faces challenges:

### Decoupled Appearance and Physical Representations

Capturing complex visual appearances requires high-density Gaussian distributions---often much denser than what's needed for physics simulation. Running MPM on every Gaussian would be prohibitively expensive.

Our solution decouples the simulation from rendering: a smaller number of MPM particles drive the physics, while the full set of Gaussians follows along for rendering. During each simulation step, MPM particles execute the full physics update first. Then, Gaussian kernels use the resulting grid velocities to update their positions. This approach nearly **doubles the frame rate** while maintaining visual quality.

![Decoupled Representations](./Improved_Gaussian.png)
**Figure 4:** *Reducing the number of MPM particles while maintaining the full Gaussian count for rendering. As the sampling ratio decreases, performance improves significantly with minimal impact on visual quality.*

### Uniform Gaussian Volume Regularization

Original 3D Gaussian training focuses on surface appearance, leaving object interiors sparse or filled with large, low-quality Gaussians. When users deform objects and expose these interiors, the result can look blurry.

We introduce a regularization loss during Gaussian training that penalizes large volume differences between Gaussians:

$$ L_{\text{vol\_ratio}} = \max \left( \frac{\text{mean}(V_{\text{top}, \alpha})}{\text{mean}(V_{\text{bottom}, \alpha})}, r \right) - r $$

This encourages more uniform Gaussian distributions throughout the object, ensuring consistent visual quality even under extreme deformations.

# Hand-Based Interactions

VR-Doh supports multiple interaction modalities that work together to enable both intuitive exploration and precise control.

## Contact-Based Modeling

The most direct interaction is simply using your hands to touch and deform virtual objects. We approximate hand geometry using the medial axis transform---a compact representation consisting of spheres and cones that accurately captures hand shape while being efficient to compute collisions against.

![Hand Geometry Approximation](./MAT_Hand.jpg)
**Figure 5:** *Medial axis approximation of hand geometry. A hand can be represented by 76 medial cones and 28 medial slabs, enabling efficient and accurate collision detection.*

Beyond bare hands, we provide auxiliary tools that attach to hand joints:

- **Planar slabs** for creating flat surfaces
- **Rods** for poking holes and detailed sculpting
- **Scissors** for cutting objects

Each tool follows hand movements naturally, and users can switch between them seamlessly.

![Featured Operations](./Featured_Operations.png)
**Figure 6:** *Featured Operations in VR-Doh: (a) Pinching and reshaping with fingers; (b) Using a sourcing tool to extrude new materials; (c) Mid-air gestures for bending and twisting; (d) Hand-based cutting; (e-f) Tool-based operations with slab and rod.*

## Mid-Air Gestural Modeling

Sometimes contact-based modeling isn't ideal---you might want to deform a specific region without accidentally affecting surrounding areas. For these situations, we provide mid-air pinch gestures that apply force fields to selected particles.

Users pinch with their thumb and middle finger to select a region, then move their hand to stretch, twist, or bend the material. A green highlight provides visual feedback showing the selection area and applied forces. Users can adjust both the selection radius and force magnitude for precise control.

## Other Operations

To support complete modeling workflows, VR-Doh includes standard operations:

- **Selection and movement** by grabbing object bounding boxes
- **Scaling** by grasping with both hands and spreading apart or together
- **Loading primitives** (spheres, cubes, tori) as starting points
- **Sourcing tool** for extruding new geometry with customizable cross-sections
- **Merge, Copy, Reset, and Delete** for object management

# User Studies: Does It Actually Work?

We conducted two user studies to evaluate VR-Doh's usability and effectiveness.

## Study 1: Creative Exploration

We invited 12 participants---6 with 3-5 years of 3D modeling experience and 6 complete novices---to complete two tasks:

1. **Editing a 3D Model**: Freely modify a pre-existing object represented by Gaussian Splatting
2. **Creating from Scratch**: Build a new 3D model using primitive shapes

![User Study Results](./User_Study_Results.jpg)
**Figure 7:** *User study outcomes showing edited 3D GS models (top row) and newly created models (bottom row) by novice and expert participants.*

### Quantitative Results

Participants rated VR-Doh highly across all dimensions:

| Metric | Score (out of 7) |
|:-------|:----------------:|
| Usefulness | 5.6 |
| Ease of Use | 5.5 |
| Ease of Learning | **6.0** |
| Satisfaction | 5.9 |

Notably, no participants reported discomfort or motion sickness during the study.

### Qualitative Feedback

Participants praised several aspects:

> "Directly manipulating the shape of objects with my hand was more flexible than deforming by setting control points."

-- P1 and P3 (Expert users)

> "It feels like walking inside the strawberry."

-- P1, describing the rod tool for adding details

> "Usually, I rarely look at the texture rendering while editing shapes because it's very inefficient in traditional software."

-- P4, appreciating real-time feedback

> "It feels like the mermaid is really swimming in front of me."

-- P2, on the immersive editing experience

Participants also identified areas for improvement, primarily related to hand-tracking stability when adjusting fine details. They suggested adding an undo feature and reducing degrees of freedom during certain operations.

## Study 2: Comparison with Blender

To directly compare VR-Doh with conventional tools, we asked 6 participants to create both a Swiss roll and a donut using VR-Doh and Blender 4.3, guided by video tutorials.

### Time Results

| Task | VR-Doh | Blender |
|:-----|:------:|:-------:|
| Swiss Roll | **5.6 min** | 9.4 min |
| Donut | **5.2 min** | 11.8 min |

**Table 2:** *Average completion times for modeling tasks.*

### Key Advantages Identified

Participants identified four key advantages of VR-Doh over Blender:

1. **Flexible selection**: Dexterous hand movements enable intuitive area selection, avoiding over-selection issues common in 2D views
2. **Realistic deformation**: Elastoplastic simulation provides more realistic results than geometric methods like ARAP
3. **Pose editing without rigging**: Body parts can be adjusted directly without skeletal binding
4. **Physics-based cutting**: Objects can be separated at weak points by applying force, without manually adjusting cutting planes

However, participants noted that for specialized tasks requiring very high precision, Blender's exact vertex/edge/face selection remains superior. The two tools exhibit a **complementary relationship**---VR-Doh excels at intuitive global shaping, while Blender provides precision for detailed edits.

# Example Creations

VR-Doh enables a wide range of creative possibilities.

## Creating Objects from Scratch

![Object Creation Examples](./complete_3.png)
**Figure 8:** *Object creation examples: crafting a snowman, assembling a hamburger with gravity-stacked layers, creating a panda holding bamboo, rolling a Swiss roll, and shaping steamed buns in a bamboo steamer.*

These examples showcase how users can:

- Assemble multi-component objects using natural stacking
- Roll and fold materials to create complex patterns
- Add accessories that integrate seamlessly with main objects

## Editing Photorealistic Objects

![3D GS Editing Examples](./Gaussian_Examples.jpg)
**Figure 9:** *3D Gaussian Splatting editing examples: reshaping mushroom houses, posing a pig for ballet, animating ancient statues, stacking wooden frames, adjusting a girl's pose, intertwining flowers, and customizing a plush toy.*

The ability to edit photorealistic 3D reconstructions opens new creative possibilities:

- **Bringing statues to life**: A terracotta warrior lifts its head to drum; Discobolus throws its discus
- **Pose editing**: A girl's arms bend to mimic eating; a red pig transitions from running to ballet
- **Material manipulation**: Flowers intertwine with automatic collision avoidance; wooden frames stack through gravity

# Discussion and Limitations

While VR-Doh significantly advances accessible 3D modeling, several challenges remain.

## Current Limitations

**Precision for fine details**: The system excels at large-scale deformations but struggles with sub-centimeter precision tasks like creating sharp creases or intricate surface engravings. This limitation stems from both computational budget constraints and vision-based hand tracking accuracy.

**Lack of haptic feedback**: Without tactile sensation, users must rely entirely on visual cues when gauging contact forces. This can lead to accidental modifications, especially in occluded regions where users can't see their hands touching the object.

**Hand tracking stability**: Consumer VR headset hand tracking introduces minor positional jitter that affects delicate manipulations.

## Future Directions

Several promising research directions could address these limitations:

**Hybrid interaction paradigms**: Integrating haptic gloves could provide both improved tracking stability and force feedback during precision tasks. Alternative feedback mechanisms like auditory cues could also help users sense their impact on unseen areas.

**Memory-efficient undo mechanisms**: Unlike traditional modeling software where operations are discrete, physics simulations are continuous. Developing specialized compression schemes for incremental deformation states could enable practical undo/redo without excessive memory overhead.

**Extended modality support**: While our current focus is on VR, the underlying techniques could extend to augmented reality or even desktop interfaces with depth-sensing cameras.

# Conclusion

VR-Doh represents a new paradigm for 3D modeling---one that prioritizes intuitive, physics-based interactions over complex geometric operations. By combining the Material Point Method with natural hand tracking in VR, we enable users to create and edit 3D content as naturally as shaping real clay.

Our user studies demonstrate that both novices and experts find the system intuitive, with participants completing tasks faster than with traditional software and expressing high satisfaction with the modeling experience. The four key advantages---flexible selection, realistic deformation, rig-free pose editing, and physics-based cutting---highlight how physics simulation fundamentally changes the modeling workflow.

As one participant noted, VR-Doh helps avoid the common frustration with conventional software where "the hands cannot keep up with the mind." By making 3D modeling more accessible and intuitive, we hope to democratize digital content creation and inspire new forms of creative expression.

VR-Doh is open-sourced at [https://github.com/Simulation-Intelligence/VR-Doh](https://github.com/Simulation-Intelligence/VR-Doh).

# References

[1] Jiang, C., Schroeder, C., Selle, A., Teran, J., & Stomakhin, A. (2015). The affine particle-in-cell method. *ACM Transactions on Graphics*, 34(4), 1-10.

[2] Hu, Y., Fang, Y., Ge, Z., Qu, Z., Zhu, Y., Pradhana, A., & Jiang, C. (2018). A moving least squares material point method with displacement discontinuity and two-way rigid body coupling. *ACM Transactions on Graphics*, 37(4), 1-14.

[3] Kerbl, B., Kopanas, G., Leimkühler, T., & Drettakis, G. (2023). 3D Gaussian splatting for real-time radiance field rendering. *ACM Transactions on Graphics*, 42(4), 1-14.

[4] Xie, T., Zong, Z., Qiu, Y., Li, X., Feng, Y., Yang, Y., & Jiang, C. (2024). PhysGaussian: Physics-integrated 3D Gaussians for generative dynamics. *CVPR 2024*.

[5] Rosales, E., Rodriguez, J., & Sheffer, A. (2019). SurfaceBrush: From virtual reality drawings to manifold surfaces. *ACM Transactions on Graphics*, 38(4), 1-15.

[6] Yu, Y., Lieu, K., Savvides, M., & Chai, J. (2021). Cassie: Curve and surface sketching in immersive environments. *CHI Conference on Human Factors in Computing Systems*.

[7] Fang, Y., Li, M., Gao, M., & Jiang, C. (2021). Interactive Design Practice with Deformable Object Simulation. *Technical Report*.

[8] Arora, R., Kazi, R. H., Kaufman, D. M., Li, W., & Singh, K. (2019). MagicalHands: Mid-air hand gestures for animating in VR. *UIST '19: Proceedings of the 32nd Annual ACM Symposium on User Interface Software and Technology*.
