# REGENIE vs PLINK: Side-by-Side Comparison

This document compares REGENIE and PLINK approaches for the same GWAS analysis on the 1000 Genomes EAS sample data.

---

## Quick Summary

| Aspect | PLINK (Module 06) | REGENIE (Module 32) |
|--------|-------------------|---------------------|
| **Method** | Logistic regression + PCs | Whole genome regression + PCs |
| **Structure correction** | Fixed effects (PCs) | Fixed effects (PCs) + Predictor (ĝ) |
| **Relatedness correction** | ❌ No | ✅ Yes (via ĝ) |
| **Speed** | ⚡ Very fast | ⚡ Fast (Step 1 overhead) |
| **Memory** | ~1 GB | ~5 GB |
| **Steps** | Single step | Two steps |
| **Best for** | Simple, unrelated samples | Large, potentially related samples |

---

## Model Comparison

### PLINK Model
```
y = μ + SNP*β + PC1*γ₁ + PC2*γ₂ + ... + PC10*γ₁₀ + ε

Where:
- SNP: Variant being tested (fixed effect)
- PC1-PC10: Principal components (fixed effects)
- ε: Residual error
```

**What it corrects for:**
- ✅ Major population structure (discrete ancestry)
- ❌ Cryptic relatedness
- ❌ Polygenic background beyond top PCs

### REGENIE Model
```
Step 1: Build ĝ = f(genome-wide SNPs)
        Using ridge regression on LD-pruned variants
        
Step 2: y = μ + SNP*β + PC1*γ₁ + ... + PC10*γ₁₀ + ĝ + ε

Where:
- SNP: Variant being tested (fixed effect)
- PC1-PC10: Principal components (fixed effects)  
- ĝ: Polygenic predictor (captures relatedness + structure)
- ε: Residual error
```

**What it corrects for:**
- ✅ Major population structure (via PCs)
- ✅ Cryptic relatedness (via ĝ)
- ✅ Polygenic background (via ĝ)
- ✅ Fine-scale structure not in top PCs

---

## Commands Side-by-Side

### PLINK Command
```bash
plink2 \
  --bfile ../04_Data_QC/sample_data.clean \
  --glm firth \
  --pheno ../01_Dataset/1kgeas_binary.txt \
  --pheno-name B1 \
  --covar ../05_PCA/plink_results_projected.sscore \
  --covar-name PC1_AVG-PC10_AVG \
  --out plink_B1_firth
```

**Runtime:** ~2-5 minutes  
**Memory:** ~1 GB  
**Output:** plink_B1_firth.B1.glm.firth

### REGENIE Commands
```bash
# Step 1: Build predictor (~5-10 min)
regenie \
  --step 1 \
  --bed ../04_Data_QC/sample_data.clean \
  --extract ../05_PCA/plink_results.prune.in \
  --phenoFile ../01_Dataset/1kgeas_binary_regenie.txt \
  --covarFile ../05_PCA/plink_results_projected.sscore \
  --covarColList PC1_AVG,...,PC10_AVG \
  --bt --bsize 1000 --lowmem \
  --lowmem-prefix tmpdir/regenie_tmp_preds \
  --out 1kg_eas_step1_BT

# Step 2: Test SNPs (~10-20 min)
regenie \
  --step 2 \
  --bed ../01_Dataset/1KG.EAS...missing \
  --ref-first \
  --phenoFile ../01_Dataset/1kgeas_binary_regenie.txt \
  --covarFile ../05_PCA/plink_results_projected.sscore \
  --covarColList PC1_AVG,...,PC10_AVG \
  --bt --bsize 400 \
  --firth --approx --pThresh 0.01 \
  --pred 1kg_eas_step1_BT_pred.list \
  --out 1kg_eas_step2_BT
```

**Total runtime:** ~15-30 minutes  
**Memory:** ~5 GB (with --lowmem)  
**Output:** 1kg_eas_step2_BT_B1.regenie

---

## Output Format Comparison

### PLINK Output
```
#CHROM  POS       ID         REF  ALT  A1   TEST  OBS_CT  BETA       SE         T_STAT     P
1       752566    rs3094315  G    A    A    ADD   504     -0.0891    0.145      -0.615     0.538
```

**Key columns:**
- CHROM, POS, ID: Variant identifiers
- REF/ALT: Reference and alternate alleles
- BETA: Log odds ratio
- SE: Standard error
- P: P-value

### REGENIE Output
```
CHROM  GENPOS   ID          ALLELE0  ALLELE1  A1FREQ  N    TEST  BETA      SE        CHISQ    LOG10P  EXTRA
1      752566   1:752566:G:A  G       A        0.385   504  ADD   -0.0892   0.146     0.373    0.267   NA
```

**Key columns:**
- CHROM, GENPOS, ID: Variant identifiers
- ALLELE0/ALLELE1: Reference/effect alleles
- A1FREQ: Effect allele frequency
- BETA: Log odds ratio
- SE: Standard error
- LOG10P: -log10(p-value) **Note: Not raw P!**
- EXTRA: Flags (e.g., "FIRTH" if Firth applied)

**Key difference:** REGENIE outputs LOG10P, PLINK outputs P directly

---

## Expected Results Comparison

For the 1000 Genomes EAS dataset (504 samples, simulated trait):

### Genomic Inflation (Lambda_GC)

| Method | Lambda_GC | Interpretation |
|--------|-----------|----------------|
| PLINK | ~1.02-1.05 | Good control (PCs work well for this homogeneous cohort) |
| REGENIE | ~1.00-1.02 | Excellent control (ĝ captures additional structure) |

**Why REGENIE is slightly lower:**
- ĝ captures more genetic background than PCs alone
- Better accounts for any hidden relatedness
- More conservative (fewer false positives)

### Top Associations

**Should be very similar:**
- Same loci reach genome-wide significance
- Effect sizes (BETA) within ~5-10% of each other
- P-values highly correlated (r > 0.95)

**Small differences:**
- REGENIE p-values slightly more conservative
- Rare variants may differ more (Firth implementation differences)
- REGENIE may detect slightly different secondary signals

### Example Top Hit

| Method | SNP | CHR | POS | BETA | SE | P-value |
|--------|-----|-----|-----|------|----|---------| 
| PLINK | rs123456 | 6 | 32500000 | 0.45 | 0.08 | 1.2e-8 |
| REGENIE | rs123456 | 6 | 32500000 | 0.44 | 0.08 | 1.5e-8 |

*Slightly more conservative p-value in REGENIE due to better background correction*

---

## When Results Differ Significantly

### Scenario 1: Hidden Relatedness

If your dataset has **cryptic relatedness** (cousins, duplicates):

**PLINK:**
- Lambda inflated (e.g., 1.15+)
- False positives
- Effect sizes biased

**REGENIE:**
- Lambda near 1.0
- Proper control
- Unbiased effects

**Solution:** REGENIE is necessary

### Scenario 2: Multi-Ethnic Cohorts

If your dataset has **discrete population structure** (EUR + AFR + EAS):

**PLINK with PCs:**
- PCs capture major axes well
- Lambda may be acceptable (~1.05)
- Top 10 PCs usually sufficient

**REGENIE:**
- ĝ + PCs capture both discrete and continuous structure
- Lambda closer to 1.0
- Better for downstream fine-mapping

**Solution:** Either works, REGENIE more robust

### Scenario 3: Homogeneous, Unrelated (This Tutorial)

If your dataset is **homogeneous and unrelated** (EAS only, 504 samples):

**PLINK with PCs:**
- Excellent control (lambda ~1.02)
- Fast and simple
- Sufficient for most purposes

**REGENIE:**
- Marginally better control (lambda ~1.01)
- Overkill for this dataset
- Good for learning the method

**Solution:** PLINK is sufficient, REGENIE is educational

---

## Computational Scaling

### Small Studies (<5K samples)

| Method | Runtime | Memory | Recommendation |
|--------|---------|--------|----------------|
| PLINK | Minutes | <1 GB | ✅ Best choice |
| REGENIE | 10-30 min | ~5 GB | ⚠️ Overkill |

**Verdict:** Use PLINK

### Medium Studies (5K-50K samples)

| Method | Runtime | Memory | Recommendation |
|--------|---------|--------|----------------|
| PLINK | 10-30 min | 1-5 GB | ✅ Good if unrelated |
| REGENIE | 30-120 min | 10-50 GB | ✅ Better if related |

**Verdict:** REGENIE if evidence of relatedness, otherwise PLINK

### Large Studies (>50K samples)

| Method | Runtime | Memory | Recommendation |
|--------|---------|--------|----------------|
| PLINK | 30-60 min | 5-10 GB | ✅ Fast but limited |
| REGENIE | 1-3 hours | 10-20 GB | ✅ **Best choice** |
| LMM (GEMMA) | Days-Weeks | 100GB-1TB | ❌ Impractical |

**Verdict:** REGENIE is the clear winner

---

## Pros and Cons

### PLINK Advantages

✅ **Speed:** Fastest method for association testing  
✅ **Simplicity:** One command, easy to understand  
✅ **Memory:** Minimal memory footprint  
✅ **Debugging:** Easy to troubleshoot  
✅ **Teaching:** Best for learning GWAS fundamentals  

❌ **No relatedness correction:** Fails with families/cousins  
❌ **Limited to top PCs:** May miss fine structure  
❌ **No polygenic adjustment:** Doesn't capture genome-wide background  

### REGENIE Advantages

✅ **Comprehensive correction:** Structure + relatedness + polygenic background  
✅ **Scalability:** Handles 500K+ samples efficiently  
✅ **No GRM needed:** Avoids n×n matrix computation  
✅ **LOCO:** Unbiased effect sizes  
✅ **Multiple traits:** Reuse Step 1 for many phenotypes  

❌ **Two-step workflow:** More complex  
❌ **Step 1 overhead:** Extra time for predictor  
❌ **More memory:** Requires more RAM than PLINK  
❌ **Learning curve:** Harder to understand initially  

---

## Decision Tree

```
Do you have evidence of relatedness?
│
├─ YES → Use REGENIE
│        (or traditional LMM if <50K samples)
│
└─ NO → Is sample size > 50K?
         │
         ├─ YES → Use REGENIE
         │        (speed advantage over LMM)
         │
         └─ NO → Is population homogeneous?
                  │
                  ├─ YES → Use PLINK + PCs
                  │        (sufficient control)
                  │
                  └─ NO → Use REGENIE
                           (handles complex structure better)
```

---

## Practical Recommendations

### For This Tutorial (504 EAS samples)

**Best choice: PLINK**
- Homogeneous population (single ancestry)
- No evidence of relatedness
- Small sample size (REGENIE overkill)
- Good for learning fundamentals

**Use REGENIE for:**
- Learning how whole genome regression works
- Practice before applying to larger datasets
- Understanding LOCO and polygenic prediction

### For Real-World Studies

**Use PLINK when:**
- <5K unrelated samples
- Clearly homogeneous population
- Speed is critical
- Exploratory analysis

**Use REGENIE when:**
- >50K samples (any population)
- Evidence of relatedness (kinship analysis shows >5% related)
- Multi-ethnic cohorts with hidden structure
- Multiple phenotypes (reuse Step 1)
- Publication-quality analysis requiring robust control

**Use traditional LMM (GEMMA/BOLT) when:**
- 5K-50K samples
- Family-based study
- Need explicit GRM for heritability estimation
- Moderate relatedness (10-30% related pairs)

---

## Converting Between Outputs

### REGENIE to PLINK format

```bash
# Convert LOG10P to P-value
awk 'BEGIN{OFS="\t"} 
     NR==1 {print $0, "P"} 
     NR>1 {print $0, 10^(-$12)}' \
     1kg_eas_step2_BT_B1.regenie > regenie_with_P.txt
```

### Extract matching variants

```bash
# Get common SNPs between PLINK and REGENIE results
awk 'NR>1 {print $1":"$2}' plink_results.glm.firth > plink_snps.txt
awk 'NR>1 {print $1":"$2}' regenie_results.regenie > regenie_snps.txt

comm -12 <(sort plink_snps.txt) <(sort regenie_snps.txt) > common_snps.txt
```

---

## Summary

| Use Case | PLINK | REGENIE |
|----------|-------|---------|
| **Teaching/Learning** | ✅ Best | ⚠️ Advanced |
| **Small unrelated cohorts** | ✅ Best | ⚠️ Overkill |
| **Large biobanks (>50K)** | ⚠️ Limited | ✅ Best |
| **Evidence of relatedness** | ❌ Fails | ✅ Best |
| **Multi-ethnic studies** | ⚠️ PCs may suffice | ✅ More robust |
| **Multiple phenotypes** | ⚠️ Repeat each time | ✅ Reuse Step 1 |
| **Speed priority** | ✅ Fastest | ⚠️ Step 1 overhead |
| **Memory constrained** | ✅ Minimal | ⚠️ Moderate |

**Bottom line:**
- **This tutorial dataset:** PLINK is sufficient, REGENIE is educational
- **Real biobanks:** REGENIE is the modern standard
- **Family studies:** REGENIE or traditional LMM

---

*Document created: 2026-07-31*
*Based on GWASTutorial Modules 06 (PLINK) and 32 (REGENIE)*
