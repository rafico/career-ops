# Knowledge Distillation — Interview Prep

**Scope:** General KD prep, reusable across any CV/edge-deployment interview (Axon Vision, PixelSight, Bluewhite, Black Rover, Wayve, Mobileye, Innoviz, etc.). Linked from `axon-vision-change-detection.md`.

**When this comes up:** Anywhere the conversation touches edge deployment, drones, embedded inference, real-time CV, or "how would you make this model run on X." For defense / surveillance / autonomous-driving / agtech roles it is almost guaranteed.

---

## 30-second framing

> *"Distillation is how I close the gap between a research-grade teacher and a deployable student. Teacher is whatever wins the offline benchmark — a ChangeFormer, a BIT, a Mask2Former, a large ViT. Student is a lightweight architecture sized for the target hardware's latency and power budget. I distill at multiple levels — output logits plus intermediate features — then run QAT and compile to TensorRT or the equivalent. Realistic win: 5–15× speedup with a 1–3% F1 drop. Anyone promising 'free lunch' KD hasn't shipped it."*

---

## Theory in one paragraph

Hinton 2015: student matches teacher's softmax with temperature **T**. Loss:
`L = α · T² · KL(softmax(z_s / T), softmax(z_t / T)) + (1 − α) · CE(z_s, y_hard)`.
`T²` keeps gradients scaled. Typical `T ∈ [2, 8]`, `α ∈ [0.5, 0.9]`. The dirty secret: for dense-prediction CV (segmentation, detection, change detection), **logit distillation alone underperforms feature distillation** — you want the student to see what the teacher *thinks*, not just what it *says*.

---

## Distillation types — what to use when

| Type | What you match | Use when |
|---|---|---|
| **Response / logit KD** | Output logits with temperature | Classification, baseline |
| **Feature KD** (FitNets, AT) | Intermediate feature maps (L2 or normalized) | Detection, segmentation, change detection |
| **Relation KD (RKD)** | Pairwise relations between samples | Metric learning, Siamese networks |
| **Attention transfer (AT)** | Attention maps from teacher | Transformer teachers |
| **Channel-Wise Distillation (CWD)** | Channel-normalized feature softmax | Dense prediction — strong default |
| **Masked Generative Distillation (MGD)** | Mask + reconstruct teacher features | Detection — current SOTA family |
| **Self-distillation** | Student is same architecture | Squeeze a bit more from a fixed model |
| **Online / mutual (DML)** | Two students teach each other | No pre-trained teacher available |
| **Data-free KD** | Synthesize data from teacher | Privacy / classified data constraints — relevant for defense |

---

## Best practices (the part that separates senior from junior)

### Loss design
- Combine 2–3 losses, don't pick one: `CE(hard) + λ₁ · KD(logits) + λ₂ · feature_loss`. Start with `λ₁ = 1.0`, `λ₂ = 1e-4` to 1e-2 depending on feature norm.
- For dense prediction, normalize features before matching (channel-wise L2 norm) — raw L2 on un-normalized features is dominated by activation magnitude.
- Use **CWD** as your default feature loss for segmentation / CD / detection heads. It consistently outperforms vanilla feature MSE.

### Architecture
- **Capacity gap matters.** Teacher / student parameter ratio > ~6× usually fails. Insert a **teacher assistant (TA)** — an intermediate model — and distill in two stages.
- Match feature scales. If teacher is at 32×32 and student at 16×16, downsample teacher or upsample student before matching. Don't skip layers blindly.
- Initialize the student backbone from ImageNet (or in-domain SSL) weights, not random. Random-init students need 2–3× more epochs and often plateau lower.

### Training
- Teacher in **eval mode** (no dropout, no BN updates). Common bug.
- Same augmentation pipeline for teacher and student in a single forward pass. Different aug = noisy targets.
- Do not freeze the student backbone. Freezing kills the point of distillation.
- Distill **before** quantization. Then quantize-aware fine-tune (QAT) on top. Order: **distill → QAT → compile**. Doing QAT first locks the student into bad representations.

### Data
- Distillation is data-hungry — soft labels carry information only when the dataset is rich enough.
- If labels are scarce, **unlabeled in-domain data + teacher pseudo-labels** is often more valuable than the labeled set. This is the trick that works in production at defense / AD / agtech companies that have lots of unlabeled domain data and few labels.
- For change detection: feed teacher the unlabeled bi-temporal pairs, use predicted change masks as soft targets.

### Evaluation
- Always report teacher vs. student on the **same test set, same metric**. Papers love to compare apples to oranges.
- Track latency **on the target hardware**, not on your dev GPU. A model 3× faster on a V100 might be only 1.2× faster on a Jetson Orin.
- For defense/AD applications, calibrate threshold for **operator-facing metrics** (false alarms per km², per hour, per scan) — not F1.

---

## Frameworks actually used in industry

| Framework | What it is | When you'd use it |
|---|---|---|
| **Plain PyTorch** | Write the distillation loop yourself | **Most common in startups and research.** KD losses are too task-specific for a generic framework to nail. Default choice. |
| **[MMRazor](https://github.com/open-mmlab/mmrazor)** (OpenMMLab) | Purpose-built KD + pruning + NAS, integrates with MMDetection / MMSegmentation | Best off-the-shelf KD framework for CV. Has CWD, MGD, FGD, RKD pre-implemented. Strong choice when training is already in MM ecosystem. |
| **[NVIDIA TAO Toolkit](https://developer.nvidia.com/tao-toolkit)** | End-to-end distill + quantize + compile, NVIDIA-native | If the company is on the NVIDIA stack (Jetson, DRIVE, DeepStream). Common in defense / automotive. Less flexible but the deployment story is seamless. |
| **HuggingFace `transformers` + `accelerate`** | DistilBERT-style methodology, vision via accelerate | For ViT / DINOv2 / SAM teachers. Patterns reusable for CV. |
| **[TorchDistill](https://github.com/yoshitomo-matsubara/torchdistill)** | Academic PyTorch KD framework with many algorithms | Good reference implementations; less production-hardened. |
| **Microsoft NNI** | NAS + pruning + KD | Rarely seen in CV production. |
| **TF Model Optimization Toolkit** | Google's KD + pruning + quantization | Only if team is on TF — declining share in CV. |
| **ONNX Runtime / ORT Mobile** | Inference, not training | Distillation **target** — compile the distilled student here. |
| **TensorRT** | NVIDIA inference compiler with INT8 calibration | The final deployment step. INT8 calibration set quality matters more than people admit. |

**Honest opinion to share in interviews:**
> *"For most CV distillation work I write the loop in PyTorch directly, because the loss combinations and feature alignment are too task-specific for a generic framework to nail. I reach for MMRazor when I want pre-built CWD or MGD heads for detection or segmentation. If the team is committed to the NVIDIA stack I'd use TAO — it removes glue code between distillation and TensorRT deployment, which matters when you're shipping to a Jetson."*

---

## Production workflow for a defense / edge CV deployment

1. **Train teacher** — large transformer or CNN on labeled data, full precision, large batch.
2. **Generate soft pseudo-labels** — run teacher over unlabeled in-domain data (defense/AD/agtech companies have lots of this).
3. **Define student** — lightweight architecture (MobileNetV3-Small backbone, EfficientNet-Lite, FastViT) sized for target hardware budget.
4. **Distill** — multi-level: encoder feature maps at multiple scales (CWD loss) + output logit KD (T=4) + hard-label CE on labeled subset. Example weighting: `1.0 · CE + 1.0 · KD_logits + 5e-3 · CWD`.
5. **QAT** — quantization-aware fine-tune with the same KD setup but in INT8 fake-quant mode.
6. **Export ONNX** → **TensorRT INT8 compile** with a calibration set of ~500 representative samples.
7. **Validate on target hardware** — Jetson Orin or whatever. Measure: latency, false-alarm rate at fixed recall, memory footprint, power draw.

Typical result on this pipeline: **8–12× latency reduction, ~2% F1 drop** vs. the teacher. That's the number to quote if asked.

---

## Likely follow-up questions

1. *"What's the capacity gap problem?"* → student too small to absorb teacher. Insert a TA (intermediate model) and distill in stages.
2. *"Why feature KD over logit KD for segmentation/detection?"* → spatial structure lives in features, not in final logits; logits lose it.
3. *"What if you have no labeled data?"* → teacher pseudo-labels on unlabeled in-domain data; data-free KD as a last resort.
4. *"Distillation vs. quantization vs. pruning — what's the order?"* → distill first, then QAT, then prune + fine-tune, then compile. Distilling a pruned model rarely works; pruning a distilled model does.
5. *"Have you ever had distillation fail?"* → yes; almost always either (a) capacity gap too large, (b) feature scales mismatched, (c) teacher overfit so soft labels are noisy. Diagnose by checking teacher-student agreement on a held-out set before training.
6. *"How does KD interact with self-supervised pretraining?"* → SSL gives a strong backbone; distillation gives task-specific shaping. Use them together: SSL pretrain → distill from a task-trained teacher.
7. *"How would you distill a transformer teacher into a CNN student?"* → cross-architecture KD is harder. Match at logit + attention map level (treat attention as a feature). Or distill into a hybrid (MobileViT) instead of pure CNN.

---

## Industry context — companies likely to ask about this

These three companies all run CV at the edge and would care about KD. Public KD writeups are thin for all of them (none publish detailed engineering blogs), but the linked pages give you enough context to talk about *what they ship* and reverse-engineer what KD looks like in their stack. Treat these as conversation hooks, not technical references.

### Wayve — autonomous driving (UK)
- **Site:** [wayve.ai](https://wayve.ai/) — embodied AI for AV2.0 self-driving
- **Science page:** [wayve.ai/science](https://wayve.ai/science/) — foundation-model-for-driving thesis
- **GAIA-2 (generative world model):** [wayve.ai/thinking/gaia-2](https://wayve.ai/thinking/gaia-2/) — explicitly names distillation as future work for inference acceleration:
  > *"Future work will explore techniques such as distillation and model parallelism at inference to further accelerate video synthesis without compromising quality."*
- **Scaling GAIA-1:** [wayve.ai/thinking/scaling-gaia-1](https://wayve.ai/thinking/scaling-gaia-1/) — 9B-parameter world model, motivates why distillation matters for them
- **Why they care about KD:** foundation-scale training models, automotive-grade ECU at inference — classic 100×+ compression problem.
- **What to say if Wayve comes up:** "Their public story is foundation models for driving with a generative world model in the loop. Deployment to vehicle compute is a clear distillation/compression problem — their own GAIA-2 paper names distillation as next work."

### Bluewhite — autonomous agriculture (Israel/US)
- **Site:** [bluewhite.ai](https://www.bluewhite.ai/) — retrofit autonomy kit for tractors
- **How it works:** [bluewhite.ai/how-it-works](https://www.bluewhite.ai/how-it-works) — perception stack: LiDAR + cameras + GPS + IMUs, sensor fusion, day/night, GPS-denied navigation
- **Solution:** [bluewhite.ai/solution](https://www.bluewhite.ai/solution) — Pathfinder retrofit kit + Compass SaaS analytics
- **Why they care about KD:** tractor-mounted compute box, multi-camera real-time perception in field conditions, fleet rollout — every watt and millisecond costs money at scale.
- **What to say if Bluewhite comes up:** "Retrofit autonomy across tractor models implies hardware variability, so they need a model family that distills down to whatever compute budget each kit has. Multi-camera real-time detection + classification + free-space estimation is exactly the workload that benefits from feature-level KD into a MobileViT-class student plus INT8 on a Jetson-class SoC."

### Black Rover — dual-use surveillance (camera + drone CV)
- **Site:** [blackrover.com](https://www.blackrover.com) — dual-use surveillance tech, plug-and-play integration with cameras and drones
- **Why they care about KD:** drone payload and surveillance camera SoCs have tight power/thermal budgets; "plug-and-play" with existing VMS implies they need to deploy across heterogeneous hardware → strong KD + quantization story.
- **What to say if Black Rover comes up:** "Dual-use surveillance with plug-and-play camera and drone integration is a hard deployment story — you need one model family that can be distilled and quantized down to whatever the host hardware allows. Their value-add is the perception model, so optimizing it across heterogeneous edges is the core engineering problem."
- **Note:** company name spelling worth confirming — earlier reference was "blackrover"; the matching site is `blackrover.com`.

---

## What to verify before quoting these in an interview

- [ ] Confirm Black Rover is the intended company (not Black-i Robotics or another similarly-named firm).
- [ ] If you cite the Wayve distillation quote, attribute to *GAIA-2 paper / blog post, "Future Work" section.*
- [ ] Don't claim Bluewhite or Black Rover have published KD research — they have not (publicly). Frame the link as "this is the deployment problem they have, here is how I'd solve it."
