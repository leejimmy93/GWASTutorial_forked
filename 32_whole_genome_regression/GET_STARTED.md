# Getting Started with REGENIE Tutorial

Welcome to the hands-on REGENIE tutorial! This guide will help you learn whole genome regression for GWAS.

---

## 📚 Tutorial Materials

Your learning path through REGENIE:

### 1. **HANDS_ON_GUIDE.md** (START HERE!)
👉 **[Open HANDS_ON_GUIDE.md](HANDS_ON_GUIDE.md)**

**What's inside:**
- Complete installation guide (3 methods)
- Step-by-step walkthrough with explanations
- All commands copy-paste ready
- Troubleshooting section
- Practice exercises
- ~30-45 minutes

**You'll learn:**
- ✅ How to install REGENIE
- ✅ What Step 1 does (build polygenic predictor)
- ✅ What Step 2 does (test SNPs)
- ✅ How REGENIE corrects for structure and relatedness
- ✅ When to use REGENIE vs PLINK

### 2. **REGENIE_vs_PLINK_COMPARISON.md** (For Context)
👉 **[Open REGENIE_vs_PLINK_COMPARISON.md](REGENIE_vs_PLINK_COMPARISON.md)**

**What's inside:**
- Side-by-side command comparison
- Model differences explained
- Decision trees
- When to use which method
- Performance benchmarks

**Read this to understand:**
- ✅ Why REGENIE exists
- ✅ When PLINK is sufficient
- ✅ When you need REGENIE
- ✅ Trade-offs between methods

### 3. **README.md** (Conceptual Background)
👉 **[Open README.md](README.md)**

**What's inside:**
- Theoretical explanations
- Algorithm details
- Ridge regression concepts
- LOCO methodology
- Advanced features

**Read this for:**
- ✅ Deep understanding of the method
- ✅ Mathematical details
- ✅ Publication-level knowledge

---

## 🚀 Quick Start (5 minutes)

### Prerequisites Check

```bash
cd /home/sagemaker-user/GWASTutorial/32_whole_genome_regression

# Check required files exist
ls ../04_Data_QC/sample_data.clean.bed  # QC'd genotypes
ls ../05_PCA/plink_results.prune.in     # LD-pruned SNPs
ls ../01_Dataset/1kgeas_binary_regenie.txt  # Phenotype
```

If all files exist, you're ready!

### Install REGENIE (Choose One)

**Option A: Pre-compiled binary** (Fastest)
```bash
mkdir -p ~/tools/bin
cd ~/tools
wget https://github.com/rgcgithub/regenie/releases/download/v3.4.1/regenie_v3.4.1.gz_x86_64_Linux.zip
unzip regenie_v3.4.1.gz_x86_64_Linux.zip
chmod +x regenie_v3.4.1.gz_x86_64_Linux
ln -s ~/tools/regenie_v3.4.1.gz_x86_64_Linux ~/tools/bin/regenie
export PATH="${HOME}/tools/bin:${PATH}"
regenie --version
```

**Option B: Conda** (Easier environment management)
```bash
conda create -n regenie_env -c conda-forge regenie
conda activate regenie_env
regenie --version
```

### Run Your First REGENIE Analysis

```bash
cd /home/sagemaker-user/GWASTutorial/32_whole_genome_regression

# Fix covariate file header
sed -i 's/#FID/FID/' ../05_PCA/plink_results_projected.sscore

# Run pre-written scripts
bash run_step1.sh  # ~5-10 minutes
bash run_step2.sh  # ~10-20 minutes

# Check results
head 1kg_eas_step2_BT_B1.regenie
```

**That's it!** You've run your first whole genome regression GWAS.

---

## 📖 Learning Path

### Beginner Path (1-2 hours)

1. **Read the Quick Start** (above) - 5 min
2. **Install REGENIE** - 10 min
3. **Read HANDS_ON_GUIDE.md Part 1-4** - 30 min
   - Understand what Step 1 and Step 2 do
4. **Run the analysis** - 20 min
   - Execute Step 1 and Step 2 commands
5. **Examine results** (HANDS_ON_GUIDE Part 5) - 15 min
   - Compare with PLINK results

**Goal:** Understand the basics and run a successful analysis

### Intermediate Path (2-4 hours)

Everything from Beginner Path, plus:

6. **Read REGENIE_vs_PLINK_COMPARISON.md** - 20 min
   - Understand when to use each method
7. **Complete HANDS_ON_GUIDE Part 6-8** - 45 min
   - Visualize results
   - Understand how correction works
   - Try advanced exercises
8. **Read README.md Concepts Section** - 30 min
   - Deep dive into ridge regression and LOCO

**Goal:** Understand the method deeply and make informed choices

### Advanced Path (4+ hours)

Everything from Intermediate Path, plus:

9. **Try Exercise variations** (HANDS_ON_GUIDE Part 8) - 60 min
   - Run without LOCO (see why it matters)
   - Test different block sizes
   - Run without PCs
10. **Read full README.md** - 30 min
    - Gene-based tests, interactions, survival analysis
11. **Explore your own data** - Variable
    - Apply to your research questions

**Goal:** Master REGENIE for production use

---

## 🎯 What You'll Achieve

After completing this tutorial, you will:

### Practical Skills
✅ Install and configure REGENIE  
✅ Run Step 1 (polygenic prediction)  
✅ Run Step 2 (association testing)  
✅ Interpret REGENIE outputs  
✅ Visualize GWAS results  
✅ Troubleshoot common errors  

### Conceptual Understanding
✅ How whole genome regression works  
✅ Why REGENIE doesn't compute GRM  
✅ What LOCO prevents (proximal contamination)  
✅ How ridge regression captures relatedness  
✅ When to use REGENIE vs PLINK vs LMM  

### Decision-Making Ability
✅ Choose appropriate GWAS method for your data  
✅ Understand trade-offs (speed vs accuracy)  
✅ Know when structure correction is sufficient  
✅ Recognize when relatedness matters  

---

## 💡 Key Concepts Preview

Before diving in, here are the core ideas:

### 1. **Two-Step Approach**
```
Step 1: Build ĝ (polygenic predictor) using ridge regression
        ↓
Step 2: Test SNPs: y = SNP + covariates + ĝ + error
```

### 2. **No GRM Needed**
```
Traditional LMM: K = XX'/m  (n×n matrix, huge!)
                 ↓
REGENIE: ĝ = ridge(X)  (just predictions, memory-efficient)
```

Both capture genetic similarity, REGENIE does it implicitly

### 3. **What ĝ Captures**
- Population structure (like PCs)
- Cryptic relatedness (unlike PCs)
- Polygenic background (genome-wide effects)

### 4. **LOCO (Leave-One-Chromosome-Out)**
```
When testing chr5 SNPs:
  Build ĝ from chr 1,2,3,4,6-22 (exclude chr5)
  
Why? Prevents ĝ from absorbing signal from causal variants on chr5
```

### 5. **When to Use It**
- ✅ Large samples (>50K) - speed matters
- ✅ Relatedness present - PLINK fails
- ✅ Multiple phenotypes - reuse Step 1
- ⚠️ Small, unrelated samples - PLINK sufficient

---

## 🔧 System Requirements

### Minimal (This Tutorial)
- **Memory:** 8 GB RAM
- **Disk:** 10 GB free space
- **CPU:** 2+ cores
- **Time:** ~30 minutes runtime

### Recommended
- **Memory:** 16+ GB RAM
- **Disk:** 50+ GB for larger datasets
- **CPU:** 8+ cores (parallel processing)
- **OS:** Linux or macOS (Docker for Windows)

### For Real Biobanks (>100K samples)
- **Memory:** 64-128 GB RAM
- **Disk:** 500GB-1TB
- **CPU:** 32+ cores
- **Time:** 2-8 hours

---

## 📞 Getting Help

### Resources

1. **REGENIE Documentation:** https://rgcgithub.github.io/regenie/
2. **REGENIE GitHub Issues:** https://github.com/rgcgithub/regenie/issues
3. **REGENIE Paper:** Mbatchou et al. (2021) Nature Genetics
4. **This Tutorial:** GWASTutorial Module 32

### Common Issues

**"Command not found: regenie"**
- Solution: Check installation, add to PATH

**"FID column not found"**
- Solution: `sed -i 's/#FID/FID/' covar_file.txt`

**"Firth correction failed"**
- Solution: Use `--approx` flag or increase `--pThresh`

**More:** See HANDS_ON_GUIDE.md Part 9 (Troubleshooting)

---

## 📈 Next Steps After This Tutorial

1. **Compare Methods** (Module 33)
   - Try GEMMA or BOLT-LMM
   - See how traditional LMM compares

2. **Advanced Features**
   - Gene-based tests (burden, SKAT)
   - Interaction testing (G×E, G×G)
   - Conditional analysis

3. **Apply to Your Data**
   - Format your genotypes/phenotypes
   - Run on real research questions
   - Publish results!

---

## ✅ Ready to Start?

Open **[HANDS_ON_GUIDE.md](HANDS_ON_GUIDE.md)** and begin with Part 1: Installation!

**Time commitment:**
- Quick run-through: 30 minutes
- Full understanding: 2-4 hours
- Mastery: 4+ hours practice

**You got this!** 🚀

---

*Created: 2026-07-31*
*Module 32: Whole Genome Regression by REGENIE*
*GWASTutorial - https://cloufield.github.io/GWASTutorial/*
