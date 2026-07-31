# Hands-On REGENIE Tutorial

This guide provides step-by-step instructions to run REGENIE on the 1000 Genomes EAS sample data and compare results with PLINK.

---

## Learning Objectives

By the end of this tutorial, you will:
1. ✅ Install and configure REGENIE
2. ✅ Understand how REGENIE corrects for population structure and relatedness
3. ✅ Run REGENIE Step 1 (whole genome regression for polygenic prediction)
4. ✅ Run REGENIE Step 2 (single-SNP association testing)
5. ✅ Compare REGENIE vs PLINK results
6. ✅ Understand when to use REGENIE vs traditional LMM methods

---

## Prerequisites

Before starting, ensure you have completed:
- ✅ **Module 04**: Data QC (`04_Data_QC/sample_data.clean.*`)
- ✅ **Module 05**: PCA (`05_PCA/plink_results_projected.sscore`, `plink_results.prune.in`)
- ✅ **Module 06**: PLINK association tests (for comparison)

**Required files:**
```bash
04_Data_QC/sample_data.clean.{bed,bim,fam}    # QC'd genotypes
01_Dataset/1kgeas_binary_regenie.txt          # Phenotype file
05_PCA/plink_results_projected.sscore         # PC covariates
05_PCA/plink_results.prune.in                 # LD-pruned SNPs for Step 1
```

---

## Part 1: Installation

### Option A: Install Pre-compiled Binary (Recommended)

```bash
# Create tools directory if it doesn't exist
mkdir -p ~/tools/bin
cd ~/tools

# Download latest REGENIE release
wget https://github.com/rgcgithub/regenie/releases/download/v3.4.1/regenie_v3.4.1.gz_x86_64_Linux.zip

# Unzip
unzip regenie_v3.4.1.gz_x86_64_Linux.zip

# Make executable
chmod +x regenie_v3.4.1.gz_x86_64_Linux

# Create symlink
ln -s ~/tools/regenie_v3.4.1.gz_x86_64_Linux ~/tools/bin/regenie

# Add to PATH (add this to ~/.bashrc for persistence)
export PATH="${HOME}/tools/bin:${PATH}"

# Test installation
regenie --version
# Should output: REGENIE v3.4.1
```

### Option B: Install via Conda

```bash
# Create conda environment
conda create -n regenie_env -c conda-forge regenie

# Activate environment
conda activate regenie_env

# Test installation
regenie --version
```

### Option C: Install via Docker

```bash
# Pull REGENIE Docker image
docker pull ghcr.io/rgcgithub/regenie:v3.4.1

# Run via Docker (example)
docker run -v $(pwd):/data ghcr.io/rgcgithub/regenie:v3.4.1 \
  regenie --version
```

---

## Part 2: Understanding the Data

### Check your data files

```bash
cd /home/sagemaker-user/GWASTutorial/32_whole_genome_regression

# Check QC'd genotypes
wc -l ../04_Data_QC/sample_data.clean.fam
# Output: 504 samples

wc -l ../04_Data_QC/sample_data.clean.bim
# Output: ~1,086,027 SNPs (after QC)

# Check phenotype file
head -5 ../01_Dataset/1kgeas_binary_regenie.txt
# Should show: FID IID B1
#             HG00403 HG00403 1
#             ...

# Check PCA covariates
head -3 ../05_PCA/plink_results_projected.sscore
# Should show: #FID IID ALLELE_CT PC1_AVG PC2_AVG ... PC10_AVG

# Check LD-pruned SNPs for Step 1
wc -l ../05_PCA/plink_results.prune.in
# Output: ~190,000 independent SNPs
```

---

## Part 3: Step 1 - Build Polygenic Predictor

### What Step 1 Does

Step 1 builds a **whole genome regression model** to capture polygenic background:

1. ✅ Uses ~190K LD-pruned SNPs (from `plink_results.prune.in`)
2. ✅ Splits genome into blocks (~1000 SNPs per block)
3. ✅ Fits ridge regression within each block
4. ✅ Combines block predictions via stacked regression
5. ✅ Creates **LOCO predictions** (Leave-One-Chromosome-Out)
6. ✅ Outputs predictor file for Step 2

**This predictor (ĝ) captures:**
- Population structure (major ancestry differences)
- Cryptic relatedness (hidden family structure)
- Polygenic background (genome-wide effects)

### Fix Covariate File Header

REGENIE requires `FID` (not `#FID`) in header:

```bash
# Backup original file
cp ../05_PCA/plink_results_projected.sscore ../05_PCA/plink_results_projected.sscore.bak

# Remove # from header
sed -i 's/#FID/FID/' ../05_PCA/plink_results_projected.sscore

# Verify
head -1 ../05_PCA/plink_results_projected.sscore
# Should show: FID IID ALLELE_CT PC1_AVG PC2_AVG ...
```

### Run Step 1

```bash
# Navigate to module directory
cd /home/sagemaker-user/GWASTutorial/32_whole_genome_regression

# Create temporary directory for lowmem mode
mkdir -p tmpdir

# Define file paths
plinkFile=../04_Data_QC/sample_data.clean
phenoFile=../01_Dataset/1kgeas_binary_regenie.txt
covarFile=../05_PCA/plink_results_projected.sscore
covarList="PC1_AVG,PC2_AVG,PC3_AVG,PC4_AVG,PC5_AVG,PC6_AVG,PC7_AVG,PC8_AVG,PC9_AVG,PC10_AVG"
extract=../05_PCA/plink_results.prune.in

# Run REGENIE Step 1
regenie \
  --step 1 \
  --bed ${plinkFile} \
  --extract ${extract} \
  --phenoFile ${phenoFile} \
  --covarFile ${covarFile} \
  --covarColList ${covarList} \
  --bt \
  --bsize 1000 \
  --lowmem \
  --lowmem-prefix tmpdir/regenie_tmp_preds \
  --out 1kg_eas_step1_BT
```

### Understanding Step 1 Parameters

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `--step 1` | - | Indicates Step 1 (build predictor) |
| `--bed` | QC'd genotypes | Input genotype data in PLINK binary format |
| `--extract` | LD-pruned SNPs | Use only ~190K independent SNPs (faster, less LD) |
| `--phenoFile` | Binary phenotype | Trait to model (B1 = case/control) |
| `--covarFile` | PCA scores | Principal components (PC1-PC10) |
| `--covarColList` | PC1_AVG...PC10_AVG | Which PCs to include as fixed effects |
| `--bt` | - | Binary trait (uses logistic ridge regression) |
| `--bsize 1000` | - | 1000 SNPs per block for Level 0 regression |
| `--lowmem` | - | Memory-efficient mode (writes to disk) |
| `--lowmem-prefix` | tmpdir/... | Where to write temporary files |
| `--out` | 1kg_eas_step1_BT | Output prefix |

### Step 1 Output Files

After completion, you should see:

```bash
ls -lh 1kg_eas_step1_BT*

# Key output files:
# 1kg_eas_step1_BT.log           - Step 1 log file
# 1kg_eas_step1_BT_pred.list     - List of LOCO predictor files (use in Step 2)
# 1kg_eas_step1_BT_1.loco        - LOCO predictions for chromosome 1
# 1kg_eas_step1_BT_2.loco        - LOCO predictions for chromosome 2
# ... (one .loco file per chromosome)
```

### Examine Step 1 Log

```bash
# Check for successful completion
grep -A 5 "Elapsed time" 1kg_eas_step1_BT.log

# Check LOCO predictions created
grep "LOCO predictions" 1kg_eas_step1_BT.log

# Check ridge regression results
grep "ridge regression" 1kg_eas_step1_BT.log
```

### What's in the Predictor Files?

```bash
# Check predictor list
cat 1kg_eas_step1_BT_pred.list
# Output: paths to .loco files for each chromosome

# Peek at LOCO predictions (binary format)
# These contain the polygenic predictor (ĝ) for each individual
# Used in Step 2 to adjust for polygenic background
```

**Expected runtime:**
- ~5-10 minutes on 504 samples with 190K SNPs
- Time scales linearly with sample size
- Much faster than traditional LMM (no GRM computation!)

---

## Part 4: Step 2 - Association Testing

### What Step 2 Does

Step 2 performs **single-SNP association tests** using the predictor from Step 1:

1. ✅ Tests **all genome-wide SNPs** (~1M variants)
2. ✅ Uses LOCO predictions to adjust for polygenic background
3. ✅ When testing chr5 SNPs, uses predictor built from chr1-4,6-22
4. ✅ Applies **Firth correction** for variants with p < 0.01
5. ✅ Outputs association results with p-values, effect sizes

**Model being tested:**
```
y = SNP + PC1 + ... + PC10 + ĝ + ε
      ↑                       ↑
  SNP effect              Predictor from Step 1
  (what we test)         (polygenic background)
```

### Run Step 2

```bash
# Use original dataset (NOT LD-pruned) for testing all SNPs
plinkFile=../01_Dataset/1KG.EAS.auto.snp.norm.nodup.split.rare002.common015.missing
phenoFile=../01_Dataset/1kgeas_binary_regenie.txt
covarFile=../05_PCA/plink_results_projected.sscore
covarList="PC1_AVG,PC2_AVG,PC3_AVG,PC4_AVG,PC5_AVG,PC6_AVG,PC7_AVG,PC8_AVG,PC9_AVG,PC10_AVG"

# Run REGENIE Step 2
regenie \
  --step 2 \
  --bed ${plinkFile} \
  --ref-first \
  --phenoFile ${phenoFile} \
  --covarFile ${covarFile} \
  --covarColList ${covarList} \
  --bt \
  --bsize 400 \
  --firth --approx --pThresh 0.01 \
  --pred 1kg_eas_step1_BT_pred.list \
  --out 1kg_eas_step2_BT
```

### Understanding Step 2 Parameters

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `--step 2` | - | Indicates Step 2 (association testing) |
| `--bed` | Full dataset | Test ALL genome-wide SNPs (not just pruned) |
| `--ref-first` | - | First allele in BIM file is reference |
| `--phenoFile` | Binary phenotype | Same as Step 1 |
| `--covarFile` | PCA scores | Same PCs as Step 1 |
| `--bt` | - | Binary trait (logistic regression) |
| `--bsize 400` | - | Process 400 SNPs per batch (memory management) |
| `--firth` | - | Apply Firth correction for rare variants |
| `--approx` | - | Use approximate Firth (faster) |
| `--pThresh 0.01` | - | Apply Firth only if p < 0.01 (speed optimization) |
| `--pred` | Step 1 predictor | LOCO predictions from Step 1 |
| `--out` | 1kg_eas_step2_BT | Output prefix |

### Step 2 Output Files

```bash
ls -lh 1kg_eas_step2_BT*

# Key files:
# 1kg_eas_step2_BT.log           - Step 2 log file
# 1kg_eas_step2_BT_B1.regenie    - Association results for phenotype B1
```

### Examine Step 2 Results

```bash
# Check column names
head -1 1kg_eas_step2_BT_B1.regenie

# Output columns:
# CHROM GENPOS ID ALLELE0 ALLELE1 A1FREQ N TEST BETA SE CHISQ LOG10P EXTRA

# View top associations
sort -k11,11gr 1kg_eas_step2_BT_B1.regenie | head -10
# Sorts by LOG10P (column 11) in descending order

# Count genome-wide significant hits (p < 5e-8, -log10(p) > 7.3)
awk '$11 > 7.3' 1kg_eas_step2_BT_B1.regenie | wc -l

# Check for Firth-corrected variants
grep "FIRTH" 1kg_eas_step2_BT_B1.regenie | wc -l
```

### Understanding the Output Columns

| Column | Description |
|--------|-------------|
| CHROM | Chromosome number |
| GENPOS | Genomic position (bp) |
| ID | Variant ID (rsID or chr:pos) |
| ALLELE0 | Reference allele |
| ALLELE1 | Alternate allele (effect allele) |
| A1FREQ | Allele frequency of ALLELE1 |
| N | Sample size for this variant |
| TEST | Test type (ADD = additive model) |
| BETA | Effect size (log odds ratio for binary traits) |
| SE | Standard error of BETA |
| CHISQ | Chi-square test statistic |
| LOG10P | -log10(p-value) |
| EXTRA | Additional info (e.g., "FIRTH" if Firth correction applied) |

**Expected runtime:**
- ~10-20 minutes for ~1M SNPs on 504 samples
- Much faster than traditional LMM testing
- Firth correction adds minimal overhead (only for suggestive variants)

---

## Part 5: Compare REGENIE vs PLINK Results

### Run PLINK for Comparison (if not done already)

```bash
cd /home/sagemaker-user/GWASTutorial/06_Association_tests

# Run PLINK with Firth correction and PCs
plink2 \
  --bfile ../04_Data_QC/sample_data.clean \
  --glm firth \
  --pheno ../01_Dataset/1kgeas_binary.txt \
  --pheno-name B1 \
  --covar ../05_PCA/plink_results_projected.sscore \
  --covar-name PC1_AVG-PC10_AVG \
  --out plink_B1_firth

cd /home/sagemaker-user/GWASTutorial/32_whole_genome_regression
```

### Extract Matching Results

```bash
# PLINK results
plink_results="../06_Association_tests/plink_B1_firth.B1.glm.firth"

# REGENIE results
regenie_results="1kg_eas_step2_BT_B1.regenie"

# Extract key columns for comparison
awk 'NR==1 || NR>1 {print $1":"$3, $9, $12}' ${plink_results} > plink_for_comparison.txt
awk 'NR==1 || NR>1 {print $1":"$2, $9, -$11/log10(exp(1))}' ${regenie_results} > regenie_for_comparison.txt
# Note: REGENIE outputs LOG10P, converting to P for comparison
```

### Compare Top Hits

```bash
# Top 10 PLINK hits
echo "=== Top 10 PLINK Associations ==="
sort -k3,3g plink_for_comparison.txt | head -11

# Top 10 REGENIE hits
echo "=== Top 10 REGENIE Associations ==="
sort -k3,3g regenie_for_comparison.txt | head -11
```

### Compare P-value Distributions (QQ Plot)

You can visualize the comparison in R or Python. Here's a quick check:

```bash
# Count significant hits in each
echo "PLINK genome-wide significant (p < 5e-8):"
awk 'NR>1 && $3 < 5e-8' plink_for_comparison.txt | wc -l

echo "REGENIE genome-wide significant (p < 5e-8):"
awk 'NR>1 && $3 < 5e-8' regenie_for_comparison.txt | wc -l

# Calculate genomic inflation factor (lambda)
# (Would need R/Python for proper calculation)
```

### Expected Differences

**What you should observe:**

1. **Top hits mostly agree**
   - Same loci significant in both methods
   - Effect sizes (BETA) very similar
   
2. **REGENIE p-values slightly more conservative**
   - REGENIE's polygenic predictor (ĝ) captures more background variation
   - Better control for cryptic relatedness than PCs alone
   - Lower genomic inflation (lambda closer to 1.0)

3. **Rare variants differ more**
   - Firth correction works slightly differently
   - REGENIE's approximation may differ from PLINK's exact Firth

4. **Runtime differences**
   - PLINK: Faster (simpler model)
   - REGENIE: Step 1 overhead, but Step 2 is fast
   - For >50K samples, REGENIE becomes much faster overall

---

## Part 6: Visualize Results

### Create Manhattan Plot

Use the existing visualization notebook or create a simple one:

```bash
cd /home/sagemaker-user/GWASTutorial/32_whole_genome_regression

# If gwaslab is installed
python3 << 'EOF'
import gwaslab as gl
import pandas as pd

# Load REGENIE results
sumstats = gl.Sumstats(
    "1kg_eas_step2_BT_B1.regenie",
    fmt="regenie",
    snpid="ID",
    chrom="CHROM",
    pos="GENPOS",
    ea="ALLELE1",
    nea="ALLELE0",
    beta="BETA",
    se="SE",
    p="LOG10P",
    build="19"
)

# Manhattan plot
sumstats.plot_mqq(
    mode="m",
    save="manhattan_regenie.png",
    dpi=300
)

# QQ plot
sumstats.plot_mqq(
    mode="q",
    save="qq_regenie.png",
    dpi=300
)

print("Plots saved: manhattan_regenie.png, qq_regenie.png")
EOF
```

### Or use the existing notebook

```bash
# Launch Jupyter and open the visualization notebook
jupyter notebook Visualization_regenie.ipynb
```

---

## Part 7: Understanding What REGENIE Did

### How Population Structure Was Corrected

**PLINK approach (Module 06):**
```
y = SNP + PC1 + PC2 + ... + PC10 + ε

Only fixed effects (PCs)
```

**REGENIE approach (this module):**
```
y = SNP + PC1 + PC2 + ... + PC10 + ĝ + ε
                                    ↑
                    Polygenic predictor from Step 1
                    Captures: structure + relatedness + polygenic background
```

### The ĝ Predictor Captures

1. **Population structure** (like PCs)
   - EUR vs EAS differences
   - Within-EAS substructure
   - Continuous ancestry gradients

2. **Cryptic relatedness** (PCs don't capture this!)
   - Hidden cousins (3rd-4th degree relatives)
   - Family structure not obvious in PCA
   - Pairwise genetic similarity

3. **Polygenic background**
   - Genome-wide trait architecture
   - Many small effects across genome
   - Better than GRM random effect in traditional LMM

### How REGENIE Avoids Computing GRM

**Traditional LMM (GEMMA, BOLT-LMM):**
```
1. Build GRM: K = XX'/m
   - 504 × 504 matrix
   - Requires ~2 MB memory (small here, but 1TB for 500K samples!)
2. Eigendecomposition: K = UΛU'
   - Expensive: O(n³)
3. Test each SNP using eigen-decomposition
```

**REGENIE (this tutorial):**
```
Step 1: Ridge regression on SNP blocks
   - Never builds full GRM
   - Predictor ĝ implicitly captures genetic similarity
   - Memory: O(n) not O(n²)
   
Step 2: Test SNPs using ĝ as covariate
   - Simple regression, very fast
   - O(n) per SNP
```

**Result:** Same correction quality, 10-100× faster!

---

## Part 8: Advanced Exercises

### Exercise 1: Compare Standard vs LOCO

Try running Step 1 without LOCO (not recommended, just for learning):

```bash
# Step 1 without LOCO (for comparison only)
regenie \
  --step 1 \
  --bed ../04_Data_QC/sample_data.clean \
  --extract ../05_PCA/plink_results.prune.in \
  --phenoFile ../01_Dataset/1kgeas_binary_regenie.txt \
  --covarFile ../05_PCA/plink_results_projected.sscore \
  --covarColList ${covarList} \
  --bt \
  --bsize 1000 \
  --out 1kg_eas_step1_noLOCO
  # Note: Removed --lowmem flag (no LOCO by default)
```

**Question:** Why might including the test chromosome in the predictor inflate test statistics?

<details>
<summary>Answer</summary>

**Proximal contamination:** If a causal SNP on chr5 is included in the predictor when testing chr5 SNPs, the predictor will capture signal from the causal SNP itself. This reduces power to detect that SNP in Step 2 and biases effect size estimates toward zero.

LOCO prevents this by excluding chr5 when building the predictor used to test chr5 SNPs.
</details>

### Exercise 2: Test Different Block Sizes

```bash
# Try smaller blocks (more blocks, more computational time)
regenie \
  --step 1 \
  --bed ../04_Data_QC/sample_data.clean \
  --extract ../05_PCA/plink_results.prune.in \
  --phenoFile ../01_Dataset/1kgeas_binary_regenie.txt \
  --covarFile ../05_PCA/plink_results_projected.sscore \
  --covarColList ${covarList} \
  --bt \
  --bsize 500 \
  --lowmem \
  --lowmem-prefix tmpdir/regenie_tmp_preds_500 \
  --out 1kg_eas_step1_bsize500

# Compare runtime and results
```

**Question:** How does block size affect runtime and prediction accuracy?

<details>
<summary>Answer</summary>

**Smaller blocks (--bsize 500):**
- ✅ More granular modeling of local LD
- ✅ May capture finer genetic structure
- ❌ Slower (more blocks to process)
- ❌ More predictors to combine in Level 1

**Larger blocks (--bsize 2000):**
- ✅ Faster
- ✅ Fewer predictors to combine
- ❌ Less granular LD modeling
- ❌ May miss local structure

**Default 1000 is usually optimal** for most applications.
</details>

### Exercise 3: Run Without PCs

Try REGENIE without PCs to see if ĝ alone is sufficient:

```bash
# Step 1 without PCs
regenie \
  --step 1 \
  --bed ../04_Data_QC/sample_data.clean \
  --extract ../05_PCA/plink_results.prune.in \
  --phenoFile ../01_Dataset/1kgeas_binary_regenie.txt \
  --bt \
  --bsize 1000 \
  --lowmem \
  --lowmem-prefix tmpdir/regenie_tmp_preds_noPCs \
  --out 1kg_eas_step1_noPCs

# Step 2 without PCs
regenie \
  --step 2 \
  --bed ../01_Dataset/1KG.EAS.auto.snp.norm.nodup.split.rare002.common015.missing \
  --ref-first \
  --phenoFile ../01_Dataset/1kgeas_binary_regenie.txt \
  --bt \
  --bsize 400 \
  --firth --approx --pThresh 0.01 \
  --pred 1kg_eas_step1_noPCs_pred.list \
  --out 1kg_eas_step2_noPCs

# Compare lambda_GC
```

**Question:** For this EAS-only cohort, are PCs necessary when using REGENIE?

<details>
<summary>Answer</summary>

For this **homogeneous EAS cohort**, the polygenic predictor ĝ likely captures most structure, so PCs may be redundant. However:

- ✅ **Including PCs is still recommended** as "belt and suspenders" approach
- ✅ PCs are cheap to compute and include
- ✅ Explicit PCs aid interpretability (which PC drives the structure?)
- ✅ For **multi-ethnic cohorts**, PCs are essential to capture discrete ancestry differences

If you observe similar lambda_GC with and without PCs, ĝ is doing the heavy lifting!
</details>

---

## Part 9: Troubleshooting

### Common Errors

#### Error: "FID column not found"

```bash
# Fix: Remove # from header
sed -i 's/#FID/FID/' ../05_PCA/plink_results_projected.sscore
```

#### Error: "Phenotype file format"

```bash
# Check phenotype file has header: FID IID PHENO
head -3 ../01_Dataset/1kgeas_binary_regenie.txt

# Binary traits should be coded as 1/2 (not 0/1)
# NA for missing
```

#### Error: "Could not open prediction file"

```bash
# Check Step 1 completed successfully
ls -lh 1kg_eas_step1_BT_pred.list

# Check paths in pred.list are correct
cat 1kg_eas_step1_BT_pred.list
```

#### Error: "Firth correction failed"

This can happen for variants with very low MAF. The `--approx` flag usually prevents this, but if it persists:

```bash
# Option 1: Increase pThresh (apply Firth less often)
--pThresh 0.001

# Option 2: Skip Firth entirely
# (Remove --firth --approx --pThresh flags)
```

### Check Log Files

Always examine the log files for errors:

```bash
# Step 1 log
tail -50 1kg_eas_step1_BT.log

# Step 2 log
tail -50 1kg_eas_step2_BT.log

# Look for ERROR or WARNING
grep -i "error\|warning" 1kg_eas_step1_BT.log
```

---

## Part 10: Summary & Key Takeaways

### What You Learned

1. ✅ **REGENIE's two-step approach**
   - Step 1: Build polygenic predictor (ĝ) via whole genome regression
   - Step 2: Test SNPs using ĝ to adjust for background

2. ✅ **How REGENIE corrects for structure/relatedness**
   - Ridge regression on SNP blocks captures genetic similarity
   - ĝ predictor replaces GRM random effect (but no explicit GRM computed!)
   - LOCO prevents proximal contamination

3. ✅ **When to use REGENIE**
   - ✅ Large samples (>50K) - huge speed advantage
   - ✅ Cryptic relatedness (cousins, hidden families)
   - ✅ Multiple phenotypes (reuse Step 1 predictions)
   - ⚠️ Overkill for small (<5K), unrelated, homogeneous cohorts

4. ✅ **REGENIE vs traditional methods**
   - PLINK (PCs only): Fast, simple, but misses relatedness
   - LMM (GEMMA/BOLT): Gold standard, but slow and memory-intensive
   - REGENIE: Best of both worlds - LMM-quality correction, PLINK-like speed

### Comparison Table

| Method | Structure | Relatedness | Speed (500K samples) | Memory |
|--------|-----------|-------------|----------------------|--------|
| PLINK + PCs | ✅ Good | ❌ No | ⚡ Very fast | ~1 GB |
| GEMMA (LMM) | ✅ Excellent | ✅ Excellent | 🐌 Days | ~1 TB |
| BOLT-LMM | ✅ Excellent | ✅ Excellent | 🐢 Slow | ~500 GB |
| REGENIE | ✅ Excellent | ✅ Excellent | ⚡ Hours | ~10 GB |

### When to Use Each

**Use PLINK when:**
- Small sample size (<5K)
- Clearly unrelated individuals (kinship filtering applied)
- Homogeneous population
- Speed is critical
- Teaching fundamentals

**Use REGENIE when:**
- Large sample size (>50K)
- Evidence of relatedness
- Multiple phenotypes to test
- Need robust control without GRM overhead

**Use traditional LMM (GEMMA/BOLT) when:**
- Moderate sample size (5K-50K)
- Family-based designs
- When you need explicit GRM for downstream analyses
- Research setting (publication standards)

---

## Part 11: Next Steps

### Further Exploration

1. **Compare with LMM methods** (Module 33)
   - Run GEMMA or BOLT-LMM on the same data
   - Compare runtime, memory, and results

2. **Gene-based testing** with REGENIE
   - Use `--anno-file` and `--set-list` for burden/SKAT tests
   - See: https://rgcgithub.github.io/regenie/options/#gene-based-testing

3. **Interaction testing** (G×E)
   - Test for SNP × covariate interactions
   - Example: SNP × sex, SNP × BMI

4. **Multiple phenotypes**
   - Add more traits to phenotype file
   - Reuse Step 1 predictions for all traits (very efficient!)

### Additional Resources

- **REGENIE Documentation:** https://rgcgithub.github.io/regenie/
- **REGENIE GitHub:** https://github.com/rgcgithub/regenie
- **REGENIE Paper:** Mbatchou et al. (2021) Nature Genetics
- **Tutorial Datasets:** https://rgcgithub.github.io/regenie/recommendations/

---

## Appendix: Quick Reference Commands

### Complete Workflow (Copy-Paste Ready)

```bash
#!/bin/bash
# Complete REGENIE workflow for GWASTutorial

cd /home/sagemaker-user/GWASTutorial/32_whole_genome_regression

# Fix covariate header
sed -i 's/#FID/FID/' ../05_PCA/plink_results_projected.sscore

# Create temp directory
mkdir -p tmpdir

# Define variables
plinkFile_clean=../04_Data_QC/sample_data.clean
plinkFile_full=../01_Dataset/1KG.EAS.auto.snp.norm.nodup.split.rare002.common015.missing
phenoFile=../01_Dataset/1kgeas_binary_regenie.txt
covarFile=../05_PCA/plink_results_projected.sscore
covarList="PC1_AVG,PC2_AVG,PC3_AVG,PC4_AVG,PC5_AVG,PC6_AVG,PC7_AVG,PC8_AVG,PC9_AVG,PC10_AVG"
extract=../05_PCA/plink_results.prune.in

# Step 1: Build polygenic predictor
echo "Running REGENIE Step 1..."
regenie \
  --step 1 \
  --bed ${plinkFile_clean} \
  --extract ${extract} \
  --phenoFile ${phenoFile} \
  --covarFile ${covarFile} \
  --covarColList ${covarList} \
  --bt \
  --bsize 1000 \
  --lowmem \
  --lowmem-prefix tmpdir/regenie_tmp_preds \
  --out 1kg_eas_step1_BT

# Step 2: Association testing
echo "Running REGENIE Step 2..."
regenie \
  --step 2 \
  --bed ${plinkFile_full} \
  --ref-first \
  --phenoFile ${phenoFile} \
  --covarFile ${covarFile} \
  --covarColList ${covarList} \
  --bt \
  --bsize 400 \
  --firth --approx --pThresh 0.01 \
  --pred 1kg_eas_step1_BT_pred.list \
  --out 1kg_eas_step2_BT

# Check results
echo "Top 10 associations:"
sort -k11,11gr 1kg_eas_step2_BT_B1.regenie | head -11

echo "Genome-wide significant hits (p < 5e-8):"
awk '$11 > 7.3' 1kg_eas_step2_BT_B1.regenie | wc -l

echo "Done! Check 1kg_eas_step2_BT_B1.regenie for full results."
```

---

## Questions & Answers

### Q1: Why use LD-pruned SNPs in Step 1 but all SNPs in Step 2?

**A:** Step 1 builds a polygenic predictor to capture background effects. Using LD-pruned SNPs:
- Reduces computational burden (fewer SNPs to process)
- Avoids redundancy (correlated SNPs don't add much information)
- Captures genome-wide signal efficiently

Step 2 tests for associations, so we want to test **all variants** including those in LD with causal variants.

### Q2: Do I always need to include PCs with REGENIE?

**A:** Not always, but usually recommended:
- **Homogeneous populations (e.g., EAS-only):** ĝ alone may be sufficient
- **Multi-ethnic cohorts:** PCs are essential for discrete ancestry differences
- **Best practice:** Include PCs as "belt and suspenders" - they're cheap and improve interpretability

### Q3: How is REGENIE's ĝ different from GRM?

**A:** Conceptually similar, computationally different:
- **GRM:** Explicitly computes n×n matrix of pairwise genetic similarity
- **ĝ:** Implicitly captures similarity via ridge regression predictions
- Both achieve the same goal: adjust for genetic background
- REGENIE never builds the n×n matrix (saves memory, faster)

### Q4: When should I use LOCO?

**A:** Almost always! LOCO prevents proximal contamination:
- Without LOCO: When testing chr5, predictor includes chr5 → absorbs signal from causal variants on chr5 → reduced power
- With LOCO: When testing chr5, predictor excludes chr5 → unbiased effect sizes

Only skip LOCO if you're in a hurry and don't care about precise effect size estimates.

### Q5: Can I use REGENIE for quantitative traits?

**A:** Yes! Remove the `--bt` flag and REGENIE will automatically use linear regression instead of logistic:

```bash
regenie \
  --step 1 \
  --bed ${plinkFile} \
  ... (other parameters)
  # No --bt flag for quantitative traits
  --out quant_step1
```

---

**Congratulations!** You've completed the hands-on REGENIE tutorial. You now understand how whole genome regression works and when to use it for GWAS analysis.

**Next modules:**
- Module 33: Linear Mixed Models (comparison with GEMMA/BOLT-LMM)
- Module 35: Saddlepoint Approximation (SAIGE for extreme case-control imbalance)

---

*Document created: 2026-07-31*
*Based on GWASTutorial Module 32: Whole Genome Regression by REGENIE*
