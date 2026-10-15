# Research proposal for CVPR 2027: reliable refinement of composed Gaussian scenes

**Date:** 15 October 2026, Asia/Kolkata.  
**Status:** proposal for author review; **implementation and experiments are not authorized**.  
**Scope:** the current relation-conditioned refiner, as previously confirmed by the author.  
**Work performed for this document:** read-only code/result inspection, literature research, and writing this Markdown file. No optimizer, renderer, benchmark, model inference, training, installation, or experiment was run for this proposal. Earlier recorded results below are historical evidence, not new experiments.

## 1. Recommendation and submission reality

The current system is a useful prototype, but its core technical contribution is not yet strong enough for a competitive CVPR methods paper. It optimizes coarse geometric proxies already used in related research, and its strongest numerical result measures those same proxies. The most consequential problem is **a mismatch between Gaussian appearance, inferred solid geometry, and the relation being enforced**, rather than a missing loss weight or a weak language-model prompt.

**Recommended primary direction:** test whether **uncertainty-aware contact and cavity reasoning** can improve independently measured relation correctness while preserving assets. The proposed novelty must be the way uncertain geometric evidence changes relation decisions and optimization—not simply mesh extraction, an SDF loss, or adding uncertainty to Gaussians.

**Recommended conditional extension:** discrete contact-region selection with explicit control of collateral object motion. Pursue it only if geometric measurements are reliable enough; searching more effectively over an incorrect geometry representation can produce more confident failures.

**Required evaluation direction:** a calibrated, relation-level evaluation that separates actual geometry, visible evidence, and unobservable cases. Treat it as infrastructure first; elevate it to a contribution only if it reveals generalizable findings across methods and asset families.

The official dates are **registration November 10, 2026; submission November 16, 2026; supplementary material November 23, 2026, all AoE**. Approximately one month remains from this proposal date. A new large-model training program, extensive asset generation, and several new subsystems are unlikely to be defensible within that window unless substantial resources and data already exist outside this checkout. A narrowly scoped pilot should determine whether a CVPR 2027 submission is realistic; acceptance cannot be inferred from a plan or a small improvement. [Official CVPR 2027 dates](https://cvpr.thecvf.com/Conferences/2027/Dates).

**Approval recommendation:** authorize only the diagnostic pilot E0–E2 first, after reviewing this document. That is a recommendation for a later decision, not authorization to start now. The complete implementation and experiment plan is below so that the decision is concrete.

## 2. What the current method actually does

Inputs are labeled Gaussian object instances, an initial layout, and an externally supplied graph of `on_top`, `inside`, `under`, `next_to`, and `leaning_against` relations. Main experiments optimize per-object translation; scale is frozen and rotation is not optimized. The core objective is a weighted sum of box collision, support, bounds, relation and displacement penalties, with annealed displacement weight. The main setting disables horizontal room bounds.

This is an optimization system, not currently a learned editing architecture. A new neural module is justified only if it solves an identified inference bottleneck and beats a non-learned alternative. “Architecture improvement” below therefore includes replacing the geometric representation and solver structure, not automatically adding a network.

Implementation references: [object model and losses](../../scripts/physrel_refine.py), [layout proposal](../../scripts/propose_layout.py), [composition](../../scripts/compose_scene.py), [relation judge](../../scripts/relation_vqa.py), [renderer](../../scripts/render_scene.py), [existing ray prototype](../../scripts/ray_contact.py). The earlier [project summary](project_summary.md), [assessment](assessment.md), and [35-paper matrix](literature_matrix.md) provide the detailed audit and bibliography.

### 2.1 Historical evidence to explain, not optimize around

| Recorded measure | Before → after | What it does and does not establish |
|---|---|---|
| AABB relation satisfaction | 13/45 → 45/45 | Satisfies the implemented surrogate on 24 development scenes; not independent physical validation. |
| Unquantized bf16 Qwen relation judgments | 30/45 → 38/45 | Net visual gain on this judge; not calibrated human accuracy. |
| `inside` visual judgments | 4/4 → 1/4 | The aggregate improvement hides a serious containment regression. |
| `next_to` visual judgments | 17/17 → 17/17 | Much of the box-metric improvement in this class has no corresponding judge gain. |
| Collision pairs / floating objects | 9 → 0 / 10 → 0 | Counts use box proxies and relation-specific exemptions. |
| Object-normalized motion | Median 8.69%; mean 28.60%; maximum 300.53% | A small median does not establish minimal collateral editing. |

These values come from the previously produced [saved-result audit](results_audit.json), not a rerun for this request. The 69 instances reuse 27 asset types; repeated objects and relations are not independent test samples.

### 2.2 Existing negative results constrain the research plan

The [project notes](../../notes.md) already describe failures of fixed 1% trimming, opacity-weighted quantiles, and three single-direction ray-contact variants. They also state that the ray correction mode was deliberately not applied to the real failure cases. Those records are exploratory observations, not a controlled held-out study, but they rule out presenting the same modifications as fresh hypotheses without explaining what changes.

A desktop may be visually opaque from one direction and partially transparent from another; a thin laptop base may disappear edge-on; a faint Gaussian shell may accumulate opacity over a long ray. Thus a single opacity threshold is neither a reliable physical surface nor a universal contact measure. Multi-view fusion may help, but **cannot recover missing physical information merely by rendering the same inconsistent asset more often**.

## 3. Critical weaknesses and their scientific consequences

| Weakness | Evidence in current implementation | Why it matters compared with recent research | Priority |
|---|---|---|---|
| One AABB per object discards functional geometry | Exterior box containment; no bowl cavity, tabletop underside, shelf compartment or handle opening. | Surface-level and simulation-based systems already model more than box overlap. This directly affects the failed containment examples. | Critical |
| Collision exemptions hide the hardest interactions | `on_top`, `inside`, and `under` pairs are exempt from the collision term/metric. | Contact and nonpenetration must hold together; exempting an entire interacting pair cannot distinguish contact from passing through a wall. | Critical |
| Apparent geometry is treated as certain | Bounds depend on extreme Gaussian extents; ray alternatives disagree on generated assets. | Rendering uncertainty and physical-property uncertainty have existing literature; the unresolved task is using uncertainty correctly in relation decisions. | Critical |
| Optimization and evaluation share the same assumptions | `measure` checks substantially the same proxies optimized by `refine`. | Perfect surrogate performance can coexist with independent failure. This is evaluation circularity, not evidence of a physical guarantee. | Critical |
| Local translation-only search has limited reach | No optimized orientation; prescribed leaning; heuristic placement can choose the wrong support region. | Cannot correct an orientation-dependent fit or jump between disconnected feasible regions. Pose optimization and planning already exist elsewhere. | High |
| No actual stability model | No center of mass, torque, friction, force equilibrium or perturbation test. | A supported box can still topple; a leaning object can slide. PAT3D and PhyScensis already evaluate dynamics/stability. | High |
| “Minimal edit” is only a soft penalty | All objects can move; no target mask in main saved runs; large motion outliers. | Maintaining unrelated relations is different from minimizing summed translation. ScanEdit and Edit-As-Act are important precedents. | High |
| Feasibility screening is not a certificate | Conservative grid-capacity heuristic; some constraints can be dropped or loosened. | Failure of the heuristic/optimizer does not prove physical impossibility. Reporting denominators after dropping hard cases can mislead. | High |
| Relation semantics are weaker than their names | `under` compares with reference top, leaning angle is not measured, adjacency has a global threshold. | A solver can satisfy the formula without achieving the intended geometric or functional relation. | High |
| Grounding is given, not solved | Benchmark graph copied into layout; object membership known. | Fair baselines must receive comparable information. Current evidence does not support unrestricted language editing. | High |
| Weak generalization evidence | 24 hand-designed scenes, five imbalanced classes, repeated assets, no held-out split. | Cannot establish robustness across shape, scale, clutter, topology or generator. | Critical |
| Uncalibrated VLM evaluation | Forced choices; no abstention; before/after framing changes; views judged jointly. | Does not distinguish unobservable from false, or visual plausibility from physical validity. | High |
| Incomplete reproducibility | Real PLYs/configuration manifests absent; external baselines not reproduced. | Prevents a trustworthy comparison and makes compute/runtime claims premature. | Critical |

Additional implementation concern: hard min/max is **not inherently a gradient defect for translation-only boxes**. All points share a translation, so the active extreme has the correct translation derivative. Replacing it with a smooth maximum can inflate bounds. The major weakness is what the box represents, not simply that `min`/`max` is nonsmooth. Rotation introduces different optimization issues and should be tested separately.

## 4. Literature review: what prior work already establishes

This review builds on the existing 35-paper bibliography and extends it with surface reconstruction, uncertainty, active observation, and newer physical scene systems. The tables below focus on implications for the proposed research rather than repeating every abstract. Sources are primary proceedings, author papers, or official project pages. “Reported” means the authors' result, not our reproduction. A task mismatch is not a demonstrated failure of a competing method.

### 4.1 Direct and near-direct scene competitors

| Work and verified status | Relevant established capability | Consequence for our proposal / comparison |
|---|---|---|
| [LayoutVLM](https://openaccess.thecvf.com/content/CVPR2025/html/Sun_LayoutVLM_Differentiable_Optimization_of_3D_Layout_via_Vision-Language_Models_CVPR_2025_paper.html), CVPR 2025 | VLM-derived relations and differentiable layout optimization; shared assets in its layout evaluation. | Mandatory practical baseline. Give it comparable graph/asset information in an adapted solver track. |
| [LayoutDreamer](https://arxiv.org/html/2502.01949v1), Pattern Recognition 2026 ([journal record](https://doi.org/10.1016/j.patcog.2026.113427)) | Layout optimization freezes other Gaussian attributes and optimizes translations/rotations. | Frozen assets and pose refinement are not novel. Detailed reading is of the accessible preprint; verify final implementation changes before reproduction. |
| [CG3D](https://arxiv.org/html/2311.17907), 2023 preprint | Gaussian composition with contact/gravity reasoning and editing; discusses friction/initialization failures. | Gaussian contact objectives predate this project. Compare an explicitly labeled fixed-asset adaptation if implemented. |
| [ScanEdit](https://openaccess.thecvf.com/content/ICCV2025/html/Boudjoghra_ScanEdit_Hierarchically-Guided_Functional_3D_Scan_Editing_ICCV_2025_paper.html), ICCV 2025 | Functional scan editing with hierarchical reasoning and physical optimization. | “Instruction editing with physical relations” is already occupied. Relevant to collateral-edit constraints, but inputs differ. |
| [PAT3D](https://simulation-intelligence.github.io/PAT3D/), ICLR 2026 | Differentiable rigid-body simulation for scene generation; editing applications and stability/penetration evaluation. | Adding a simulator is not enough novelty. Strong mesh-backed comparator if representation conversion is controlled. |
| [Scenethesis](https://arxiv.org/html/2505.02836), ICLR 2026 ([official project](https://research.nvidia.com/labs/dir/scenethesis/)) | Visual grounding, pose correction, SDF/surface-based physical objectives, and judge-driven replanning. | New addition to the earlier shortlist. Rules out generic “surface contact plus visual feedback” novelty. Study fixed-asset optimization separately from its generator. |
| [PhyScensis](https://arxiv.org/html/2602.14968), ICLR 2026 ([official project](https://physcensis.github.io/)) | Predicates, geometric/physics solvers and feedback for dense arrangements; evaluation includes settle distance and robot-task transfer. | New strong precedent for packing, support, solver feedback and stability. Code availability needs confirmation; do not invent a runnable release. |
| [HOG-Layout](https://openaccess.thecvf.com/content/CVPR2026/papers/Jiang_HOG-Layout_Hierarchical_3D_Scene_Generation_Optimization_and_Editing_via_Vision-Language_CVPR_2026_paper.pdf), CVPR 2026 | Hierarchical generation, optimization and editing. | Adding hierarchy alone is not a contribution. Comparison should isolate pose correction and information supplied. |
| [Edit-As-Act](https://openaccess.thecvf.com/content/CVPR2026/html/Noh_Edit-As-Act_Goal-Regressive_Planning_for_Open-Vocabulary_3D_Indoor_Scene_Editing_CVPR_2026_paper.html), CVPR 2026 | Goal-regressive editing with explicit preconditions/effects for support, contact and collision. | Discrete relation planning and validation have direct precedent; a new planner needs a narrower technical distinction. |
| [SceneWeaver](https://papers.nips.cc/paper_files/paper/2025/hash/cd2b3c429c8a2ca57656970e010b4b60-Abstract-Conference.html), NeurIPS 2025 | Iterative reflective scene synthesis with physical execution. | An agent loop is not a research contribution by itself. |
| [RoomPlanner](https://link.springer.com/chapter/10.1007/978-3-032-37038-9_9), ECCV 2026 | Collision/reachability-constrained Gaussian room generation and view sampling. | Gaussian scenes, physics and camera selection are already combined; our target must be local relation validity under ambiguous assets. |
| [SceneSmith](https://proceedings.mlr.press/v306/pfaff26a.html), ICML 2026 | Hierarchical generation of dense simulation-ready scenes, physical properties and simulation evaluation. | Relevant beyond the requested venues because it raises the standard for functional scene evidence. Full-system performance is not a directly comparable local-refiner score. |

**Experimental limitations versus our inference.** PhyScensis reports failures involving asset retrieval, unsuccessful search and unstable stacking; its appendix also reports difficulties adapting LayoutVLM to crowded tabletop arrangements. That is evidence in its particular setup, not proof LayoutVLM fails universally. Scenethesis reports collision and instability metrics separately from alignment; PAT3D also goes beyond images. We therefore cannot claim existing work lacks physical evaluation. Our potential distinction is how an editor handles **uncertain asset geometry and selective acceptance**, which these citations do not by themselves establish as solved or unsolved.

### 4.2 Surface representation, uncertainty and active views

| Work and status | What it contributes | Use here and limitation of transfer |
|---|---|---|
| [SuGaR: Surface-Aligned Gaussian Splatting for Efficient 3D Mesh Reconstruction and High-Quality Mesh Rendering](https://openaccess.thecvf.com/content/CVPR2024/papers/Guedon_SuGaR_Surface-Aligned_Gaussian_Splatting_for_Efficient_3D_Mesh_Reconstruction_and_CVPR_2024_paper.pdf), CVPR 2024 | Surface alignment, mesh extraction and Gaussian/mesh refinement. | Strong geometry baseline. Refitting changes the representation and requires supervision; it is not a free fixed-asset conversion. |
| [2D Gaussian Splatting for Geometrically Accurate Radiance Fields](https://surfsplatting.github.io/), SIGGRAPH 2024 | Surface-oriented disks and geometric regularization. | Tests whether better upstream geometry removes the need for our proposed contribution. Results on captured multi-view data do not guarantee SDS-asset repair. |
| [Gaussian Opacity Fields](https://niujinshuchong.github.io/gaussian-opacity-fields/), TOG / SIGGRAPH Asia 2024 | Opacity level sets and adaptive surface extraction. | Essential alternative to our box/ray proxies. Distinguish extraction from an existing asset from retraining under GOF regularization. |
| [GSRecon](https://openaccess.thecvf.com/content/ICCV2025/html/Yang_GSRecon_Efficient_Generalizable_Gaussian_Splatting_for_Surface_Reconstruction_from_Sparse_ICCV_2025_paper.html), ICCV 2025 | Cross-view geometric enhancement and fusion; tested on DTU, BlendedMVS and Tanks and Temples. | Useful learned geometry alternative when posed images exist. Generated views from one flawed asset are a different input distribution. |
| [GeoSVR](https://papers.nips.cc/paper_files/paper/2025/hash/9ca825deb6ce588c96f880728d3b8aea-Abstract-Conference.html), NeurIPS 2025 | Sparse-voxel surface reconstruction with uncertainty-aware depth constraints. | A non-Gaussian geometry alternative. Demonstrates that uncertainty-assisted reconstruction is not new in itself. |
| [Bayes' Rays](https://openaccess.thecvf.com/content/CVPR2024/html/Goli_Bayes_Rays_Uncertainty_Quantification_for_Neural_Radiance_Fields_CVPR_2024_paper.html), CVPR 2024 | Post-hoc volumetric uncertainty for pretrained NeRFs using a Bayesian approximation. | Uncertainty reference; not directly a calibrated physical occupancy estimator for generated GS. |
| [FisherRF: Active View Selection and Mapping with Radiance Fields Using Fisher Information](https://link.springer.com/chapter/10.1007/978-3-031-72624-8_24), ECCV 2024 | Information-based uncertainty and view selection. | Necessary active-view comparator/reference. Acquiring new observations differs fundamentally from rendering more views of an unchanged asset. |
| [PhysGS](https://openaccess.thecvf.com/content/CVPR2026/html/Chopra_PhysGS_Bayesian-Inferred_Gaussian_Splatting_for_Physical_Property_Estimation_CVPR_2026_paper.html), CVPR 2026 | Bayesian physical-property estimation and uncertainty on Gaussian representations. | Do not claim first uncertainty-aware physical GS reasoning. Our proposed uncertainty concerns contact/cavity geometry and edit validity. |
| [Splat-Nav](https://doi.org/10.1109/TRO.2025.3552348), IEEE T-RO 2025 | Collision-constrained planning in Gaussian maps. | Essential precedent for native-Gaussian collision reasoning, although navigation is not relation editing. |
| [Let us Make a Splan](https://ieeexplore.ieee.org/document/11059887/), IEEE T-RO 2025 | Risk-aware trajectory optimization with collision probability bounds in normalized Gaussian splats. | Rules out blanket novelty for probabilistic Gaussian collision constraints. Its radiance/probability model assumptions must be compared carefully. |
| [Conformal Risk Control](https://proceedings.iclr.cc/paper_files/paper/2024/hash/f3549ef9b5ff520a7e41ff3cc306ab2b-Abstract-Conference.html), ICLR 2024 | Calibration controlling expected monotone losses under specified assumptions. | Possible statistical tool, not automatic certification of optimized scene geometry or conditional accuracy among accepted edits. |

### 4.3 Editing and evaluation that constrain our claims

| Literature | What must be acknowledged |
|---|---|
| [GaussianEditor—Chen et al.](https://openaccess.thecvf.com/content/CVPR2024/html/Chen_GaussianEditor_Swift_and_Controllable_3D_Editing_with_Gaussian_Splatting_CVPR_2024_paper.html) and [GaussianEditor—Wang et al.](https://openaccess.thecvf.com/content/CVPR2024/html/Wang_GaussianEditor_Editing_3D_Gaussians_Delicately_with_Text_Instructions_CVPR_2024_paper.html), CVPR 2024 | Localized Gaussian editing already exists. Their appearance-edit objectives make them unsuitable primary layout baselines. |
| [Gaussian Grouping](https://arxiv.org/abs/2312.00732) and [GaussCtrl](https://arxiv.org/abs/2403.08733), ECCV 2024 | Instance manipulation and multi-view-consistent generative edits have precedents. We assume labels and freeze attributes; that is a restriction, not a new capability. |
| [3DitScene](https://proceedings.iclr.cc/paper_files/paper/2025/hash/07c256a163a7559186ec1c71e95b9ec9-Abstract-Conference.html), ICLR 2025 | Language-guided disentangled Gaussian scene/object manipulation. Add to the editing context; do not confuse composition control with physical validation. |
| [PhysSplat](https://openaccess.thecvf.com/content/ICCV2025/html/Zhao_PhysSplat_Efficient_Physics_Simulation_for_3D_Scenes_via_MLLM-Guided_Gaussian_ICCV_2025_paper.html), ICCV 2025; [PhysTalk](https://arxiv.org/abs/2512.24986), 2025 preprint | Physical simulation and language-driven Gaussian animation already have adjacent systems; adding a physics-engine connection is insufficient. PhysTalk's venue was not verified. |
| [SceneEval](https://openaccess.thecvf.com/content/WACV2026/html/Tam_SceneEval_Evaluating_Semantic_Coherence_in_Text-Conditioned_3D_Indoor_Scene_Synthesis_WACV_2026_paper.html), WACV 2026 | Decomposed semantic/geometric scene evaluation is prior work; use relevant parts rather than claiming the first structured evaluator. |
| [GPTEval3D](https://arxiv.org/abs/2401.04092), CVPR 2024; [VQAScore](https://www.ecva.net/papers/eccv_2024/papers_ECCV/papers/01435.pdf), ECCV 2024 | Visual judging and compositional scoring have human-alignment evidence in their settings, not a guarantee for our spatial-contact task. |
| [VSI-Bench](https://arxiv.org/abs/2412.14171), CVPR 2025; [SpatialVLM](https://arxiv.org/abs/2401.12168), CVPR 2024 | Spatial reasoning must be evaluated explicitly. Neither supplies ground truth for our saved judge decisions. |
| [MUSE](https://arxiv.org/abs/2606.14168), 2026 preprint | Memory-grounded preservation across scene edits is already being studied. Do not claim untouched-requirement preservation is unexplored. |

**Watchlist, not accepted evidence of a published baseline:** [PhyMix: Towards Physically Consistent Single-Image 3D Indoor Scene Generation with Implicit–Explicit Optimization](https://openreview.net/pdf/a08871c55f476dfd4fca9f88ae466e84b0ec4927.pdf). The primary indexed submission describes a physics evaluator, training-time preference optimization and test-time correction, but the accessible record was marked under review and full access was blocked. Acceptance, final authors, release and exact method need verification before citation as an accepted ICLR paper or a mandatory reproduction. Its existence is a warning against claiming evaluator-driven test-time correction is unprecedented.

**Review conclusion:** the most promising gap is not “physics-aware Gaussian editing.” It is a narrower, still-unverified question: **can a local editor use calibrated geometric ambiguity to make more reliable relation corrections, while explicitly controlling which existing objects and relations change?** No novelty priority claim is established by this review alone.

## 5. Ranked research directions

Ratings concern research potential, not demonstrated performance. All proposed components and experiments are pending approval.

| Rank / direction | Hypothesized contribution | Novelty risk | Feasibility from current checkout | Recommendation |
|---|---|---|---|---|
| **R1. Relation-specific geometric uncertainty** | Propagate ambiguous contact/cavity geometry into pose constraints and explicit acceptance/abstention decisions. | Medium–high: robust optimization and uncertainty are established; the relation-specific formulation and evidence must add something substantive. | Medium; requires reliable geometry/evaluation data first. | Primary research hypothesis; approve diagnostics before implementation. |
| **R2. Contact-region search with minimal collateral editing** | Jointly choose valid contact regions and the subset of objects allowed to move, preserving unaffected relations. | High: planning, sparse optimization and contact search are established. | Medium on small scenes; low for a broad planner in one month. | Conditional extension or alternative if search, rather than geometry, dominates. |
| **R3. Calibrated relation evaluation with targeted views** | Distinguish physical truth, visible evidence and unknown cases; select views that discriminate relation failures at fixed cost. | Medium for a broad diagnostic study; high if only a new prompt/camera heuristic. | Medium–high once labels and assets are available. | Required evaluation infrastructure; independent paper direction only with strong findings. |
| **R4. Local asset repair under relation constraints** | Correct genuinely inconsistent geometry in restricted regions while preserving rendered identity elsewhere. | High and technically difficult; surface-aligned editing already exists. | Low for this deadline; changes the task. | Longer-term direction, not part of the first approval request. |

### R1. Relation-specific geometric uncertainty — primary hypothesis

**Research question.** When Gaussian appearance admits several plausible contact surfaces, can the editor optimize relations against the uncertainty that matters for that relation, and avoid declaring an edit correct when the evidence is insufficient?

#### Proposed representation and architecture

1. **Keep the original Gaussian asset for rendering.** Build a separate object-space geometric representation for physical reasoning. Initial alternatives are a fused depth/TSDF surface, a GOF-derived surface, and simple component geometry. Preserve cavities and thin structures; do not automatically close every hole or take a convex hull.
2. **Estimate geometric ambiguity locally.** Store candidate surfaces or signed-distance intervals, together with observation support, view disagreement and reconstruction residual features. Validate these against known geometry. Gaussian covariance is the spatial footprint of a primitive, **not epistemic uncertainty**. Low opacity is not automatically low confidence in a surface.
3. **Represent relations using geometric witnesses.** A witness is a concrete contact patch, cavity/free-space region, clearance region, or paired surface sample that makes a relation checkable. `on_top` requires a suitable support patch plus nonpenetration; `inside` requires occupancy of the actual cavity rather than the exterior box; `under` requires clearance below the relevant underside. A single “nearest point” is insufficient for support stability or containment.
4. **Propagate uncertainty into relation constraints.** Use uncertainty around the relevant patches/clearance volumes, not one global per-object confidence scalar. A bowl can have a well-observed exterior and an unreliable interior. Jointly optimizing the relation with all its uncertainty set may be conservative; quantify that cost rather than quietly narrowing the set.
5. **Optimize poses subject to accepted evidence.** Start with translation-only refinement to isolate the representation contribution. Add SE(3) only as a separate factor. Keep nonpenetration active for interacting pairs; contact is a constraint, not a collision exemption.
6. **Return explicit outcomes:** accepted under the stated model; insufficient evidence; optimization failure/time budget exceeded; or infeasible under a precisely defined restricted model. Do not label a nonconvergent run physically impossible.

The proposed core novelty is **coupling relation-specific geometric ambiguity to the edit decision and its feasible pose set**. A point-estimate mesh, an uncertainty visualization, or an extra regularization term alone does not establish that contribution. Bayes' Rays, FisherRF, PhysGS and SPLANNING are essential precedents; Scenethesis and PAT3D are essential point-estimate/simulation comparisons.

#### Mathematical formulation to investigate

Let \(G_i\) be the fixed rendered asset, \(T_i\) its pose, and \(\mathcal U_i\) a set of plausible solid geometries consistent with validated evidence. For relation \(r\), let \(z_r\) select a geometric witness and let \(c_r(T,z_r,S)\leq0\) denote its violation constraint for geometry \(S\). A candidate robust formulation is

\[
\min_{T,z} D(T,T^0)
\quad\text{subject to}\quad
\sup_{S\in\mathcal U}c_r(T,z_r,S)\leq0
\quad\text{for the requested relations and all relevant collision checks}.
\]

This formulation is a **research proposal**, not an implemented algorithm or a new robust-optimization theorem. The uncertainty set may be wrong, constraints can be nonconvex, and excessive conservatism can make useful contact impossible. An approximate solver must report residuals and timeouts; it cannot claim global optimality.

**A useful analytical diagnostic:** suppose a horizontal support height is only known to lie in \([h_-,h_+]\), and the load's bottom is placed at \(z\). Nonpenetration for every plausible height requires \(z\geq h_+\). A gap at most \(\tau\) for every height requires \(z\leq h_-+\tau\). Both are possible only if

\[
h_+-h_-\leq\tau.
\]

This elementary interval result explains why exact robust contact and uncertain surfaces can be incompatible. It motivates better evidence or abstention, not increasing the test tolerance until an edit passes. It is a sanity check under planar assumptions, not a sufficient theoretical contribution by itself.

**Possible conditional guarantee:** if a signed-distance approximation has a uniform error bound over the queried region, and occupied-volume samples have a known covering radius, a sufficient positive margin can bound missed penetration. The proof must include approximation error, sampling coverage and both objects' geometric error. Surface samples alone can miss one solid entirely inside another; unknown interiors and non-watertight proxies invalidate a blanket guarantee. A sampled empirical error bar must not be advertised as a uniform geometric bound.

**Calibration caveat:** marginal prediction coverage does not automatically survive adaptive pose/witness selection, certify each scene, or bound error conditional on accepting an edit. A statistical version would need scene-level calibration of the complete frozen decision pipeline, a suitable loss/selection guarantee, and an explicit treatment of distribution shift. Conformal Risk Control is a possible tool under its assumptions, not a plug-in proof. For the first pilot, use held-out reliability/coverage curves and avoid unsupported guarantees.

#### Key tests and falsification

- Compare the same deterministic surface solver with and without uncertainty; otherwise surface quality and uncertainty are confounded.
- Compare against a simple uncertainty threshold that abstains after refinement. If that matches the full method, coupled robust optimization may be unnecessary.
- Evaluate correctness at fixed acceptance coverage and motion budgets. Rejecting nearly everything is not improvement.
- Use clean, corrupted and generator-shifted assets. If ambiguity only predicts the corruption injected by us, the method has not demonstrated realistic transfer.
- **Stop or downgrade R1** if surface disagreement does not predict independent relation error, if calibrated sets are too broad for useful coverage, or if an ordinary mesh/SDF solver matches the complete method.

### R2. Contact-region search and minimal collateral editing — conditional extension

**Research question.** Can an editor repair a local relation while selecting a physically valid contact region and moving fewer unrelated objects than continuous whole-scene optimization?

The current local optimizer cannot change from one shelf compartment to another or correct an unsuitable prescribed lean. Merely switching Adam to another optimizer, adding rotations, or increasing restarts would be baseline improvements, not the central contribution.

**Proposed structure:** enumerate a small set of support/cavity/contact candidates per affected object; construct a local graph of requested and preserved relations; search contact assignments and movable-object subsets; solve continuous pose correction for each candidate; verify the result independently. Expand the affected region when a preserved relation is violated, with a fixed search budget. Compare beam search, random restarts and exhaustive enumeration on small cases.

For object \(i\), let \(m_i\in\{0,1\}\) indicate permission to move. A candidate objective is lexicographic:

1. satisfy explicit requested relations and mandatory nonpenetration;
2. preserve designated existing relations and fixed anchors;
3. minimize weighted count of moved non-target objects;
4. minimize normalized translation/rotation within that choice.

Use an explicit infeasibility/relaxation policy when requirements conflict. The author of an edit must decide which constraints may be relaxed; the optimizer must not silently rewrite the task. For a given translation bound \(B_i\), \(\|t_i-t_i^0\|\leq m_i B_i\) is one way to link motion to the discrete decision; it is standard mixed-integer modeling, not novelty.

**Where a technical contribution might lie:** exploiting relation/contact structure to search a small set of physically meaningful edits with measurable minimality bounds in a restricted setting. For tiny analytic scenes, exhaustive enumeration can establish the optimal movable subset and reveal the approximation gap. A local graph heuristic alone cannot guarantee global minimality or discovery of all collision dependencies.

**SE(3) implementation requirements, if later approved:** transform centers and covariances consistently; compose rotations correctly; handle directional SH coefficients or explicitly use a DC-only representation in all compared methods. A rigid transform preserves object shape, but it does not imply unchanged world-view appearance or occlusion. Scaling must be excluded from the main rigid-edit track or reported separately.

**Precedents and tests:** compare against ScanEdit/Edit-As-Act-style preservation/planning, LayoutVLM, multistart surface optimization and a generic sparse-motion penalty. Test clutter, support chains, objects initially in the wrong compartment, and independent distractors. Require a lower collateral-motion/Pareto curve at comparable physical success—not simply fewer moved objects achieved by leaving requested edits incomplete.

**Stop or downgrade R2** if simple multistart or hard anchor masks achieve the same result, if contact candidates omit valid solutions, or if the method only helps cases constructed to favor its graph rules.

### R3. Calibrated relation evaluation and targeted views — necessary companion

**Research question.** Which geometric and visual measurements reliably distinguish contact, floating, penetration, containment and unobservability across methods and asset types?

Create two separate label spaces:

- **Physical/geometric state:** known from analytic solids or vetted reference meshes; dynamics where valid physical properties are available.
- **Visible evidence:** humans judge whether a relation appears satisfied, violated, or unobservable in a fixed view set. This label is not promoted to hidden-geometry ground truth.

Include counterfactual pairs that preserve much of the image while changing contact/containment, and pairs with the same physical state but different occlusion, opacity artifacts or camera distance. Use exact analytic manipulations for physical labels; use independent human labels for visible evidence.

Compare fixed views, random views, geometry-coverage views, an information-based strategy inspired by FisherRF, and relation-focused disagreement views under the same image/token budget. A candidate acquisition rule chooses the next view where the competing hypotheses—floating, touching, penetrating—would be most distinguishable, subject to actual visibility. Do not claim that an extra synthetic rendering reduces uncertainty about unobserved real geometry. It can reduce uncertainty in the **judge's interpretation of the represented scene**.

Split camera/threshold development from evaluation. Freeze the view policy before held-out testing; include an external fixed-camera test so that selecting convenient views cannot hide failures. If a judge is used inside optimization, use different held-out judges and people for final assessment.

**Potential contribution:** a validated diagnostic benchmark and a decision protocol with measurable reliability/efficiency. A new prompt, more cameras or a larger judge is not sufficient novelty. SceneEval, GPTEval3D, VQAScore and VSI-Bench must be credited and compared appropriately.

**Stop or downgrade R3 as a method claim** if improvement disappears at equal view/token cost, a fixed set of close-up views performs equally well, or human agreement is too low to support the proposed labels. Retain transparent evaluation even if active selection fails.

### R4. Restricted geometry repair — longer-term alternative

If no rigid placement can reconcile the appearance and physical geometry of an asset, allow a separate task in which a limited contact-region geometry/opacity update is permitted. Couple a surface model to Gaussians, restrict changes spatially, penalize changes in held-out renders, and preserve unaffected object geometry. Compare against SuGaR/2DGS-style refitting, generic mesh repair and regeneration.

The main risk is cheating the relation task by shrinking, hiding, deleting or repainting the offending geometry. Require retained mass/volume or shape constraints where meaningful, visible object completeness, explicit change budgets and human identity checks. Report shape-changing repair separately from rigid refinement. This direction has substantial overlap with surface-aligned representations and generative editing, requires more data and validation, and is **not recommended for the initial CVPR 2027 effort**.

### Ideas not recommended as standalone contributions

| Change | Appropriate role |
|---|---|
| Better loss weights, smoother bounds, longer optimization | Engineering/ablation; may fix implementation issues, but no strong novelty. |
| A global trim percentage or opacity-weighted contact | Already explored negatively in this repository; use as historical/controlled baselines only. |
| Rotation or scale optimization | Important task capability/baseline, already common in prior methods. |
| A stronger VLM or an agent feedback loop | Potential quality improvement; requires cost-matched controls and is not itself novel. |
| Mesh extraction followed by off-the-shelf physics | Essential baseline and possibly a useful solution, but extensive precedent. |
| Gaussian-overlap analytic loss | Useful surrogate to test; overlap of radiance kernels is not automatically solid penetration. Closed-form Gaussian products are standard. |
| A GNN/transformer that predicts pose corrections | Consider only if measured search cost warrants amortization and a substantial disjoint training set exists. A network replacement alone is superficial. |
| RL against the current AABB score or one VLM judge | Likely to optimize the evaluator's weaknesses; requires independent rewards/validation and is too costly for the first stage. |
| Claiming stability from a center-of-mass footprint test alone | Valid only under restricted support assumptions; multi-contact leaning and friction require richer mechanics. |

## 6. Experiment design — all pending explicit approval

### 6.1 Separate tasks and information access

**Track A: controlled local refinement.** Same assets, initial poses, instance labels, requested relation graph, preserved relations, bounds, transformation freedoms and compute budget for every method. This is the main scientific comparison. Keep translation-only and SE(3) results separate.

**Track B: full language/planning pipelines.** Methods infer their own relations/initialization from a common instruction and asset set. Report planning success and refinement success separately, including missing/incorrect graphs. Do not mix these scores with oracle-graph Track A.

**Track C: asset-changing repair.** Geometry/refitting/regeneration is allowed under a defined budget. Evaluate shape and appearance changes; do not rank it as if it preserved the assets used by Track A.

The oracle reference mesh is available **only to evaluation** in the main estimated-geometry condition. A separate oracle-geometry solver condition measures the upper bound and isolates representation error. Baselines must not receive clean meshes while the proposed method receives noisy Gaussians, or vice versa, without labeling that information advantage.

### 6.2 Proposed data and splits

Counts below are **planning targets**, not collected data, power guarantees, or permission to generate anything. Final scale depends on the approved pilot and resource budget.

| Set | Proposed construction | Role / precautions |
|---|---|---|
| D0: existing pilot | Current 24 scenes, 27 asset types. Recover exact PLYs and provenance. | Development and qualitative regression only; never relabel as held-out after repeated tuning. |
| D1-small: diagnostic geometry | 60 analytic/mesh-backed development cases balanced across support, containment and underside clearance; add lean/stability checks only when materials/geometry are known. | First bottleneck test. Include thin slabs, hollow bowls, tables with legs, overhangs, nested objects and contradictory requirements. Analytic primitive fixtures alone are not a publication benchmark. |
| D1-main: controlled generalization | Target 60 development, 60 calibration, 180 held-out base scenes, with disjoint asset-family splits. Three perturbation magnitudes per held-out scene yield 540 initial layouts per method. | Use vetted meshes to generate matched Gaussian assets with known geometry. Same-base perturbations are repeated measures, not 540 independent scenes. D1-small may be part of development, never part of calibration/test. |
| D2: imperfect generated assets | Target 60 new scenes from at least 30 additional asset families, subject to asset access and generation cost. | Tests transfer to the real problem. No true solid geometry is presumed; human visual labels and proxy-agreement diagnostics remain separate from physical claims. |
| D3: captured/reconstructed assets | Optional small transfer subset with reference geometry or validated surface measurements. | Tests whether the method overfits a particular generator. Requires data rights, labels and an explicit reconstruction protocol; not required for the first pilot. |

Split by **underlying asset family and scene template**, not just rendering or random seed. Variants of one CAD model or Gaussian asset must remain in the same split. If an uncertainty predictor is learned, create a separate training split; the proposed calibration set cannot double as training or threshold-tuning data.

Factor coverage should include:

- Relation type; concave versus convex geometry; object scale ratio; surface thickness; support area; cavity opening size and topology.
- Initial position error and, separately, rotation error; wrong support compartment; stack depth; clutter; preserved unrelated objects.
- Gaussian outliers, opacity shells, missing surfaces and view-dependent artifacts, using both controlled corruptions and naturally generated failures.
- Known feasible, analytically infeasible under declared freedoms, and unknown-feasibility cases. An open bowl and a sealed box can have similar exterior bounds but very different relation requirements.
- Final-state containment versus insertion-path feasibility as separate tasks. Current refinement edits a static scene; a collision-free final placement does not prove an executable insertion trajectory.

Do not claim arbitrary perturbation amplitudes are “realistic.” Estimate error ranges from recovered real initial layouts and report deliberately extreme stress tests separately. Lock all held-out cases before the final method is tuned.

### 6.3 Strong baselines and what they require

| Baseline | Why it is needed | Planned adaptation / fairness requirement | Priority |
|---|---|---|---|
| B0: unchanged scene | Establishes repair versus regression. | Identical initial state and evaluator. | Mandatory |
| B1: current refiner | Measures progress over the actual project. | Freeze main_v3-compatible parameters; evaluate all original relations regardless of dropped constraints. | Mandatory |
| B2: sequential projection + collision relaxation | Tests whether a simple solver explains gains. | Same geometry and graph; several relation orders; explicit iteration/time budget. | Mandatory |
| B3: deterministic surface/SDF refinement | Separates richer geometry from the proposed uncertainty contribution. | Same extracted proxy and pose freedoms as R1; use a conventional constrained optimizer or augmented Lagrangian. Standard optimization is not labeled new. | Mandatory |
| B4: mesh-proxy physics settling + relation correction | Strong physical alternative. | Same estimated proxies; controlled mass/friction; no oracle mesh unless every method has it. Recheck desired relations after settling. | Mandatory for physical claims |
| B5: LayoutVLM | Best immediately adaptable external method; bridge already exists. | Pin official code/model/prompt. Report native pipeline and fixed-input solver adaptation separately. Substitute Qwen backend must be labeled as a substitution. | Mandatory external comparison |
| B6: PAT3D | Strong simulation-based scene method. | Establish supported input adapter and official release; distinguish reproduced paper pipeline from extracted solver components. | High, conditional on feasibility |
| B7: Scenethesis / PhyScensis | Newer surface-optimization/physics-predicate precedents. | Verify release/access; use common asset/graph inputs when possible. If reimplemented, name the component and deviations. Never invent full-method numbers from a generic simulator. | High conceptual priority; at least one feasible relevant component/system comparison desirable |
| B8: LayoutDreamer / CG3D layout objectives | Close Gaussian-native objectives. | Official code if obtainable; otherwise a documented fixed-asset objective reimplementation with supported relations identified. | Secondary to completing reliable mandatory baselines |
| B9: ScanEdit / Edit-As-Act | Strong preservation/planning precedents for R2. | Restricted repositioning subset, common semantics and input information. Full instruction pipelines evaluated only in Track B. | Required if R2 becomes the main claim |
| B10: simple uncertainty gate + best deterministic solver | Tests if abstention alone explains R1. | Same ambiguity estimator, calibration data and acceptance coverage. | Mandatory for R1 |
| B11: random/multistart surface search | Tests if discrete planning is merely extra compute. | Match wall time, proposals and allowed transformations. | Mandatory for R2 |

For all methods, report complete requested-scene denominators, initialization/planning failures, unsupported relations, timeouts and missing outputs. Do not compare only successful outputs or select the best baseline seed using test results. Where native methods solve different tasks, publish a capability table and a declared common subset rather than forcing meaningless scores.

The existing LayoutVLM bridge is evidence of an integration path, not a completed baseline. Its notes-only substitute-backend trial is not a published-setting reproduction. All external comparisons above remain proposals.

### 6.4 Experiment matrix

| ID / purpose | Hypothesis and design | Main controls / ablations | Readout and decision |
|---|---|---|---|
| **E0: provenance and validity** | Recover real inputs; verify transforms, labels, units, camera conventions and thresholds; establish independent analytic geometry checks. | Freeze historical reports; retain source hashes; inspect failures manually without using them as test data. | Stop if source assets or trustworthy labels cannot be recovered. No new success claim until provenance is clear. |
| **E1: geometry diagnostic** | Better contact/cavity estimates reduce error on known geometry before introducing a new optimizer. | AABB, trimmed box, existing rays, multiview TSDF, GOF-style extraction, component geometry; held-out reconstruction supervision where available. | Contact-distance error, cavity validity, thin-surface retention, and agreement with true relation states. If all fail, prioritize representation repair/uncertainty rather than solver tuning. |
| **E2: bottleneck isolation** | Representation error and search error can be separated. | Cross estimated versus oracle geometry with local versus stronger multistart/constrained solving. Keep tasks, initial states and freedoms fixed. | An oracle-geometry gain identifies geometric error; an oracle/search gain identifies optimization error. Choose R1 or R2 using evidence. |
| **E3: uncertainty contribution** | R1 improves independent correctness at fixed coverage and displacement. | Full R1; identical deterministic surface solver; post-hoc uncertainty gate; shuffled uncertainty; oracle uncertainty as diagnostic only. | Paired correctness/coverage curves and calibration. If post-hoc gating matches R1, do not claim coupled optimization adds value. |
| **E4: contact witness and cavity modeling** | Relation-specific witnesses outperform one global distance or scalar uncertainty. | No cavity model; nearest-point contact; per-object scalar uncertainty; patch-level uncertainty; shared versus independently selected witnesses. | Per-relation correctness, penetration and solver failures. Containment should improve without creating contact/appearance regressions. |
| **E5: collateral-edit search** | R2 reduces unnecessary object motion at matched requested-relation success. | Hard anchors; soft L2; sparse/group penalty; random restarts; fixed witness graph; searched witness graph; tiny exhaustive oracle. | Number of changed non-targets, violated preserved relations, success–motion frontier and small-case optimality gap. |
| **E6: degrees of freedom and mechanics** | Gains are not merely due to adding rotations or favorable physics settings. | Translation-only versus SE(3); scale excluded from main track; with/without settling; varied plausible mass/friction. | Geometry and stability separately; report unsupported physical parameters and sensitivity. |
| **E7: judge and view calibration** | R3 improves reliability or labeling efficiency at fixed view/token cost. | Fixed versus random versus coverage versus information/disagreement views; joint versus per-view judgment; alternative judges; option permutations. | Human agreement, observable coverage, false-acceptance rate, accuracy, abstentions and cost. |
| **E8: transfer and stress** | The chosen method generalizes beyond the development assets and corruption types. | Disjoint asset families, generators, camera sets, clutter/scale regimes; known infeasible versus unknown cases. | Worst-group and per-relation performance; failures must not disappear through abstention or filtering. |
| **E9: efficiency and scaling** | A local relation formulation has a useful quality/cost tradeoff. | Increasing objects/Gaussians/edges; cached versus uncached proxies; same hardware and warm-start conditions. | End-to-end and refinement-only time, preprocessing, memory, model calls and accuracy at equal budgets. |
| **E10: representation-changing alternative** | Local repair helps where rigid refinement cannot. | R4 versus refitting, mesh repair and regeneration under matched appearance/shape-change budgets. | Rigid and shape-changing tasks reported separately. Longer-term; not in first pilot. |

**Approval stage boundaries:** E0–E2 diagnose; E3–E4 test R1; E5 tests R2 only if needed; E7 validates evaluation; E6/E8/E9 support final claims. E10 requires a separate scope decision. No stage is authorized by being listed here.

### 6.5 Independent metrics and statistical design

The principal endpoint should be **scene-level success under independently defined relation and nonpenetration checks**, alongside per-relation macro averages. For scene \(s\), define success as every required relation passing its declared tolerance and every relevant solid pair passing nonpenetration. A dropped relation is not removed from that scene's denominator. Use reference geometry for this endpoint only where such geometry is available.

| Dimension | Proposed measurement | Important restriction |
|---|---|---|
| Support/contact | Signed gap distribution, supported area/patch, nonpenetration, and relation semantics. | Nearest-point distance alone can reward a single stray point or side contact. |
| Containment | Fraction of object volume inside the validated cavity; collision with container solid; opening/region semantics. | “Inside exterior bounds” is not sufficient; surface sampling can miss enclosed overlap. |
| Under/clearance | Clearance from actual underside and surrounding solid parts; footprint/semantic condition. | Reference top height is not the underside. |
| Stability | Pose drift, fall/topple rate and perturbation tests under a declared simulator/material model. | Finite simulation is an empirical test, not proof of real-world stability; assess sensitivity. |
| Preservation | Changed non-target count, normalized translation/rotation, preserved-relation violations, shape/attribute checks. | Fixed camera pixels may change because of intended occlusion; separate intrinsic preservation from visibility. |
| Reliability | Error versus acceptance coverage, false acceptance on independently invalid scenes, abstention, timeout and infeasibility status. | Report coverage with accuracy; abstention is not full task success. |
| Visual assessment | Blinded human relation states with an unobservable option; fixed-view and close-up protocols. | Humans label visible evidence separately from physical ground truth. |
| Efficiency | Proxy construction, optimization, rendering, VLM calls and memory separately and end-to-end. | Cached geometry is amortized only when reuse is part of the declared task. |

Tolerance sweeps should be predeclared and expressed in both physical units and normalized object scale. Do not select tolerance after seeing which method wins. Preserve room coordinates and lighting across before/after renders; close-up reframing is a separate condition.

Use paired methods on the same scenes. Resample at the **scene/asset-family level**, not individual relations or views; report confidence intervals and effect sizes. For stochastic methods, use at least three prespecified seeds in the pilot where affordable, then select the final repetition count from observed variance. Deterministic methods do not need repeated identical executions just to produce artificial error bars. Multiple ablations should not become many uncorrected significance claims.

The proposed 60-scene calibration set is a starting budget, not evidence for rare-failure guarantees or precise class-specific bounds. If controlling a low error rate is part of the claim, determine sample requirements and the selection-adjusted statistical guarantee before promising that rate. Calibration units must match inference units; many relations on one scene are not many independent calibration scenes.

### 6.6 Proposed go/no-go thresholds

These are **research decision targets**, not predicted results or established publication standards. Final thresholds should be locked before held-out evaluation after inspecting pilot noise and measurement reliability.

1. **Geometry gate:** the chosen non-oracle representation must improve independent contact/cavity error over boxes and retain thin structures. A candidate planning target is at least 20% relative median-error reduction on development cases with no large worst-case regression; inspect error distributions, not only the median.
2. **R1 gate:** target at least five percentage points higher independent relation success at matched coverage/motion, or a substantial reduction in false acceptance at the same coverage, relative to the strongest deterministic and post-hoc-gating baselines. A paired interval including no meaningful gain does not justify a superiority claim.
3. **R2 gate:** target at least 20% fewer changed non-target objects or normalized collateral motion at comparable success; require no systematic increase in preserved-relation failures. Report the full Pareto curve to prevent cherry-picked operating points.
4. **R3 gate:** demonstrate a reliability or annotation-efficiency gain at equal view/token budget and consistent human labels. If fixed close-ups are equally effective, use them and drop the active-view novelty claim.
5. **Submission gate:** recover reproducibility, complete at least one faithful/adapted strong external comparison with clearly stated deviations, show held-out effects beyond the original 24 scenes, and identify a technical contribution not explained by an off-the-shelf component. Meeting an arbitrary percentage target alone is insufficient.

If the most effective solution is a conventional mesh proxy plus a standard solver, that is useful evidence. Report it honestly and reconsider the paper's contribution instead of adding complexity solely to obtain a novel-sounding pipeline.

## 7. Proposed implementation scope after approval

The following names are design placeholders. **No files or modules in this table have been created for this proposal.** The first phase should add an isolated diagnostic harness rather than silently change the historical baseline.

| Proposed component | Responsibility | Verification needed before scientific use |
|---|---|---|
| `geometry_proxy` | Object-space surface/solid interfaces, cavities, normals, visibility and provenance; interchangeable box/TSDF/mesh backends. | Analytic distance/containment cases; thin walls; open and closed geometry; unit/coordinate consistency. |
| `relation_witnesses` | Contact patches, cavity regions and underside-clearance candidates with support evidence. | Witness coverage and semantic correctness; candidate omission rate measured independently. |
| `geometry_uncertainty` | Local ambiguity features, intervals/ensembles and frozen calibration. | Held-out error calibration, shifts, high-confidence errors, and selection effects. |
| `constrained_refine` | Deterministic surface baseline and uncertainty-aware solver under the same interfaces. | Residuals, convergence failures, finite-difference checks where differentiability is claimed, and independent final-state checks. |
| `edit_search` | Optional R2 witness/movable-set search with a strict budget. | Tiny exhaustive cases, comparison with multistart, preserved-relation regressions. |
| `independent_eval` | Reference-solid metrics, optional dynamics and explicit outcome accounting. | Independent implementation/geometry, not a wrapper around the optimizer's own loss. |
| `judge_calibration` | Fixed camera manifests, per-view results, abstention and blinded human labels. | Camera leakage checks, order bias, inter-rater agreement and held-out judges. |
| `experiment_manifest` | Asset hashes, relation graphs, seeds, revisions, settings, runtime, failures and artifact paths. | Every table value traceable to a manifest; failed jobs retained in denominators. |

Preserve the old implementation as a frozen baseline. Prefer new adapters/interfaces and explicit configuration selection; no default behavior changes until the new mode is independently validated and approved. Do not execute the existing `ray_contact.py correct` path merely because a new proxy can be produced.

### 7.1 Dependencies that must be resolved

- Exact real asset and composed-scene PLYs, labels, source generation/reconstruction settings, and renderer versions must be restored or located.
- A compatible Python/PyTorch/CUDA environment and available GPU/CPU resources must be identified. The earlier environment mismatch is recorded in [assessment.md](assessment.md); this proposal has not rerun the tests or installed packages.
- External baseline release availability, model revisions, access requirements and permitted datasets must be confirmed. A paper/project link does not imply reproducible code is available.
- Physical evaluation needs trustworthy solid geometry; stability evaluation additionally needs justified material and mass assumptions. VLM-estimated properties can be a separately labeled sensitivity condition, not silently treated as ground truth.
- A held-out human-labeling protocol and budget are needed for visual claims. No participants have been contacted or labeling jobs created.

### 7.2 Staged resource and schedule plan

These are provisional work allocations, not measured runtimes or a guarantee of finishing by the deadline. They start **only after explicit approval** and after inputs are available.

| Stage | Approximate work allocation | Deliverable / gate |
|---|---|---|
| Pilot E0–E2 | 3–5 working days; propose a cap of 48 GPU-hours plus bounded CPU geometry jobs; no large-model training; initially use existing assets and analytic geometry. | A bottleneck report showing whether geometry, optimization, or evaluation dominates. Stop at the cap and report incomplete work rather than silently expanding scope. |
| One selected hypothesis | Approximately 5–8 working days, conditional on pilot success. | R1 **or** R2 implementation against mandatory controls; request/agree a separate compute budget after measuring pilot costs. Do not implement all four directions together. |
| Held-out evaluation and reliability | Approximately 5–7 working days, with baseline adapters and labeling preparation overlapping only if approved. | Locked test results, uncertainty, failures, cost, and reproducibility package. |
| Paper integration | Reserve at least several days before November 16 AoE. | Method derivation, claim audit, figures, complete baseline caveats and limitations; no last-minute expansion to unsupported capabilities. |

The proposed D1-main scale could require several thousand solver executions across methods/seeds, plus proxy construction; its feasibility must be estimated from pilot measurements. API cost and human labeling cost cannot be credibly quoted without the chosen models, view count, prompts and annotation rate. For budgeting, 60 D2 scenes × four compared outputs × three raters already means **720 scene-output assessments**, each potentially containing several relations. This is a workload estimate, not a study that has been scheduled.

If approval or asset recovery is delayed, reduce scope or target a later venue. Do not compensate by omitting strong controls, using the 24 development scenes as the final test, or describing unfinished experiments as forthcoming evidence. Registration is not a scientific readiness gate; the result quality is.

## 8. Potential paper contributions and claim discipline

If R1 succeeds, a coherent paper could make at most three central claims:

1. **A relation-specific inference formulation:** ambiguous Gaussian contact/cavity evidence is represented explicitly and propagated to admissible edits and acceptance decisions. This needs a precise algorithm and comparison against point-estimate and post-hoc-gating alternatives.
2. **A useful reliability–editing tradeoff:** independently validated improvement at matched motion, coverage and compute, including failure/unknown cases. This is an empirical claim to be established, not a current result.
3. **A reproducible diagnostic evaluation:** controlled geometric labels and separate human-visible labels reveal when apparent contact and physical validity diverge across methods and asset sources. Releaseable assets, annotation protocol and calibrated evaluation are prerequisites.

R2 should replace or strengthen the first claim only if it yields an independent contribution; do not add it just to lengthen the method. The basic interval argument in R1 is explanatory mathematics, not enough to claim a major new theory. A stronger result would need clearly stated assumptions, a nontrivial algorithmic consequence, and either a useful bound or consistent empirical gains attributable to it.

**Not currently defensible:** first physics-aware Gaussian editor; first surface-based layout optimizer; guaranteed physical validity for arbitrary generated assets; universal stable support; globally minimal edits; general instruction following; cross-view consistency merely because four views are shown; superiority to state of the art without matched runs.

**A viable negative outcome:** clean geometry plus standard optimization may solve the task well while generated-asset ambiguity dominates. That would motivate a representation/evaluation study rather than an embellished optimizer paper. Its competitiveness would still depend on broad evidence, useful diagnostics and a contribution beyond documenting our own implementation's failures.

## 9. Source verification, remaining uncertainty, and review checklist

Literature was checked through **15 October 2026**, prioritizing primary CVPR, ICCV, ECCV, NeurIPS and ICLR sources from 2024–2026, with a few indispensable graphics/robotics sources and older foundations. This is a targeted review, not a claim that every contemporaneous paper has been found. Primary method/evaluation passages were examined for the new close scene systems; more distant alternatives were reviewed at the method/abstract level and are not reproduction recipes.

Important source qualifications:

- The earlier [35-paper matrix](literature_matrix.md) remains the full-author bibliography for the first review. This proposal adds decision-relevant literature without changing those earlier artifacts.
- Scenethesis has a title variation: the accessible arXiv/OpenReview record uses **“A Language and Vision Agentic Framework for 3D Scene Generation”**, while the official project citation uses **“Combining Language and Visual Priors for 3D Scene Generation.”** Its ICLR 2026 status is supported by the official project and indexed proceedings. Verify the final bibliographic export before manuscript submission.
- PhysGS is now verified as **CVPR 2026** in CVF proceedings, rather than left at its 2025 preprint date. Gaussian Opacity Fields is **TOG / SIGGRAPH Asia 2024**, and FisherRF is **ECCV 2024** under the published “Active View Selection and Mapping” title.
- PhyScensis is supported as ICLR 2026 by its official project and conference indexing; detailed reading used the author arXiv paper. A publicly runnable code release was not established in this review.
- PhysTalk and MUSE are treated as preprints here. PhyMix remains a separately marked submission/watchlist item; secondary pages were not used to assert acceptance.
- Some OpenReview full-text requests were blocked by browser verification. Accessible author arXiv pages, official projects, proceedings and indexed primary passages were used instead. No restriction was bypassed and no inaccessible implementation is assumed available.
- The full final LayoutDreamer journal text and RoomPlanner chapter were not completely accessible; detailed preprint and final-version claims are kept distinct. The proposal does not claim their complete code/protocols were audited.

### Additional bibliography beyond the earlier matrix

These entries give exact titles and authors for the main additions; their technical relevance and primary links are in Section 4.

| Work | Authors | Verified status |
|---|---|---|
| Scenethesis: A Language and Vision Agentic Framework for 3D Scene Generation | Lu Ling; Chen-Hsuan Lin; Tsung-Yi Lin; Yifan Ding; Yu Zeng; Yichen Sheng; Yunhao Ge; Ming-Yu Liu; Aniket Bera; Zhaoshuo Li | ICLR 2026; author-paper roster. Proceedings indexing uses Max Li for the last author; title variant noted above. |
| PhyScensis: Physics-Augmented LLM Agents for Complex Physical Scene Arrangement | Yian Wang; Han Yang; Minghao Guo; Xiaowen Qiu; Tsun-Hsuan Wang; Wojciech Matusik; Joshua B. Tenenbaum; Chuang Gan | ICLR 2026; author-paper roster. |
| SceneSmith: Agentic Generation of Simulation-Ready Indoor Scenes | Nicholas Pfaff; Thomas Cohn; Sergey Zakharov; Rick Cory; Russ Tedrake | ICML 2026, official PMLR record. |
| SuGaR: Surface-Aligned Gaussian Splatting for Efficient 3D Mesh Reconstruction and High-Quality Mesh Rendering | Antoine Guédon; Vincent Lepetit | CVPR 2024. |
| 2D Gaussian Splatting for Geometrically Accurate Radiance Fields | Binbin Huang; Zehao Yu; Anpei Chen; Andreas Geiger; Shenghua Gao | SIGGRAPH 2024. |
| Gaussian Opacity Fields: Efficient Adaptive Surface Reconstruction in Unbounded Scenes | Zehao Yu; Torsten Sattler; Andreas Geiger | TOG / SIGGRAPH Asia 2024. |
| GSRecon: Efficient Generalizable Gaussian Splatting for Surface Reconstruction from Sparse Views | Hang Yang; Le Hui; Jianjun Qian; Jin Xie; Jian Yang | ICCV 2025. |
| GeoSVR: Taming Sparse Voxels for Geometrically Accurate Surface Reconstruction | Jiahe Li; Jiawei Zhang; Youmin Zhang; Xiao Bai; Jin Zheng; Xiaohan Yu; Lin Gu | NeurIPS 2025. |
| Bayes' Rays: Uncertainty Quantification for Neural Radiance Fields | Lily Goli; Cody Reading; Silvia Sellán; Alec Jacobson; Andrea Tagliasacchi | CVPR 2024. |
| FisherRF: Active View Selection and Mapping with Radiance Fields Using Fisher Information | Wen Jiang; Boshu Lei; Kostas Daniilidis | ECCV 2024. |
| PhysGS: Bayesian-Inferred Gaussian Splatting for Physical Property Estimation | Samarth Chopra; Jing Liang; Gershom Seneviratne; Dinesh Manocha | CVPR 2026. |
| 3DitScene: Editing Any Scene via Language-guided Disentangled Gaussian Splatting | Qihang Zhang; Yinghao Xu; Chaoyang Wang; Hsin-Ying Lee; Gordon Wetzstein; Bolei Zhou; Ceyuan Yang | ICLR 2025. |
| PhysTalk: Language-driven Real-time Physics in 3D Gaussian Scenes | Luca Collorone; Mert Kiray; Indro Spinelli; Fabio Galasso; Benjamin Busam | arXiv preprint, 2025; venue not verified. |
| Conformal Risk Control | Anastasios N. Angelopoulos; Stephen Bates; Adam Fisch; Lihua Lei; Tal Schuster | ICLR 2024. |

Splat-Nav and SPLANNING are additional robotics comparators linked in Section 4.2; their geometry/probability assumptions deserve a focused technical audit if R1 proceeds. Inclusion here is not a claim that an uncertainty-aware contact editor is novel relative to all robotics planning literature.

### Decisions for author review

- Is the primary scope **rigid relation refinement under uncertain geometry**, with asset-changing repair explicitly separate?
- Should the first authorized stage be limited to **E0–E2**, with an explicit diagnostic/compute cap and a report before implementing R1/R2?
- Can the original assets and a suitable compute environment be supplied or located after approval?
- Is preserving a credible technical claim more important than forcing a November submission if the pilot finds no advantage over standard geometry/solver baselines?

**No implementation or experiment will begin until the author gives explicit approval.** A later approval should identify the selected direction or experiment stage and any resource limits; approval of the diagnostic pilot would not automatically authorize every experiment or the longer-term asset-repair task.
