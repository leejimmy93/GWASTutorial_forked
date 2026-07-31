# GWAS Tools Comparison: Plant vs Human Genomics

## Overview

This document compares major GWAS tools and their suitability for plant biology research versus human genetics, based on their design, features, and adoption in each field.

## Tool Comparison

### 1. regenie (Regeneron Genetics Center)

**Primary Focus:** Human genetics (biobank-scale studies)

**Approach:** 
- Two-step whole genome regression model
- Step 1: Builds polygenic prediction model using SNP subset
- Step 2: Tests each variant while accounting for polygenic background

**Strengths:**
- Extremely fast with large sample sizes (100K+ samples)
- Handles unbalanced case/control ratios (Firth approximation)
- Actively maintained (2020+)
- Good documentation and support

**Limitations for Plants:**
- Optimized for human population structure and diploid organisms
- No plant-specific experimental designs (field trials, GxE interactions)
- No polyploidy handling (critical for wheat, potato, sugarcane)
- Overkill for smaller studies (<5K samples)

**Plant Usage:** ~5% of recent papers, increasing as datasets grow

---

### 2. BOLT-LMM (UK Biobank)

**Primary Focus:** Human genetics (biobank-scale)

**Approach:**
- Bayesian mixed model with extreme computational efficiency
- Designed for 500K+ sample human studies

**Strengths:**
- Very fast computation
- Handles complex population structure well
- Works well for large-scale studies

**Limitations for Plants:**
- Optimized for human diploid genomes
- Less flexible for plant-specific experimental designs
- Limited plant breeding trial support

**Plant Usage:** ~8% of recent papers, mostly large plant biobanks

---

### 3. statgenGWAS ⭐ (Biometris, Wageningen)

**Primary Focus:** Plant biology and breeding

**Approach:**
- R package specifically designed for plant breeding trials
- Integrates with agricultural experimental design frameworks

**Strengths:**
- **Purpose-built for plant research**
- Handles multi-environment trials (GxE interactions)
- Spatial models for field layout effects (row/column adjustments)
- Replicated experimental designs
- Integration with ASReml and SpATS spatial models
- Understands agricultural trial structures

**Common Use Cases:**
```r
# Handles:
# - Randomized complete block designs
# - Augmented designs
# - Spatial autocorrelation in field trials
# - Multi-location trials
# - Genotype × Environment interactions
```

**Plant Usage:** ~20% of plant genomics papers, growing rapidly in breeding programs

**Typical Applications:**
- Maize yield trials across locations
- Potato disease resistance with field replication
- Wheat quality traits with spatial variation
- Rice multi-environment performance

---

### 4. GEMMA (Zhou Lab)

**Primary Focus:** Cross-species (both plant and animal genetics)

**Approach:**
- Linear mixed models (LMM)
- Bayesian sparse linear mixed models (BSLMM)
- Well-balanced feature set

**Strengths:**
- Widely adopted and trusted across domains
- Excellent documentation
- Good balance of features and usability
- Handles relatedness and population structure elegantly
- Strong community support

**Limitations:**
- Less specialized than statgenGWAS for complex field designs
- No built-in spatial modeling for agricultural trials

**Plant Usage:** ~35% of plant genomics papers - most established LMM tool in plants

**Why Popular in Plants:**
- Mature, stable software
- Works well for natural populations and diversity panels
- Good for both simple and moderately complex designs
- Widely cited and trusted methodology

---

### 5. fastLMM (Microsoft Research)

**Primary Focus:** Human genetics (historical)

**Approach:**
- One of the first efficient LMM implementations for GWAS (early 2010s)
- Linear mixed models with computational optimizations

**Strengths:**
- Pioneering tool for LMM-based GWAS
- Handles relatedness/population structure

**Current Status:**
- Development has slowed significantly
- Superseded by newer, faster tools (BOLT-LMM, regenie, GEMMA)
- Less plant-specific functionality

**Plant Usage:** Seen in older plant papers (2012-2016), now largely replaced

---

## Ranking for Plant Biology

| Rank | Tool | Best For | Sample Size Sweet Spot |
|------|------|----------|------------------------|
| 1 | **statgenGWAS** | Plant breeding trials, field experiments, GxE | 100-10K |
| 2 | **GEMMA** | Natural populations, diversity panels, general use | 100-50K |
| 3 | **BOLT-LMM** / **regenie** | Large plant biobanks, cross-species comparisons | 50K+ |
| 4 | **fastLMM** | Historical/legacy analysis | Not recommended |

---

## Decision Matrix: When to Use Each Tool

### Sample Size-Based Selection

| Sample Size | Best Choice | Alternative | Rationale |
|-------------|-------------|-------------|-----------|
| <1K samples | GEMMA, PLINK | statgenGWAS (if complex design) | GEMMA trusted, PLINK simple |
| 1K-10K samples | GEMMA, statgenGWAS | PLINK, regenie | Sweet spot for both |
| 10K-50K samples | regenie, BOLT-LMM | GEMMA | Speed becomes important |
| 50K+ samples | regenie, BOLT-LMM | - | Computational efficiency critical |
| **Complex field trial** | **statgenGWAS** | ASReml + custom | Only tool with spatial models |

### Study Design-Based Selection

| Study Design | Recommended Tool | Why |
|--------------|------------------|-----|
| Multi-environment trial | statgenGWAS | Built-in GxE modeling |
| Field trial with spatial variation | statgenGWAS | Row/column/spatial autocorrelation |
| Natural population | GEMMA | Standard for diversity studies |
| Large biobank | regenie, BOLT-LMM | Computational efficiency |
| Simple case-control | PLINK2 | Simplicity, teaching |
| Breeding program | statgenGWAS | Agricultural design support |

### Organism-Specific Considerations

**Plant-Specific Challenges:**
- **Polyploidy** (wheat, potato, sugarcane) - Requires specialized tools, none of these handle natively
- **Highly structured populations** (breeding programs) - statgenGWAS, GEMMA
- **High heterozygosity** (outcrossing species) - GEMMA, custom solutions
- **Presence/absence variation** (pangenome) - Specialized tools (TASSEL, GAPIT)

**Human Genetics Optimizations:**
- Diploid organisms
- Large sample sizes (biobanks)
- Electronic health records integration
- Standardized phenotyping

---

## Real-World Usage Estimates

Based on recent genomics literature (approximate percentages):

### Plant Genomics Papers
- **GEMMA:** 35% - Most established
- **PLINK/PLINK2:** 25% - Simple analyses, teaching
- **statgenGWAS:** 20% - Growing in breeding programs
- **BOLT-LMM:** 8%
- **regenie:** 5% - Increasing with larger datasets
- **Other (TASSEL, GAPIT, custom):** 7%

### Human Genetics Papers
- **PLINK/PLINK2:** 30% - Standard baseline
- **regenie:** 25% - Growing rapidly
- **BOLT-LMM:** 20% - UK Biobank standard
- **GEMMA:** 10%
- **SAIGE:** 10% - Unbalanced designs
- **Other:** 5%

---

## Plant-Specific GWAS Tools (Not Covered Above)

### TASSEL
- **Focus:** Plant genetics, especially maize
- **Strengths:** GUI, diversity analysis, kinship calculation
- **Used by:** Plant breeding community, USDA researchers

### GAPIT (R package)
- **Focus:** Plant GWAS with emphasis on model comparison
- **Strengths:** Multiple models (MLM, CMLM, FarmCPU), easy to use in R
- **Used by:** Plant researchers preferring R workflows

### FarmCPU
- **Focus:** Addressing population structure confounding
- **Strengths:** Fixed and random model circulating probability unification
- **Used by:** Plant studies with complex population structure

---

## Key Takeaways

1. **No one-size-fits-all:** Tool choice depends on study design, sample size, and organism

2. **Plant biology has specialized needs:**
   - Field trial designs require spatial modeling (statgenGWAS)
   - GxE interactions are common (statgenGWAS)
   - Polyploidy is frequent (requires specialized tools)

3. **Human genetics tools work for plants but aren't optimal:**
   - regenie and BOLT-LMM are fast but lack plant-specific features
   - Good for large-scale plant biobanks
   - Less suitable for breeding trials

4. **GEMMA is the "safe middle ground":**
   - Works well for both plant and human genetics
   - Mature, trusted, well-documented
   - Good choice when unsure

5. **Sample size matters:**
   - Small studies (<5K): GEMMA, PLINK sufficient
   - Large studies (>50K): regenie, BOLT-LMM necessary for speed

6. **Experimental design trumps sample size for plants:**
   - A 500-sample field trial needs statgenGWAS more than a 10K-sample diversity panel needs regenie

---

## Recommendations for This Tutorial

**Current approach (PLINK2):**
- ✅ Excellent for teaching fundamentals
- ✅ Simple, widely understood
- ✅ Good for case/control traits
- ❌ Doesn't model relatedness as elegantly as LMMs
- ❌ No plant-specific experimental designs

**Potential additions:**
1. **Add GEMMA module** - Show LMM approach, standard in field
2. **Add statgenGWAS section** - Demonstrate plant-specific designs (if tutorial expands to breeding)
3. **Add regenie comparison** - Show scaling to large datasets
4. **Comparison module** - Side-by-side: PLINK vs GEMMA vs regenie

---

## References for Further Reading

### Tool Documentation
- **regenie:** [rgcgithub.github.io/regenie](https://rgcgithub.github.io/regenie/)
- **BOLT-LMM:** [alkesgroup.broadinstitute.org/BOLT-LMM](https://alkesgroup.broadinstitute.org/BOLT-LMM/)
- **statgenGWAS:** [CRAN statgenGWAS](https://cran.r-project.org/web/packages/statgenGWAS/)
- **GEMMA:** [github.com/genetics-statistics/GEMMA](https://github.com/genetics-statistics/GEMMA)
- **PLINK2:** [www.cog-genomics.org/plink/2.0/](https://www.cog-genomics.org/plink/2.0/)

### Key Papers
- Loh et al. (2015) - BOLT-LMM original paper, *Nature Genetics*
- Zhou & Stephens (2012) - GEMMA original paper, *Nature Genetics*
- Mbatchou et al. (2021) - regenie paper, *Nature Genetics*
- van Rossum et al. (2020) - statgenGWAS package, *Bioinformatics*

---

*Document created: 2026-07-31*
*Based on discussion about GWAS tool selection for plant and human genomics research*
