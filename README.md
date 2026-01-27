# OpenADMET ExpansionRx Challenge - Prediction Scripts

This repository describes the submission to the OpenADMET-ExpansionRx Blind Challenge for the `moka` user.

Author: Davide Boldini

Date: January 2026

## Overview

The modelling pipeline splits the challenge dataset into four endpoint categories, each targeting specific ADME properties. Each category is modelled independently, using multitasking only between tasks belonging to the same group.

All models were built using the `MATCHA` package from Merck Healthcare.

### Endpoint Categories

1. MPPB, MBPB, MGMB

2. HLM CLint, MLM CLint

3. LogD, KSOL

4. Caco-2 Permeability Papp A>B, Caco-2 Permeability Efflux

## Methodology

### Model Architectures

The final ensemble combines predictions from four complementary architectures:

- **GatedGCN** with a virtual node for enhanced global context (https://arxiv.org/abs/2003.00982)
- **GraphGPS** as the transformer-based architecture to provide a different inductive bias (https://arxiv.org/abs/2205.12454)
- **GIN** with laplacian and random-walk positional encodings (https://arxiv.org/pdf/1810.00826)
- **Chemprop** with RDKIT descriptor concatenation (https://pubs.acs.org/doi/10.1021/acs.jcim.5c02332)

### Pretraining strategies

Three pretraining datasets were used:

#### Large
Sparse multitask dataset encompassing regression and classification on 3D QM properties, RDKIT descriptors, ADME data and biochemical assays. Curriculum was used to adjust task importance during training. The general idea is inspired by this paper: https://arxiv.org/abs/2404.11568

#### ADME
Multitask regression-only dataset with ADME-related tasks.

### Descriptor-based (CheMeleon)
Multitask regression on molecular descriptors (https://arxiv.org/abs/2506.15792)

GatedGCN, GraphGPS and GIN models were pretrained using the first dataset, while Chemprop was pretrained using the second one. CheMeleon was used as-is from the Github repo.

Both ADME and Large had a mix of public and internal data.

### Finetuning strategies

Two finetuning strategies were used:

#### Type A
Simply finetune on the challenge endpoints, grouped as indicated above

#### Type B
Finetune on the grouped challenge endpoints, Jazzy descriptors and most significant MOE descriptors for those endpoints, identified via XGBoost + SHAP. Curriculum was used to adjust task importance during training.

#### Details

- **Optimizer**: AdamW with learning rate 1e-3, weight decay 1e-4,
- **Pretraining LR**: 1e-4 with model-specific decay schedules across the architecture, e.g. making earlier layers with lower LRs.
- **Epochs**: 200 maximum with early stopping (patience=20)
- **Batch Size**: 32
- **Loss Function**: Mean Absolute Error (MAE) in multitask framework, adjusting for sparser endpoints

Strategy A or B was selected depending on the endpoint and / or algorithm.

Appropriate transformations were applied per endpoint (log10p for most targets, none for LogD).

Each architecture trains an ensemble of 10 models. Each model per algorithm uses different random initializations, and uses a different slice of the training data, obtained via 10-fold CV. The validation set for each CV split is used for early stopping. The CV seeds are different for each algorithm to improve inter-model variability

**Final Prediction**: Simple averaging across all model predictions

### Model selection and performance evaluation

- **Splits**: I used a time-based sliding window splitting scheme, using the molecular IDs as a proxy for time. The procedure sorts the dataset according to the IDs, then e.g. uses the first 50% of the data as train, and the subset between 50% and 60% as test. Then, repeats training using 60% of the data, and testing using the 60% - 70% set and so forth.  The idea is to simulate model deployment and prospective inference, as that matches the train/test setting of the challenge. The procedure is repeated 5 times and performance metrics are averaged across splits
- **Consistency with leaderboard**: Generally speaking, performance improvements using the splitting scheme described above matched improvements on the leaderboard, but it wasn't always necessarily a match when performance improvements were minor
- **Hyperparameter tuning**: Final submission used default hyperparameters for pretraining and finetuning
- **Model seletion**: I tried different pretraining approaches (e.g. increase in model capacity, data curation, data sources, loss curriculum etc) and selected the best ones based on the performance on the sliding window performance

### Performance comments

- Pretrained models generally outperformed models trained from scratch for all endpoints
- For certain endpoints (e.g. CACO), the addition of auxiliary tasks like MOE descriptors or Jazzy was particularly useful, for others it was not important
- Descriptor or fingerprint-based methods were generally outperformed by graph-based methods, but the two approaches had usually low correlation in their residuals. Further work is needed to refine the ensembling strategy
- SMILES-based methods were generally outperformed by graph-based methods. Larger scale pretraining for this modality is needed
- HPO was attempted for the finetuning config, but it was generally quite noisy and did not improve on the leaderboard. Higher trial number likely needed, but I think the gains are likely to be small