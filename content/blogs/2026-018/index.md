---
post_id: "2026-018"
title: "Benchmarking context-aware seq2func models against fine-tuned sequence-only models"

# Optional: image filename "your-image.png" in the same folder

# Optional: a PDF version of the post, e.g. the paper or preprint it reports,
# in the same folder and under 5 MB. Readers get View PDF and Download PDF
# buttons, and Google Scholar and Zenodo get this PDF instead of a printout of
# the page. Its first page should show the same title and authors as below.
# pdf: "paper.pdf"

# Optional: Enable KaTeX for inline/block math (e.g. $10^{-K}$)
math: true

# Author(s): list of names (used for /authors/<slug>/)
authors: ["Toby Clark", "Nathan Skene"]

# Optional: full details for citation, display and JSON-LD
authors_display:
  - name: " Toby Clark"
    affiliation: "UK Dementia Research Institute at Imperial, Imperial College London"
    orcid: "0009-0006-7119-4094"
  - name: "Nathan Skene"
    affiliation: "UK Dementia Research Institute at Imperial"
    orcid: "0000-0002-6807-3180"

editor: "Editor Name"

# Set automatically by the submission form to the GitHub login of the original
# submitter. Used by the form to surface "Update one of my previous posts" for
# the same account on later revisions. Safe to omit when authoring manually.
# submitter_github: "your-github-login"

# Add any number of tags. They're searchable on the blog homepage. See https://genomicsxai.github.io/tags/ for examples.
tags: ["genomics", "Seq2func", "Context-Seq2func", "Fine-Tuning", "Benchmarking", "Gene-Regulation", "Chromatin-Accessibility", "Alphagenome", "Alphagenome-2", "Cerberus", "Bolero", "ChromBPNet"]
# Category determines which homepage pill filter the post appears under.
# Supported values: "Announcement", "Blog Post", "Tutorial", "Perspective", "Paper Reviews"
#   - "Announcement"  → appears under the Announcements pill (editorial/community announcements)
#   - "Blog Post"     → appears under the Blogs pill (default for most posts)
#   - "Tutorial"      → appears under the Tutorials pill (step-by-step technical guides)
#   - "Perspective"   → appears under the Perspectives pill (opinion pieces, commentary)
#   - "Paper Reviews" → appears under the Paper Reviews pill (summaries/critiques of a published paper)
# Note: the homepage pills filter by `categories` only, not by `scope`.
categories: ["Blog Post"]

# One or more: protocols, tutorials, negative-results, discussions, insights, ideas
scope: ["insights"]
# One or more: technical, general, new-to-field
audience: ["technical"]
labs: ["Skene Lab"]

status: "submitted"
revision: 1

date_submitted: 2026-02-19
date_accepted: 
date: 2026-02-19

doi: ""
zenodo_url: ""
revision_history:
  - version: 1
    date: 2026-02-19
    notes: "Initial submission"
    # Optional: version-specific DOI / Zenodo record link
    doi: ""
    zenodo_url: ""
---

{{< summary >}}

Include a high-level summary of your post here. Alternatively editors can write a summary of the post if requested.

{{< /summary >}}

---

## Introduction

Sequence-to-function (seq2func) models predict molecular readouts, such as gene expression, transcription factor binding and chromatin accessibility, from DNA sequence. Traditional seq2func models, of which [AlphaGenome](https://www.nature.com/articles/s41586-025-10014-0)[1] is the current generalist state-of-the-art, predict a predefined set of outputs from sequence alone, enabling prediction of many cell type-specific measurements in parallel. AlphaGenome uses transformer attention within a U-Net architecture to integrate information across distant positions in the DNA sequence, but a recent model, [Cerberus](https://www.biorxiv.org/content/10.64898/2026.10.02.756377v1)[2], instead uses bidirectional state-space blocks built upon Mamba-2, which scale more effectively with increasing sequence length than full attention. Sequence-only models are restricted to making predictions only in tracks they see in training (without further fine-tuning), with cellular context being implicit in each output track. 

Context-aware seq2func models instead condition their sequence predictions on molecular measurements, for example gene expression profiles, which describe a particular cell type or state. This means that cellular context is supplied as an input rather than fixed by the choice of output track. With the recent release of [Bolero](https://www.biorxiv.org/content/10.64898/2026.10.19.760026v1)[3] and [AlphaGenome 2](https://storage.googleapis.com/deepmind-media/DeepMind.com/Blog/alphagenome-2-deciphering-how-cells-read-dna-to-better-understand-disease/alphagenome-2.pdf)[4], Google DeepMind's context-aware successor to AlphaGenome, these models have reached a scale comparable to that of sequence-only models. Although both classes of seq2func model are trained to predict molecular profiles, a particularly useful application is to predict the effects of genetic variants by comparing predictions for a variant's reference and alternate alleles. This is valuable because models can learn from molecular profiles mapped to the reference genome, whereas experimentally profiling the effects of every possible variant is not feasible at genome scale. Making *in silico* variant effect predictions can still require substantial GPU time, meaning that it is important to know for a given task, which model and scoring approach is likely to give the most accurate predictions.

Recent seq2func models have often been evaluated on their ability to predict the effects of expression quantitative trait loci (eQTLs), which are variants associated with differences in gene expression. Chromatin accessibility QTLs (caQTLs) offer a complementary benchmark, and are potentially easier predict from sequence as they are more driven by the local surrounding sequence rather than by distal enhancers, as shown by [Tu et al., 2026](https://www.biorxiv.org/content/10.64898/2026.02.01.702969v1)[5]. Accessibility is also a less direct readout of the conditioning data for context-aware models, which help describe the relevant cell state but do not measure accessibility itself. We therefore evaluate models on a microglial caQTL dataset from [Kosoy et al., 2022](https://www.nature.com/articles/s41588-022-01149-1)[6], previously used to evaluate [ChromBPNet](https://www.biorxiv.org/content/10.1101/2024.12.25.630221v4)[7], AlphaGenome and AlphaGenome 2. Furthermore, as AlphaGenome lacks a dedicated microglial chromatin accessibility track, we can ask how fine-tuning a sequence-only model to a new cellular context compares with conditioning a model on the same data at inference time. By running this benchmark on AlphaGenome 2, AlphaGenome, Bolero and Cerberus, as well as ChromBPNet to give a short-context baseline, we aim to find out the best way to make variant effect predictions for a dataset of interest, and explore how this is affected by test-time augmentation (TTA), distillation and ensembling.

## The benchmark

The microglial caQTL dataset on which we test our model is the filtered benchmark subset used by previously mentioned papers, consisting of 877 unique lead caQTL variants which overlap open chromatin peaks in microglia.

For each variant, we scored models as follows:
 1. Predefine a set of 100 shifts for TTA, chosen with the same seed for each model within a $\pm$ 2048 bp window for the long-context models and $\pm$ 100 bp for ChromBPNet
 2. Build a two sequence windows of the model's standard context length (2,114 bp for ChromBPNet, 524 kb for Bolero, 786 kb for Cerberus, 1 Mb for AlphaGenome and AlphaGenome 2), centred on the variant, one with the variant's reference allele and the other with the alternate allele.
 3. Build sequence windows with each shift applied for each model, and additionally for the reverse complement of each window with each shift.
 4. Score each constructed window using the recommended scorer for each model - for long-context models this is the $\log_2$ fold change of the summed alternate allele sequence relative to the reference allele sequence, and for ChromBPNet it is the $log_2$ fold change in predicted counts for alternate relative to the reference allele.
 


![Alt text](filename.png "width=400")

## Section two


## References

1. Avsec, Ž. et al. Advancing regulatory variant effect prediction with AlphaGenome. *Nature*, 649, 1206–1218 (2026). https://doi.org/10.1038/s41586-025-10014-0
2. Kelley, D.R., Yuan, H., Huang, X., Linder, J. Cerberus: bidirectional state space blocks improve accuracy and efficiency of regulatory sequence models. *bioRxiv* (2026). https://doi.org/10.64898/2026.10.02.756377
3. Liu, H. et al. Predicting cell-state-specific gene regulation from DNA across species. *bioRxiv* (2026). https://doi.org/10.64898/2026.10.19.760026
4. Gayoso A., Mottram, A., Karollus A. et al. AlphaGenome 2 predicts regulatory activity and variant effects across cell states. *Google DeepMind preprint* (2026). https://storage.googleapis.com/deepmind-media/DeepMind.com/Blog/alphagenome-2-deciphering-how-cells-read-dna-to-better-understand-disease/alphagenome-2.pdf
5. Tu, X. et al. A modality gap in personal-genome prediction by sequence-to-function models. *bioRxiv* (2026). https://doi.org/10.64898/2026.02.01.702969
6. Kosoy, R. et al. Genetics of the human microglia regulome refines Alzheimer's disease risk loci. *Nat. Genet.* **54**, 1145-1154 (2022). https://doi.org/10.1038/s41588-022-01149-1
7. Pampari, A. et al. ChromBPNet: bias factorized, base-resolution deep learning models of chromatin accessibility reveal cis-regulatory sequence syntax, transcription factor footprints and regulatory variants. *bioRxiv* (2024). https://doi.org/10.1101/2024.12.25.630221


```