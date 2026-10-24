# Assignment 2 Report

- **Name**: Gonzalo Domínguez
- **Student ID**: 18691
- **Email**: gdominguez.ieu2022@student.ie.edu
- **Group**: BBADBA 5B

## Business problem, and the cost of being wrong

The business goal is to ensure every person within a construction site is wearing hard hats (for security reasons). To ensure this, cameras have been installed, and a site safety supervisor receives photographs periodically that check whether what the person being photographed is a hard hat or not. The supervisor must decide which images warrant specific attention.

A positive case is an image with at least one annotated exposed head (that is, a possible missing hard hat).

There's 3 actions: Pass (no sufficiently strong signal), Review (ambiguous or low-evidence, pending supervisor inspection), and Flag/Fail (strong signal). 

For the purposes of this project:
- FP: unnecessary check. An inconvenience with a related cost of supervision
- FN: exposure to unsafe conditions leading to potential investigations, litigation, and reputational costs.
Because of this asymmetry, I went for a 8 v. 250 cost (31.25:1), which are illustrative costs. The high missed-case cost intends to motivate high recall instead of merely maximizing accuracy.

Each justified intervention has a related cost too of 2, and each image revision has a related cost of 0.75. s such, a reviewed image containing an exposed head would cost 2.75 (human review does not make intervention free)

*The decision this model supports, who or what acts on it, and what happens
when it's wrong in each direction. Be concrete about the asymmetry: which
mistake is worse, roughly by how much, and why. Everything downstream (your
threshold, how much you send to a person, which metric you care about) has to trace
back to this section.*

## Dataset

*Workspace / project / version on Roboflow Universe, and the link. What one
image represents, what the classes are, and why this dataset actually serves
the problem above rather than merely resembling it.*

Roboflow Universe workspace: joseph-nelson
Project: hard-hat-workers
Version: 10 (raw_AllClasses)
Dataset: https://universe.roboflow.com/joseph-nelson/hard-hat-workers/dataset/10
Task: Object detection
Classes: head, helmet, person.

This is a detection exercise because the supervisor should be examining the actual head/helmet box, and if an image shows many different workers with protected and unprotected heads, a single class for the whole image is too simple.

In containing both labeled hads and helmets in workers scenes, and having train, val and test partitions differentiated, the dataset seemed very relevant to the business problem of insecurity in the work scene. 


## What the dataset's composition implies

*Instances per class, train/val/test sizes, typical object size relative to the
image. Then the interpretation: which per-class numbers will be too thin to
trust, whether the test split resembles the images you'd really see, and what
you therefore can't conclude from your results.*

[RESULT TABLE]

Across the partitions provided, the head-positive rate is very similar (barely a 1.8% difference at most), which is great, but still not proof that cameras installed in the future, the time of day or a different construction site would still output the same distribution.

The test split contains 140 positive images, with many possibly coming from similar, related environments. A SHA-256 comparison found no byte identical images present in more than 1 split, which discards the presence of duplicates, but cannot combat video-adjacent frames. 

Regarding object size, many target objects are very small, which poses some risks for the task (low resolution takes, distant photos, etc.), which in turn made it both impossible and illogical to shrink images to speed up training and instead retained the 640 pixel inputs.

The dataset has way more Helmet than Person, and so the latter is less stable (perhaps the sparse labeling does not cover every person visible in the photos). 

Accuracy is thus also not a viable measure because, with head-positive images being a minority, an "always pass" rule would score high while entirely missing the business case.




## Finetuning and baselines

*Which model and task (detection or classification) and why. A results table on
the same test split: your finetuned model, the pretrained COCO model zero-shot,
and a trivial baseline. What changed, and whether the gap is as large as you
expected.*

I fine-tuned YOLO11s startng from yolo11s.pt rather than random weights. The small Yolo11 variant provides more representational capacity than yolo11n does for these tiny objects I'm dealing with. I however did not do any architecture-wide search — transfer learning seemed anyway appropriate because of the mid-sized dataset, and because this sort of pretraining provides general visual features.

The final experiment used 30 epochs, the previously mentioned 640 pixel inputs, a batch size of 16, deterministic training settings, and early stop patience. Mosaic augmentation was disabled for epochs 21-30. Ultralytics handled the optimizer and standard training augmentation.
- GPU training was performed remotely on NVIDIA T4.
- Predictions for the cost calculations were generated in batches of 4 to avoid GPU memory failures.
- The pred/eval pipeline used NMS IoU of 0.7 and a low inference confidence flor of .001 so that later threshold sweeps are possible.

The same 706 image test-split was used for all 3 detection comparisons.

[table]

Fine-turning learns the categories while zero-shot COCO checkpoint has no directly corresponding classes, with only the shared person label being mapped. 




## Per-class evaluation and failure modes

*Per-class AP50, or per-class precision/recall and the confusion matrix.
Which class carries the headline number, which one is dragging, and what the
model actually confuses with what. Show where it breaks, not just that it
works.*

| Class | Test GT boxes | Precision @ 0.25 | Recall @ 0.25 | AP50 | mAP50–95 |
|---|---:|---:|---:|---:|---:|
| **`head`** | 726 | 0.8965 | 0.9545 | **0.9591** | 0.6719 |
| `helmet` | 1,915 | 0.9242 | 0.9671 | **0.9792** | 0.6882 |
| `person` | 64 | 0.0000 | 0.0000 | 0.0314 | 0.0170 |

The results above are on the held-out test split, with precision and recall calculated at detection confidence .25, with 1-to-1 box matching at IoU .5. 

The model performs quite well on head and helmet objects, but person has almost no usable detections. Its very low test support and the sparse training labels make overall mAP less representative of the performance. 

The saved object-level confusion matrix (test split, confidence 0.25, matching IoU 0.50) is:

| Actual \ Predicted | `head` | `helmet` | `person` | Background (missed) |
|---|---:|---:|---:|---:|
| `head` | 693 | 13 | 0 | **20** |
| `helmet` | 4 | 1,852 | 0 | **59** |
| `person` | 0 | 0 | 0 | **64** |
| Background (unmatched prediction) | **76** | **139** | 0 | — |

Thus, 13 actual heads were called helmets, and 4 actual helmets were called heads, while 20 Head boxes and 59 Helmet boxes were not matched. There were 76 unmatched Head predictions and 139 unmatches Helmet predictions.

Potential error sources includes
- small objects
- Crowded scenes
- overlapping boxes
- inconsistent person labeling



## Cost matrix and operating point

*Your euro cost for each kind of error and where each figure came from. The
base rate you assumed. The cost-against-threshold sweep, and the operating
point you chose. State the assumption that would most change your answer if it
were wrong.*







## What a person still has to check

*Your two thresholds, and what share of cases they send to a person. How many
that is per day, how long one takes to check, and how many people that needs.
Is that realistic? Then your design priced against the three alternatives:
doing nothing, having a person check everything, and letting the model decide
on its own.*

## Limitations & next steps

*Real limitations you found, and concretely how you'd address each one with
more time or data. Not generic hedging.*

## Generative AI use disclosure

*Per the syllabus AI Policy: what you used and how, or "no AI content used."*
