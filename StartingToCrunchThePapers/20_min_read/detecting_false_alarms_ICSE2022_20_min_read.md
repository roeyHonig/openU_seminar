# Detecting False Alarms from Automatic Static Analysis Tools: How Far Are We?

**Authors:** Hong Jin Kang, Khai Loong Aw, and David Lo  
**Publication:** ICSE 2022, May 21–29, 2022  
**Affiliation:** Singapore Management University, Singapore  
**DOI:** https://doi.org/10.1145/3510003.3510214

---

# Abstract

Automatic Static Analysis Tools (ASATs), such as FindBugs, detect potentially problematic code by matching source code against predefined bug patterns. Although these tools can identify real defects at relatively low cost, they also produce many false alarms. A false alarm is a warning that developers do not consider actionable, either because it does not represent a genuine problem in context or because fixing it is not worthwhile or is too risky.

To reduce the effort required to inspect warnings, researchers have proposed machine-learning techniques that classify warnings as either actionable or false alarms. A prominent approach uses a collection of 23 features known as the **Golden Features**, identified by Wang et al. [53]. Previous studies reported remarkably strong results, including F1 scores approaching 0.9 and AUC values approaching 1.0. These results suggested that distinguishing actionable warnings from false alarms might be an almost solved problem.

This paper investigates why the Golden Features appear so effective and whether their reported performance reflects realistic conditions.

The authors identify two major experimental problems:

1. **Data leakage:** Five Golden Features use warning labels calculated with information from a future reference revision. Consequently, the classifier receives information that would not be available when predicting whether a current warning is actionable.
2. **Data duplication:** Many warning instances appear in both the training and testing datasets. The classifier can therefore encounter warnings it has effectively already seen during training, making the evaluation unrealistically easy.

After addressing these problems, the authors find that the Golden Features SVM performs substantially worse than previously reported and does not consistently outperform a simple baseline that predicts every warning is actionable.

The paper also investigates the reliability of the **closed-warning heuristic**, a procedure used to generate labels automatically. This heuristic assumes that a warning that disappears from a later revision is actionable, while a warning that remains is a false alarm. The authors demonstrate that this assumption is unreliable: warnings may disappear for reasons unrelated to fixing a bug, and actionable warnings may remain open for years.

The central conclusion is that detecting actionable static-analysis warnings remains an open research problem. Reliable evaluation requires realistic experimental designs, trustworthy labels, appropriate baselines, and careful examination of seemingly impressive results.

# 1. Introduction

Automatic Static Analysis Tools, including FindBugs, analyze source code to identify patterns associated with potential defects. For example, a tool may identify code that could dereference a null pointer, use an inefficient object-construction method, or contain incorrect synchronization.

These tools are valuable because they can detect potential problems without requiring developers to execute every possible program path. They can be used during local development, continuous integration, and code review.

However, a warning does not necessarily mean that the code is definitely incorrect.

Static analyzers frequently identify suspicious patterns without knowing the complete runtime context or the developer's intentions. A warning may therefore be technically consistent with the tool's detection rules while not representing a problem that needs to be fixed in the particular program.

For example, a tool might warn that a certain operation could fail under particular conditions. The surrounding code, however, may guarantee that those conditions never occur. Alternatively, a warning might identify an inefficient implementation that is intentional and whose replacement would introduce unnecessary risk.

The paper adopts a practical understanding of false alarms: warnings are considered unactionable when developers do not consider them worth acting upon. This includes warnings that do not represent genuine bugs and warnings whose correction is too risky or costly.

Consequently, the problem is not simply determining whether a static analyzer has detected a suspicious code pattern. It is determining whether the resulting warning is something a developer should act upon.

## 1.1 Machine learning for actionable-warning detection

Researchers have proposed using machine learning to classify warnings automatically. The objective is to reduce the number of warnings developers must inspect and prioritize warnings that are more likely to be actionable.

A machine-learning classifier learns a relationship between measurable characteristics of warnings and their known labels. These characteristics are called **features**.

Features can describe:

- The source code containing a warning.
- The file and method in which the warning occurs.
- The warning's pattern, type, and priority.
- The history of the file and its changes.
- The historical behavior of similar warnings.

Wang et al. [53] systematically evaluated 116 previously proposed features and identified 23 particularly effective features, collectively known as the Golden Features.

Subsequent studies reported impressive performance using these features. For example, earlier work reported Recall as high as 96%, Precision as high as 98%, and AUC as high as 0.995.

These results suggested that the problem might be intrinsically easy: perhaps a small set of features could reliably distinguish actionable warnings from false alarms.

The authors of the present paper question this conclusion. They investigate whether the strong performance results from genuinely useful patterns or from problems in how the experiments and datasets were constructed.

## 1.2 The role of labels and the warning oracle

To train a supervised machine-learning classifier, researchers need examples with known answers.

In this setting, each warning must be assigned a label indicating whether it is actionable or a false alarm. These labels are referred to as **ground-truth labels**.

Ideally, the labels would reflect reliable judgments from developers who understand the code and the warning. However, manually examining large numbers of warnings is expensive and difficult to scale.

Previous studies therefore used an automated labeling procedure called the **closed-warning heuristic**.

The heuristic compares a warning at one point in a project's history with the warning's status in a later reference revision:

- If the warning disappears and its file still exists, it is labeled actionable.
- If the warning remains, it is labeled a false alarm.
- If the file has been deleted, the warning is labeled unknown and excluded from the dataset.

The procedure that assigns these labels is called the **warning oracle** in this paper. The closed-warning heuristic is the specific rule used as that oracle.

The distinction is important. The heuristic does not predict whether a genuinely new warning will be actionable. Instead, it uses historical information to assign labels to examples that researchers will subsequently use to train and evaluate a classifier.

The machine-learning classifier and the warning oracle therefore have different responsibilities:

- **Warning oracle:** Supplies the labels used as answers for historical examples.
- **Feature extractor:** Calculates the characteristics of each warning.
- **Machine-learning classifier:** Learns from the features and labels to predict the label of a warning.

The closed-warning heuristic is convenient because it enables large datasets to be constructed automatically. However, its assumptions may not reflect what developers actually consider actionable.

## 1.3 Research questions and contributions

The paper addresses two principal research questions:

**RQ1: Why do the Golden Features work?**

The authors reproduce earlier experiments, investigate which features drive predictions, and examine the effects of data leakage and data duplication. They then evaluate the Golden Features under a more realistic experimental setting.

**RQ2: How suitable is the closed-warning heuristic as a warning oracle?**

The authors investigate whether the heuristic produces consistent labels when the reference revision changes and whether those labels agree with human judgments and developer-maintained FindBugs filter files.

The paper's main contributions are:

1. Identifying data leakage and data duplication that substantially inflate previously reported classifier performance.
2. Demonstrating that the closed-warning heuristic does not always produce reliable labels.
3. Showing that the apparent effectiveness of the Golden Features is sensitive to experimental design and dataset quality.
4. Providing recommendations for more reliable evaluation and benchmark construction.

# 2. Background

## 2.1 Automatic Static Analysis Tools

Static-analysis tools examine source code without necessarily executing it. They apply predefined rules or analyses to identify code patterns that may indicate defects.

FindBugs, the tool studied in this paper, contains hundreds of bug patterns. Examples include possible null-pointer dereferences, impossible class casts, and incorrect synchronization.

These tools can detect genuine bugs and are used in real development workflows. Nevertheless, their warnings are not guaranteed to represent defects that developers should fix.

A warning can be unactionable because the suspected behavior cannot occur in the actual context, because the behavior is intentional, or because changing the code is not worthwhile.

The distinction between a suspicious code pattern and an actionable defect motivates the research problem.

## 2.2 The Golden Features

The Golden Features are a set of 23 features identified by Wang et al. [53] as effective for classifying FindBugs warnings.

They include characteristics of the code, the file, the warning, and the project's history.

| Feature category | Examples |
|---|---|
| Warning combination | Warning context in method, warning context in file, warning context for warning type, defect likelihood for warning pattern |
| Code characteristics | Comment-to-code ratio, method depth, file depth, number of methods in a file, number of classes in a package |
| Warning characteristics | Warning pattern, warning type, warning priority, package |
| File history | File age, file creation, developers |
| Code analysis | Parameter signature, method visibility |
| Code history | Lines of code added in a file over recent revisions, lines added in a package over a recent period |
| Warning history | Warning lifetime by revision |

Two groups of features are particularly important to the paper.

**Warning context features** estimate the distribution of actionable warnings and false alarms in a relevant population, such as warnings in the same method or file.

**Defect likelihood features** estimate the proportion of warnings associated with a particular bug pattern that are actionable.

These features are based on the intuition that warnings in the same context may behave similarly. For example, if developers have repeatedly fixed warnings in a particular file, other warnings in that file might also be worth fixing.

However, these features require information about which warnings are actionable. This dependency becomes central to the paper's discovery of data leakage.

## 2.3 The machine-learning pipeline

The overall classification process consists of four stages:

1. Collect historical warnings.
2. Assign labels to those warnings using a warning oracle.
3. Calculate the features and train a classifier using labeled examples.
4. Calculate features for a new warning and use the trained classifier to predict its label.

The Golden Features are the classifier's inputs. They are not themselves the classifier or the warning oracle.

The authors focus on a Support Vector Machine (SVM), a supervised learning algorithm that learns a decision boundary between classes. Here, the classes are actionable warnings and false alarms.

An SVM must be trained on examples before it can make predictions. The quality of those predictions depends on the features, the training examples, the reliability of the labels, and whether the evaluation accurately represents real use.

## 2.4 Training, testing, and reference revisions

A revision is a snapshot of a software project's state at a particular point in its version history.

The paper uses three distinct revisions:

```text
PAST                                               FUTURE

[Training revision] ---> [Testing revision] ---> [Reference revision]
       |                        |                         |
       v                        v                         v
  Learn from              Evaluate on              Generate labels
  historical data         test examples             using future state
```

The **training revision** provides historical information used to train the model.

The **testing revision** represents the simulated time at which the trained model is evaluated on warnings.

The **reference revision** occurs later and is used by the closed-warning heuristic to determine labels for historical warnings.

Previous studies commonly placed the reference revision two years after the testing revision.

The reference revision is not a second training or testing dataset. Its purpose is to help determine the labels assigned to warning instances.

This distinction is essential because using future information to create a label is different from using that information to calculate a feature that the classifier receives as input.

## 2.5 Evaluation metrics

The paper evaluates classifiers using Precision, Recall, F1, and AUC.

A warning is treated as the positive class when it is actionable.

| Actual label | Predicted actionable | Predicted false alarm |
|---|---|---|
| Actionable | True Positive (TP) | False Negative (FN) |
| False alarm | False Positive (FP) | True Negative (TN) |

A **false positive** occurs when the classifier predicts that a warning is actionable even though it is actually a false alarm. A **false negative** occurs when an actionable warning is incorrectly classified as a false alarm.

### Precision

Precision measures how often warnings predicted to be actionable really are actionable.

\[
\text{Precision} = \frac{TP}{TP+FP}
\]

High Precision means developers can place greater confidence in warnings that the classifier recommends addressing.

### Recall

Recall measures how many of all genuinely actionable warnings the classifier successfully identifies.

\[
\text{Recall} = \frac{TP}{TP+FN}
\]

High Recall means the classifier misses relatively few actionable warnings.

### F1

F1 combines Precision and Recall through their harmonic mean:

\[
F1 = 2 \times
\frac{\text{Precision}\times\text{Recall}}
{\text{Precision}+\text{Recall}}
\]

F1 ranges from 0 to 1. It is especially useful when both Precision and Recall matter and the classes are imbalanced.

### AUC

AUC stands for Area Under the Receiver Operating Characteristic Curve.

The ROC curve plots the True Positive Rate against the False Positive Rate as the classifier's decision threshold changes.

- AUC = 1 indicates perfect separation.
- AUC = 0.5 indicates random-level discrimination.
- AUC below 0.5 indicates performance worse than random ranking under the usual positive-class interpretation.

AUC and F1 measure different properties. F1 evaluates the classifier at a particular decision threshold, whereas AUC summarizes its ability to distinguish the classes across thresholds.

## 2.6 The strawman baseline

A baseline is a reference method used to determine whether a more sophisticated technique provides meaningful improvement.

The principal strawman baseline in this paper predicts that **every warning is actionable**.

This method does not inspect the code, calculate sophisticated features, or learn a complex decision boundary.

If the Golden Features SVM cannot outperform this trivial strategy, its practical value is questionable.

The strawman baseline is not the warning oracle. The oracle generates labels for historical warnings; the baseline is a simple prediction rule evaluated against those labels.

# 3. Study Design

This section explains how the authors structure their investigation and evaluate the Golden Features and the closed-warning heuristic.

## 3.1 RQ1: Why do the Golden Features work?

The authors begin by reproducing previous experiments using the datasets studied by Wang et al. [53] and Yang et al. [56].

The purpose of reproducing earlier results is to establish whether the reported high performance can be obtained using the original experimental setup.

They then investigate why the classifier performs so well.

Rather than assuming that high performance proves that the features are genuinely useful, they examine the contribution of individual features and progressively simplify the classification approach.

They use LIME, an explainable-AI technique, to identify which features contribute most strongly to individual predictions.

LIME stands for Local Interpretable Model-agnostic Explanations. It helps explain a model's predictions by identifying influential input features.

The authors sample 50 predictions made by the Golden Features SVM and use LIME to identify important features. They find that warning context in file and warning context in package appear among the top three features for every sampled prediction.

This observation directs their attention toward these features and how they are calculated.

On inspecting the feature-extraction implementation, the authors discover that five features depend on warning labels derived from the future reference revision.

They then investigate the contribution of these leaked features and the effects of duplicated warning instances in the training and testing datasets.

Finally, they evaluate the Golden Features after addressing both problems.

## 3.2 RQ2: How suitable is the closed-warning heuristic?

The second research question investigates whether the closed-warning heuristic is a reliable way to determine the ground-truth labels of warnings.

A useful warning oracle should satisfy two important properties.

**Robustness:** Small changes in the choice of reference revision should not lead to substantially different conclusions about the dataset or classifier.

**Agreement with human judgment:** The labels should reasonably reflect whether developers consider warnings actionable.

The authors examine the heuristic from three perspectives:

1. **Reference-revision sensitivity:** They compare labels generated using reference revisions two, three, and four years after the testing revision.
2. **Unconfirmed actionable warnings:** They manually inspect warnings labeled actionable by the heuristic to determine whether the warnings disappeared because of an actual fix.
3. **Unconfirmed false alarms:** They examine developer-maintained FindBugs filter files to determine whether warnings that remain open have actually been confirmed as false alarms.

The purpose is to determine whether the heuristic's assumptions reliably represent developer decisions.

## 3.3 Experimental setting

The initial experiments use datasets from nine Java projects previously studied in the literature.

The testing revisions correspond to the last revisions checked into the main branches on January 1, 2014. The training revision is approximately six months before the testing revision.

The authors obtain 31,058 warning instances by running FindBugs on the training and testing revisions.

Only about 14.1% of the warnings in the initial dataset are labeled actionable, demonstrating substantial class imbalance.

The authors use the same testing revisions as the earlier studies so that they can reproduce and compare the reported results.

They train a separate model for each project.

The reference revision is later than the testing revision and is used to assign the labels according to the closed-warning heuristic.

This setup allows the authors to investigate both the original experimental claims and the validity of the process used to produce the labels.

# 4. Analysis of the Golden Features

The authors begin by reproducing the impressive performance reported by previous studies. They then investigate whether the experimental setup makes the classification task artificially easy.

Their investigation reveals two distinct problems: data leakage and data duplication.

## 4.1 Reproducing the reported performance

Using the original experimental setting, the authors successfully reproduce the high performance of the Golden Features SVM.

The average F1 score is 0.88, with values ranging from 0.65 to 0.95. AUC values reach 0.99.

These results initially support the conclusion that the Golden Features can distinguish actionable warnings from false alarms very effectively.

However, reproducing an experimental result does not establish that the experiment accurately measures real-world performance.

A classifier can obtain excellent results when its inputs contain information about the answers or when the testing dataset includes examples already encountered during training.

The authors therefore investigate which features drive the predictions and how the training and testing examples were collected.

## 4.2 LIME identifies influential features

The authors use LIME to understand which features contribute most strongly to the Golden Features SVM's predictions.

They sample 50 predictions and examine the most influential features for each prediction.

Two features consistently stand out:

- Warning context in file.
- Warning context in package.

Both appear among the top three features for every sampled prediction.

These features estimate how actionable warnings tend to be within a relevant context. Their strong influence suggests that the model relies heavily on historical patterns among groups of warnings.

The authors therefore inspect the implementation of these features.

This leads to the discovery that the feature calculations use labels determined by comparing warnings against the future reference revision.

The crucial insight is that the classifier is receiving input derived from the same future information used to establish the answers against which its predictions are evaluated.

LIME does not prove that leakage exists. Instead, it helps identify suspiciously influential features, prompting the authors to inspect their implementation and discover the underlying problem.

## 4.3 Data leakage in the Golden Features

### 4.3.1 What is data leakage?

Data leakage occurs when information that should not be available to a model at prediction time enters the model's inputs or training process in a way that gives it an unrealistic advantage.

In this paper, the problem arises because five Golden Features use labels obtained through the closed-warning heuristic.

The heuristic examines a future reference revision to determine whether a warning eventually disappears.

That future information is used to calculate features for warnings at the testing revision, even though the classifier is supposed to predict whether those warnings are actionable at that earlier point in time.

### 4.3.2 The five leaked features

The authors identify the following five features as affected by data leakage:

1. Warning context in method.
2. Warning context in file.
3. Warning context for warning type.
4. Defect likelihood for warning pattern.
5. Discretization of defect likelihood.

The first three estimate the distribution of actionable warnings and false alarms within different warning populations.

The fourth estimates the proportion of actionable warnings associated with a specific FindBugs bug pattern.

The fifth measures how much defect likelihood varies across bug patterns belonging to a category.

All five require information about which warnings are actionable.

### 4.3.3 How the leakage occurs

Consider a warning \(W_t\) observed at the testing revision.

The warning context in file feature attempts to answer a question such as:

"What proportion of warnings in this file are actionable?"

To calculate this proportion, the original implementation uses the labels of warnings in the file. Those labels were generated by checking whether the warnings eventually disappeared in the future reference revision.

The resulting information flow is:

```text
Testing revision
      |
      v
Warnings in a file or other relevant population
      |
      v
Closed-warning heuristic
      |
      v
Future-derived labels
      |
      v
Warning context / defect likelihood features
      |
      v
SVM prediction
```

The problem is especially serious because the warning being predicted can contribute its own future-derived label to the feature calculation.

The classifier may therefore receive information derived from the very answer it is supposed to predict.

In real use, a developer cannot know whether a warning will disappear two years in the future. That information cannot legitimately be used to predict the warning's current actionability.

### 4.3.4 Why leakage inflates performance

The leaked features make the classification task easier than it would be in a realistic setting.

If a feature already contains information about whether warnings will eventually be labeled actionable, the classifier does not need to learn that relationship entirely from legitimate historical evidence.

Instead, it can exploit information that indirectly reveals the target label.

The authors demonstrate the impact by removing the five leaked features.

The average F1 falls from 0.88 to 0.38.

| Experimental setting | Average Precision | Average Recall | Average F1 |
|---|---:|---:|---:|
| All Golden Features | 0.84 | 0.94 | 0.88 |
| Without leaked features | 0.26 | 0.70 | 0.38 |
| Without data duplication | 0.88 | 0.93 | 0.90 |
| Without leaked features and duplication | 0.27 | 0.57 | 0.31 |

The results show that the original performance depends heavily on the leaked features.

Removing them causes a dramatic decline, suggesting that the earlier results substantially overestimated the model's real predictive ability.

However, data leakage is only one part of the problem.

## 4.4 Data duplication between training and testing

### 4.4.1 What is data duplication?

In a properly designed evaluation, the test dataset should represent examples on which the model has not already been trained.

Data duplication occurs when the same warning instance appears in both the training and testing datasets.

This can happen when a warning exists at the training revision and continues to exist at the testing revision. Because the original data-collection procedure includes warnings from both revisions, the same warning can appear in both datasets.

The chronological relationship is:

```text
Training revision          Testing revision          Reference revision
       |                          |                         |
       v                          v                         v
Warning W exists  ----------> Warning W exists
       |                          |
       +--------------------------+
          Same warning instance

The future reference revision is used to generate its label.
```

The warning may therefore have the same label in both datasets.

The classifier can encounter a test warning that it has effectively already seen during training.

This violates the principle that testing should measure how well a model generalizes to unseen examples.

### 4.4.2 How duplication was discovered

The authors progressively simplify the classifier and examine the resulting performance.

They find that a k-Nearest Neighbors (kNN) classifier performs surprisingly well, particularly when \(k=1\).

A kNN classifier predicts the label of a new example based on the labels of its nearest examples in the training dataset.

When \(k=1\), it considers only the single most similar training example.

The Golden Features kNN with \(k=1\) achieves:

- Precision: 0.87.
- Recall: 0.90.
- F1: 0.84.

This is surprisingly strong because relying on a single neighbor can make a classifier sensitive to noise and outliers. The result prompts the authors to investigate the training and testing datasets more closely.

They discover that the datasets contain many duplicated warnings.

The classifier's success can therefore be explained partly by its ability to identify warnings that are already represented in the training data.

### 4.4.3 A simple baseline exposes the problem

To investigate the contribution of duplication, the authors construct a deliberately simple classifier.

For each test warning, the classifier searches the training dataset for a warning with the same class name and FindBugs bug-pattern name.

If it finds a matching warning, it copies that warning's label.

If multiple matches exist, it selects one randomly. If no matching training warning exists, it predicts the majority class, which in this experiment is the false-alarm class.

The process is:

```text
Test warning
     |
     v
Search training data for matching class + bug pattern
     |
     v
Matching warning found?
     |
     +---- Yes ----> Copy its training label
     |
     +---- No -----> Predict the majority class
```

This simple baseline achieves an F1 score of 0.75.

It outperforms the Golden Features SVM after the leaked features have been removed.

The result demonstrates that duplicated warnings alone provide enough information for a simple method to achieve surprisingly strong performance.

The model does not need to learn a sophisticated generalizable relationship between warning characteristics and actionability. It can partly exploit the repeated instances.

## 4.5 Constructing a more realistic dataset

The authors address data duplication by changing how the testing dataset is constructed.

Instead of including every warning reported at the testing revision, they include only warnings introduced after the training revision and before the testing revision.

This ensures that the testing dataset contains newly introduced warnings rather than warnings already included in the training dataset.

The number of testing instances decreases from 15,695 to 2,615 after deduplication.

The revised dataset better reflects a realistic situation in which a classifier is trained on previously available warnings and evaluated on new warnings.

However, deduplication alone does not solve the leakage problem. Both issues must be addressed separately.

## 4.6 Eliminating leakage by reimplementing the features

The authors first evaluate the classifier after removing the five leaked features.

They then investigate whether similar features can be retained without using future information.

To do so, they reimplement the warning context and defect likelihood features using only information available at the relevant training or testing revision.

The revised calculation follows three principles:

1. Consider warnings introduced within the year before the relevant revision.
2. Determine whether those warnings are present or absent at that revision.
3. Do not use the future reference revision to calculate the features.

For the training revision, the feature calculation considers warnings introduced during the preceding year and uses the training revision to determine whether those warnings remain open.

For the testing revision, it considers warnings introduced after the training revision and within the year before testing. It uses the testing revision to determine whether those warnings remain open.

The reference revision is no longer used to calculate these features.

The information flow becomes:

```text
Historical warnings
        |
        v
Information available before training
        |
        v
Training features
        |
        v
Train the SVM
        |
        v
New warnings introduced after training
        |
        v
Features available at testing time
        |
        v
Predict actionable / false alarm
```

This approach preserves the general idea of using historical warning patterns while avoiding the use of future labels as model inputs.

The authors cannot perform this reimplementation for the Phoenix project because of difficulties building older project versions and insufficient historical revision data. Phoenix is therefore omitted from the subsequent experiments involving the reimplemented features.

## 4.7 Results after addressing both problems

The authors evaluate the Golden Features SVM under increasingly realistic conditions.

The following results are the average values reported for the original dataset:

| Technique | Precision | Recall | F1 |
|---|---:|---:|---:|
| Golden Features SVM | 0.84 | 0.94 | 0.88 |
| Golden Features SVM without leaked features | 0.26 | 0.70 | 0.38 |
| Golden Features SVM without duplicated data | 0.88 | 0.93 | 0.90 |
| Golden Features SVM without leakage and duplication | 0.27 | 0.57 | 0.31 |
| Reimplemented leaked features | 0.32 | 0.57 | 0.38 |
| Golden Features kNN, \(k=10\) | 0.91 | 0.57 | 0.68 |
| Golden Features kNN, \(k=5\) | 0.86 | 0.72 | 0.78 |
| Golden Features kNN, \(k=3\) | 0.87 | 0.78 | 0.82 |
| Golden Features kNN, \(k=1\) | 0.87 | 0.90 | 0.84 |
| SVM using only leaked features | 0.79 | 0.94 | 0.83 |
| Baseline copying a matching training label | 0.72 | 0.80 | 0.75 |

The results reveal several important points.

First, removing the leaked features causes a major decline in performance.

Second, the duplicated data helps explain why even simple classifiers achieve strong results.

Third, removing both problems substantially reduces the apparent effectiveness of the Golden Features.

After removing the leaked features and duplicated instances, the average F1 falls to 0.31. The average AUC also falls substantially, from approximately 1.00 in the original experiments to 0.59 in the more realistic setting.

The authors compare this result with a strawman baseline that predicts every warning is actionable. This baseline achieves an average F1 of approximately 0.52 in the more realistic experiment.

Consequently, the Golden Features SVM does not outperform the simple baseline in terms of F1.

Its AUC remains above 0.5, suggesting that the features retain some ability to distinguish between the classes. However, this limited predictive power does not justify the earlier conclusion that the task is almost perfectly solvable.

### Answer to RQ1

The Golden Features are not a silver bullet for actionable-warning detection.

Their previously reported near-perfect performance was substantially inflated by data leakage and data duplication. When the experiments are redesigned to avoid these problems, the Golden Features SVM underperforms a simple baseline in terms of F1, although its AUC indicates some remaining predictive ability.

The findings demonstrate the importance of realistic evaluation procedures and careful investigation of apparently strong experimental results.

# 5. Analysis of the Closed-Warning Heuristic

Section 4 establishes that experimental flaws inflated the apparent performance of the Golden Features. Section 5 investigates a different but closely related problem: whether the labels used to train and evaluate the classifier are reliable in the first place.

The closed-warning heuristic automatically assigns labels by comparing warnings at an earlier revision with a later reference revision.

Its central assumptions are:

- A warning that disappears is actionable.
- A warning that remains is a false alarm.

These assumptions are convenient, but they are not necessarily correct.

A warning may disappear because its underlying bug was fixed. However, the code might also have been changed for an unrelated reason.

Similarly, a warning that remains open may represent a genuine defect that developers have not inspected, have postponed, or consider too risky to fix.

The authors examine these limitations through three experiments.

## 5.1 Choosing a different reference revision

### 5.1.1 Why the reference interval matters

Previous studies typically used a reference revision two years after the testing revision.

The authors investigate what happens when the reference revision is moved further into the future.

They use reference revisions two, three, and four years after the testing revision.

The reasoning is straightforward: if developers sometimes delay fixing warnings, then warnings that remain open after two years may disappear by the third or fourth year.

According to the heuristic, these newly closed warnings become actionable.

Therefore, changing the reference interval may change the labels assigned to the same historical warnings.

If the resulting datasets have different class distributions, the measured performance of a classifier may also change.

### 5.1.2 Changes in actionability ratios

The authors find that the average proportion of actionable warnings increases from approximately 40% with a two-year interval to 54% with a four-year interval.

The actionability ratio increases by approximately 14 percentage points. The reported Wilcoxon signed-rank test finds this change statistically significant, with \(p=0.03\).

However, the changes vary substantially between projects.

For example, Derby has an actionable-warning ratio of 10% with a two-year interval, 59% with a three-year interval, and 66% with a four-year interval.

Other projects show different patterns, including projects in which the ratio remains relatively stable.

The results do not establish a universal relationship between reference-interval length and classifier quality. They show that the choice of interval can change the underlying dataset and, consequently, the conclusions drawn from an experiment.

### 5.1.3 Changes in classifier performance

The Golden Features SVM's average F1 increases from 0.39 with a two-year interval to 0.48 with a three-year interval and 0.57 with a four-year interval in the experiment reported in Table 5.

However, the strawman baseline also changes because the proportion of actionable warnings changes.

| Reference interval | Average actionable-warning ratio | Golden Features SVM F1 | Strawman F1 |
|---|---:|---:|---:|
| 2 years | 40% | 0.39 | 0.43 |
| 3 years | 42% | 0.48 | 0.55 |
| 4 years | 54% | 0.57 | 0.67 |

The SVM underperforms the strawman baseline across these average results.

Individual projects show considerable variation. For example, Derby's F1 increases from 0.06 at two years to 0.58 at three years and 0.72 at four years.

The authors also find that the AUC of some projects changes from below 0.5 to above 0.5 when the reference interval changes.

This means that the same classification approach can appear to perform differently depending on how the dataset is labeled.

### 5.1.4 Interpretation

The experiment shows that the reference revision is not a neutral implementation detail. It can change the ground-truth labels, the distribution of the classes, and the conclusions researchers draw about a classifier.

The central problem is not that one particular interval is necessarily wrong. Rather, the heuristic's labels are sensitive to an arbitrary choice about how far into the future researchers should look.

A warning that remains open after two years is not necessarily a false alarm. It might be fixed after three or four years.

Therefore, the reference interval must be considered when interpreting experimental results based on automatically generated labels.

## 5.2 Unconfirmed actionable warnings

### 5.2.1 Does a closed warning mean that a bug was fixed?

The closed-warning heuristic labels a warning actionable if it disappears in the reference revision while its file remains present.

However, disappearance does not prove that a developer intentionally fixed the problem identified by the warning.

Code changes can eliminate a warning incidentally. A feature may be redesigned, a method may be replaced, or the relevant code may be modified for reasons unrelated to the warning.

The authors therefore investigate whether warnings labeled actionable by the heuristic were actually fixed because of the warning.

### 5.2.2 Manual examination of closed warnings

The authors sample 1,357 warnings that the heuristic labeled actionable.

Two authors independently inspect the warnings and determine whether the code changes indicate that the warning was addressed.

They distinguish three outcomes:

- **Actionable:** The evidence suggests that the warning was removed because the underlying issue was fixed.
- **False alarm:** The original code or its documentation indicates that the warning did not represent a problem that needed fixing.
- **Unknown:** The code changed, but the evidence does not establish whether the warning was intentionally addressed.

When the annotators disagree, they discuss their decisions and reach a consensus.

The authors calculate Cohen's Kappa to assess inter-annotator agreement. They report a value of 0.83, indicating strong agreement beyond what would be expected by chance.

The results are:

| Human-assigned label | Number of warnings | Percentage |
|---|---:|---:|
| Actionable | 660 | 49% |
| False alarm | 176 | 13% |
| Unknown | 520 | 38% |
| Total | 1,356 | Approximately 100% |

The paper's accompanying discussion also describes the proportion of closed warnings considered actionable as approximately 47%. The detailed counts above give approximately 49%, so the important finding is that only about half of the examined warnings were confirmed actionable.

The remaining warnings were either judged false alarms or could not be reliably classified.

### 5.2.3 An example of incidental disappearance

The paper presents a FindBugs warning concerning the use of `new Long(conglomId)`, which creates a `Long` object using a constructor that the tool considers less efficient than `Long.valueOf`.

The original code contains:

```java
if (is_temporary)
{
    if (tempCongloms != null)
        tempCongloms.remove(new Long(conglomId));

    tempCongloms.put(new Long(conglomId), conglom);
}
```

In the later revision, the surrounding functionality changes. The code is modified to handle a different condition related to invalidating a conglomerate when an error occurs.

The warning disappears, but the change does not provide evidence that the developer intended to address the object-construction warning.

In this situation, the appropriate human-assigned label is unknown, rather than automatically actionable.

This example demonstrates that a warning can disappear because the code containing it has changed, not because the developer acted on the warning itself.

### 5.2.4 Interpretation

The closed-warning heuristic identifies a change in warning status, but it cannot reliably identify the reason for that change.

A closed warning might represent a successful bug fix, an intentional suppression, an unrelated code modification, or a change that makes the original warning irrelevant.

Consequently, the heuristic overestimates the number of actionable warnings when it assumes that every closed warning was intentionally fixed.

The results demonstrate that historical disappearance alone is insufficient evidence of actionability.

## 5.3 Unconfirmed false alarms

### 5.3.1 The problem with warnings that remain open

The preceding experiment examines warnings that disappear. The authors now investigate warnings that remain open.

The closed-warning heuristic labels an open warning a false alarm.

However, a warning may remain open because a developer has not yet inspected it, has postponed the work, or considers fixing it too risky or costly.

A warning's continued existence therefore does not establish that developers consider it unactionable.

The authors investigate this issue using developer-maintained FindBugs filter files.

### 5.3.2 Developer-maintained filter files

FindBugs filter files allow developers to suppress warnings that they have decided should not be reported.

Developers may add warnings to these files after inspecting them and determining that they are false alarms.

For projects that maintain such files, the authors assume that a developer who inspects a warning would either fix the problem or suppress the warning when it is not worth fixing.

Under this assumption, a warning that remains open and matches a developer-maintained filter file has evidence of being explicitly treated as a false alarm.

By contrast, an open warning that does not match a filter may simply not have been inspected.

The absence of a filter entry does not prove that the warning is actionable, but it means there is no equivalent evidence that developers explicitly classified it as a false alarm.

### 5.3.3 Project selection and results

The authors initially identify three projects from the earlier dataset that maintain FindBugs filter files: JMeter, Tomcat, and Commons-Lang.

They then search GitHub for additional mature projects that use FindBugs and maintain filter files. They exclude forks and projects with fewer than 100 stars or filter files containing fewer than 10 lines.

The resulting analysis covers 11 projects.

| Project | Open warnings | Filtered warnings | Percentage filtered |
|---|---:|---:|---:|
| JMeter | 710 | 6 | 1% |
| Tomcat | 1,624 | 9 | 1% |
| Commons-Lang | 106 | 19 | 18% |
| Flink | 4,934 | 4,754 | 96% |
| Hadoop | 3,053 | 269 | 9% |
| Jenkins | 1,212 | 178 | 15% |
| Kudu | 1,873 | 464 | 25% |
| Kafka | 4,668 | 2,993 | 64% |
| Morphia | 65 | 0 | 0% |
| Undertow | 347 | 113 | 33% |
| XMLGraphics-FOP | 949 | 909 | 96% |
| Average (mean) | 1,666 | 818 | 31% |
| Average (median) | 1,212 | 178 | 18% |

On average, 31% of open warnings match the projects' filter files, with a median of 18%. The proportions vary considerably across projects.

This indicates that only a minority of open warnings, on average, have been explicitly marked by developers as false alarms. Many open warnings have no such confirmation.

The authors emphasize that unfiltered warnings could still be false alarms. However, they could also be actionable warnings that developers have not inspected.

The heuristic therefore makes an unsupported leap when it treats every open warning as a false alarm.

### 5.3.4 Does cleaner data improve the classifier?

The authors next investigate whether removing warnings whose labels are not sufficiently confirmed affects the Golden Features SVM.

They examine two types of dataset cleaning:

1. Removing unconfirmed actionable warnings: warnings labeled actionable by the heuristic but not confirmed as actionable through human examination.
2. Removing unconfirmed false alarms: open warnings that have not been explicitly confirmed as false alarms by developer-maintained filter files.

For the first experiment, the authors retain warnings confirmed actionable by human annotators and sample open warnings to preserve a similar actionability ratio.

For the second experiment, they use projects with developer-maintained filter files and retain open warnings that match those files. They exclude projects in which fewer than 10% of open warnings match the filter, because the low proportion may indicate that the filter files are not maintained consistently.

The reported results are:

| Dataset | Actionable warnings (%) | SVM F1 | Strawman F1 | SVM AUC |
|---|---:|---:|---:|---:|
| Original dataset | 39.9 | 0.39 | 0.43 | 0.54 |
| Removing unconfirmed actionable warnings | 40.0 | 0.61 | 0.57 | 0.66 |
| Projects using FindBugs | 38.0 | 0.43 | 0.44 | 0.62 |
| Removing unconfirmed false alarms | 40.0 | 0.41 | 0.46 | 0.60 |

The accompanying narrative reports a similar improvement for the first cleaning experiment, describing F1 increasing from 0.39 to 0.64 and AUC from 0.54 to 0.68. The table reports 0.61 and 0.66, respectively. The precise values differ, but both presentations support the same qualitative conclusion.

Removing unconfirmed actionable warnings improves the classifier's performance enough to outperform the strawman baseline.

However, removing unconfirmed false alarms does not produce a comparable improvement.

These findings suggest that dataset quality affects classifier performance and that mislabeling warnings as actionable can be particularly harmful in the studied setting.

### 5.3.5 Interpretation

The experiment reveals two complementary weaknesses in the closed-warning heuristic.

A closed warning is not necessarily actionable because it may have disappeared incidentally.

An open warning is not necessarily a false alarm because it may not yet have been inspected or fixed.

The heuristic therefore confuses the observed status of a warning with the developer's judgment about whether the warning deserves action.

This distinction is fundamental. A warning's status in a future revision is evidence about what happened to the code, but it does not always reveal why it happened or whether the developer considered the warning actionable.

### Answer to RQ2

The closed-warning heuristic is useful for constructing large datasets, but it is not sufficiently reliable to serve as the sole source of ground-truth labels.

Its labels are sensitive to the choice of reference revision, and it can incorrectly classify warnings that disappear incidentally as actionable or warnings that remain open as false alarms.

The experiments indicate that more reliable labels can improve classification performance. A robust benchmark should combine scalable automatic labeling with human validation and evidence from developer actions.

# 6. Discussion

## 6.1 Lessons learned

The paper draws several broader lessons for empirical software-engineering research.

### The Golden Features are not a silver bullet

The Golden Features SVM does not achieve the near-perfect performance previously reported when data leakage and duplication are addressed.

Its performance is only marginally useful compared with a simple baseline in the more realistic experiment.

Further research is needed to develop features and techniques that genuinely generalize to new warnings.

### Strong results require qualitative investigation

A high evaluation score does not necessarily demonstrate that a model has learned a useful relationship.

The authors' use of LIME helps identify influential features, which leads them to discover data leakage. Their investigation of simpler classifiers exposes duplicated examples.

These findings show the importance of examining not only performance metrics but also the reasons behind those metrics.

### Simple baselines are essential

New techniques should be compared against straightforward baseline methods.

If a complex classifier cannot outperform a baseline that predicts every warning is actionable, its additional complexity may not provide practical value.

### Automatic labeling should be combined with human validation

The closed-warning heuristic enables researchers to construct large datasets efficiently, but its labels are not always reliable.

Manual labeling is more expensive and can introduce subjectivity, yet it can distinguish genuine fixes from incidental changes.

The authors therefore recommend combining heuristic-based labeling with manual validation. Developer commit messages and other historical evidence can help annotators determine why a warning disappeared.

They also encourage community efforts to build large, reliable benchmarks. As part of their study, they provide 1,300 labeled closed warnings as a starting point.

## 6.2 Threats to validity

The authors acknowledge several limitations.

- **Internal validity:** Their implementation could contain errors. They mitigate this risk by reusing existing datasets and code where possible.
- **Construct validity:** Evaluation metrics may not perfectly capture classifier effectiveness. They use metrics from previous studies and include F1 to account for the trade-off between Precision and Recall in an imbalanced dataset.
- **External validity:** The findings may not generalize to every project, language, or static-analysis tool. The study examines large, mature Java projects and focuses on FindBugs.
- **Human-label limitations:** Manual judgments and developer-maintained filter files can be imperfect. Multiple annotators independently label warnings, and the authors report strong inter-annotator agreement, with Cohen's Kappa greater than 0.8.
- **Project and feature selection:** Other projects, datasets, features, or machine-learning approaches might produce different results.

The authors do not claim that these limitations have been eliminated. Instead, they identify them and use methodological safeguards to reduce their impact.

# 7. Related Work

The paper relates to several broader research themes.

First, it joins studies that retrospectively evaluate apparently successful software-engineering techniques and identify limitations in their original experimental settings.

Second, it contributes to research on dataset quality, bias, duplication, and leakage. Previous studies in other software-engineering tasks have shown that duplicated examples or improperly constructed datasets can inflate performance estimates.

Third, it highlights the difficulty of automatically inferring ground-truth labels from software-development histories. Developers may act on warnings after substantial delays, so a warning's status at a particular point in time does not necessarily reveal whether it is actionable.

Finally, the paper distinguishes the duplication found in its dataset from other forms of duplication studied in software engineering. Here, the problem is that the same warning instance and its label can occur in both training and testing data, rather than merely having similar features across distinct examples.

# 8. Conclusion and Future Work

The paper demonstrates that detecting actionable warnings from Automatic Static Analysis Tools remains an open research problem.

Previously reported strong results for the Golden Features were substantially influenced by two experimental flaws: data leakage and data duplication.

Data leakage allowed the classifier's inputs to contain information derived from future warning labels. Data duplication allowed warnings in the testing dataset to overlap with those in the training dataset.

After addressing these problems, the Golden Features SVM performs considerably worse than earlier studies suggested and does not consistently outperform a simple baseline.

The authors also show that the closed-warning heuristic cannot always produce reliable ground-truth labels. A warning that disappears may not have been intentionally fixed, and a warning that remains open may still be actionable.

Future work should therefore focus on:

1. Developing more reliable benchmarks that combine automatic labeling with human validation.
2. Designing realistic experiments that prevent data leakage and training/testing duplication.
3. Comparing proposed techniques against appropriate simple baselines.
4. Investigating additional features and machine-learning approaches.
5. Building large, community-supported datasets with trustworthy labels.

The broader lesson is that impressive performance metrics must be interpreted in light of the experimental procedures and data used to produce them. Reliable research requires not only effective algorithms but also realistic evaluation and trustworthy ground truth.

**Replication package:** https://github.com/soarsmu/SA_retrospective