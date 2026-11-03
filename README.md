# Mechanistic Interpretability of Tissue-Specific Gene Expression via MLM-Pretrained Transformers

### ⚠️ Note on Code Availability

> The original raw Python training scripts and data parsers for this project were hosted on the university faculty HPC cluster, Entropy, and were unfortunately lost during a server wipe in Summer 2026. However, to demonstrate the complete methodology, this repository contains the final project report, detailed architectural hyperparameters, and the original validation metrics. The training plots demonstrate successful Masked Language Modeling (MLM) pretraining and subsequent regression.

## Project Overview

This collaborative project investigated how Transformer models trained to predict tissue-specific gene expression from DNA sequences develop internal routing mechanisms driven by a tissue-context token. The genome routes identical DNA sequences through tissue-specific "logic" to produce different expression levels. To study this, our team developed a data pipeline to normalize raw Drosophila melanogaster RNA-seq counts and evaluated two architectures: a simple 4-layer flat Transformer (my contribution) and a hierarchical local-to-global Transformer (my teammate's contribution).   

## My Contributions

This project was conducted in a two-person team. I was exclusively responsible for the data preparation pipeline, the simple MLM-pretrained Transformer architecture, the training stability, and the causal activation patching analysis. (Note: The hierarchical model experiments detailed in the report were conducted by my teammate and are excluded from this summary).

My specific independent contributions include:

### 1. Data Preprocessing & Normalization (Real & Synthetic)

I developed the bioinformatics pipeline to translate raw RNA-seq reads into stable continuous targets for a regression head.

* **Real Biological Data Pipeline:** Processed raw Drosophila melanogaster (w1118) RNA-seq read counts mapped across 7 distinct tissues. The data was perfectly balanced (14.3% per tissue) and partitioned into strict 80/10/10 splits for training, validation, and testing.
* **Count Normalization & Variance Control:** Applied Median of Ratios normalization to scale data against a geometric pseudoreference, followed by a Variance Stabilizing Transformation (VST) to decouple variance from the mean and prevent massive count outliers from dominating the MSE loss.
* **Sequence Mapping:** Extracted 1000 nucleotides of the promoter region and the entire 5'UTR, capped at 2200 characters.
* **Synthetic Data Generation (Pipeline Validation):** To decouple model development from complex raw annotations, I constructed a deterministic synthetic dataset using an AT-rich grammar with a rigid causal motif trigger (GATTACAA). While later used extensively by my teammate to benchmark long-context aggregation, my primary intent for this synthetic data was to validate tensor shapes, verify gradient flow, and test the pipeline "plumbing" before introducing noisy biological data.

### 2. Architecture: MLM-Pretrained Transformer

I designed and trained the baseline model to establish a foundational understanding of the Drosophila genomic grammar.
* **Encoder Design:** Built a 4-layer Transformer encoder utilizing overlapping 6-mer tokenization with a stride of 1.
* **Contextual Routing:** Prepended a [TISSUE] token to the input sequence. The final predicted continuous expression value was derived entirely from a regression head attached to the tissue token's output state.

### 3. Training & Validation

* **Two-Stage Training:** Executed an initial Masked Language Modeling (MLM) pretraining phase (14 epochs, 8% masking rate) with cross-entropy loss to learn local motifs and DNA syntax, followed by 35 epochs of MSE-based regression fine-tuning. Gradient accumulation (effective batch size 256) was used to ensure stability.
* **Real-World Limitations & Training Duration:** The model achieved a final validation MSE of 0.42 after 35 epochs. Training was halted at this point acknowledging that predicting gene expression from strictly local DNA sequence (2,200 bp) is a highly partially observable problem which lacks critical biological features like 3D chromatin conformation and distant enhancer loops. Still, future work should ideally prolong training as much as the dataset allows to establish the model's true empirical performance ceiling.   
* **Tissue Routing Behavior:** Inference interference tests confirmed the model successfully increased its dependence on the [TISSUE] token over the 35 epochs. However, performance was uneven across tissues; the model established large prediction gaps primarily for highly specialized or physically isolated tissues (Head and Gonads) but struggled to establish clear separation boundaries for tissues with more blended borders (e.g., Abdomen vs. Digestive).
* **Synthetic Task Limitations:** On the rigidly defined synthetic dataset, the simple 6-mer flat Transformer degraded on long 2200-nucleotide sequences, recovering only 7.8% ± 9.0% of the target interaction (which motivated the development of the hierarchical model). It remains an open question to what extent this rigid synthetic causal rule accurately represents the natural noise distribution of real genomic sequences.


### 4. Mechanistic Interpretability

* **Activation Patching:** To identify structural elements responsible for predictions, I executed causal activation patching on the model. By swapping tissue tokens and patching internal clean activations into a corrupted forward pass, I calculated Restoration Scores to quantify causal significance.
* **Head Isolation:** For the successfully separated HEAD vs. DIGESTIVE tissue pair, the scan successfully isolated Layer 4, Head 1 (L4H1) as the primary attention head driving the prediction gap.


## Detailed Hyperparameters

The following parameters were utilized to configure the final pipeline and prevent overfitting:

| Category | Parameter | Value |
| --- | --- | --- |
| Sequence & Tokenization | Vocabulary | `[PAD]`, `[MASK]`, `N`, `A`, `C`, `T`, `G` |
| Sequence & Tokenization | k-mer size | 6 (stride: 1) |
| Sequence & Tokenization | Max sequence length | 2200 bp |
| Training (General) | Train/Val/Test Split | 0.8 / 0.1 / 0.1 |
| Training (General) | Batch size | 8 |
| Training (General) | Gradient accumulation steps | 32 |
| Training (General) | Warmup period | 5 epochs |
| MLM Pretraining | Masking rate | 8% |
| MLM Pretraining | Epochs | 14 |
| MLM Pretraining | Start Learning Rate | $1.00 \times 10^{-6}$ |
| MLM Pretraining | Max Learning Rate | $1.00 \times 10^{-4}$ |
| Regression Fine-Tuning | Epochs | 35 |
| Regression Fine-Tuning | Start Learning Rate | $1.00 \times 10^{-7}$ |
| Regression Fine-Tuning | Max Learning Rate | $5.00 \times 10^{-5}$ |
