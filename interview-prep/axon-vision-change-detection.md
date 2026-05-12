# Axon Vision — Change Detection Interview Question

**Company:** Axon Vision (Israeli defense / computer-vision startup)
**Question asked:** *"How would you do change detection?"*
**Context:** Defense CV / aerial / surveillance domain — false-alarm rate and robustness matter more than raw accuracy.

---

## Answer structure (4 layers)

### 1. Clarify before solving (signals seniority)
Ask back before diving in:
- *Input modality?* Satellite, aerial/drone, ground EO, video stream.
- *Are images co-registered?* If yes, much simpler. If no, registration is the dominant problem.
- *Definition of "change"?* Binary (changed/not) vs. semantic (new building, new vehicle, vegetation loss).
- *Real-time on edge vs. offline batch?* Determines model size and latency budget.
- *Labels available?* Supervised vs. self-supervised / unsupervised.

This reframing alone separates senior from junior candidates.

### 2. Preprocessing — where most projects actually fail
- **Registration / alignment** — same scene, different time. Feature-based (SIFT/ORB + RANSAC homography) or learned matchers (SuperPoint, LoFTR). Sub-pixel matters; mis-alignment = false positives everywhere.
- **Radiometric normalization** — illumination, seasonal, sensor differences. Histogram matching, relative radiometric normalization, or learned domain adaptation.
- **Geometric correction** — ortho-rectification for aerial/satellite (especially for tall structures with parallax).
- **Atmospheric correction** — for satellite multispectral (haze, scattering).

### 3. Method — simple to SOTA

| Approach | When to use |
|---|---|
| **Image differencing + threshold** ( |I_t2 − I_t1| > τ ) | Baseline; fast; fails on illumination/season |
| **Background subtraction** (MOG2, KNN, ViBe) | Static-camera video |
| **Change Vector Analysis (CVA)** | Multispectral satellite, direction + magnitude |
| **PCA / IR-MAD** | Classical unsupervised CD |
| **Siamese CNN (FC-Siam-Diff / FC-Siam-Conc)** | Standard supervised bi-temporal — two shared-weight encoders → feature diff → decoder → binary mask |
| **BIT (Bitemporal Image Transformer)** | Adds tokenized self-attention; current strong baseline on LEVIR-CD |
| **ChangeFormer** | Pure transformer; SOTA on WHU-CD, DSIFN-CD |
| **Foundation models (SAM, DINOv2)** | Zero/few-shot; embed both, compare in feature space — useful when labels are scarce (common in defense) |
| **Self-supervised pretraining** then fine-tune | Labels expensive |

**Safe default for an interview answer:**
> *"I'd start with a Siamese U-Net trained with weighted BCE + Dice loss to handle class imbalance, benchmark it against differencing as a sanity baseline, and reach for BIT/ChangeFormer if the dataset and compute justify it."*

### 4. Evaluation + production concerns (what they actually care about)

**Metrics**
- Precision, Recall, F1, IoU **on the change class only** — pixel accuracy is misleading because change is ~1–5% of pixels.
- For operational deployment: false alarms per km², per hour, or per scan — operator-facing numbers, not academic ones.

**Class imbalance**
- Weighted BCE, Dice, Focal loss, Tversky loss.
- Hard-negative mining; online hard example mining.

**Failure modes to call out**
- Illumination and shadow changes (sun angle, time of day).
- Seasonal vegetation.
- Parallax on tall structures with off-nadir view.
- Registration drift.
- Cloud/haze in satellite imagery.

**Defense-specific** (score points here for Axon)
- False-alarm rate is the operator's pain point — calibrate threshold for high precision, accept lower recall.
- Human-in-the-loop verification on borderline detections.
- Multi-sensor fusion (EO + IR + SAR) for robustness to weather/lighting.
- Edge inference: quantization (INT8), distillation, TensorRT/ONNX deployment.
- Active learning: operator confirmations feed back into training.

---

## 60-second verbal version

> *"First I'd clarify the modality and whether images are co-registered, because registration is usually the dominant source of false positives. Assuming bi-temporal aerial pairs, the pipeline is: register with a learned matcher like SuperPoint, normalize radiometrically, then run a Siamese encoder that produces per-pixel feature differences fed into a segmentation head, trained with class-balanced loss because change is rare. As a baseline I'd compare to image differencing. For SOTA I'd benchmark BIT or ChangeFormer. In production I care more about false-alarm rate than F1 — defense operators won't trust a system that cries wolf — so I'd calibrate thresholds for high precision and put a human-in-the-loop on borderline detections."*

---

## Likely follow-up questions

1. *"How do you handle illumination changes?"* → radiometric normalization, learn invariant features via contrastive pretraining, augment with synthetic lighting.
2. *"What if you only have 100 labeled pairs?"* → self-supervised pretrain on unlabeled pairs (SimCLR/MoCo or DINOv2), then fine-tune; or use SAM/foundation features in a non-parametric comparator.
3. *"How would you deploy this on a drone?"* → distill to MobileViT/EfficientNet-Lite Siamese, INT8 quantize, TensorRT, fp16 on Jetson; budget per-frame latency.
4. *"How do you evaluate operationally?"* → false alarms per km²/hour at fixed recall, ROC at the operator's working point, not mean IoU.
5. *"What about video / multi-temporal?"* → temporal model (3D conv, ConvLSTM, or temporal transformer) over a window; or per-frame CD + temporal smoothing of the change mask.

---

## Connecting to your background (use these as proof points)

- **ISI vessel detection** across optical / SAR / multispectral → directly relevant: tiling huge images, multi-sensor, class imbalance, AIS matching as a weak-label / verification signal.
- **IAI sensor fusion + tracking (JPDA / graph matching)** → answers "how do you fuse multiple sensors" and "how do you handle uncertain detections."
- **Flow Retail real-time tracking @ 20 FPS** → answers edge / latency / deployment questions.
- **GM free-space / lane annotation** → answers data quality and annotation strategy.

---

## Knowledge Distillation

Distillation will almost certainly come up — defense CV companies live on "big model in lab → small model on drone/edge box." Full prep moved to its own file so it can be reused across companies (Bluewhite, Black Rover, Wayve, etc.):

→ [knowledge-distillation.md](knowledge-distillation.md) — theory, types, best practices, framework comparison, defense-edge workflow, follow-up questions, company links.

**Change-detection-specific KD note:** for a CD student, the soft targets that actually move the needle are the **teacher's predicted change masks on unlabeled bi-temporal pairs**, not just labeled logits. Defense companies have a lot of unlabeled paired imagery and very few labels — exploit that.

---

## Open issues / things to verify before next round

- [ ] Confirm the company is **Axon Vision** (Israeli defense CV), not a different "Axos."
- [ ] Are they doing aerial/drone, ground, or satellite change detection? Determines which proof points to lead with.
- [ ] Is the next round technical deep-dive, system design, or coding?
- [ ] What's their actual deployment target? Jetson? Custom ASIC? Cloud? Determines which distillation/quant story to lead with.
