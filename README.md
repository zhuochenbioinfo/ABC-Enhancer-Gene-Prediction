# Activity by Contact Model of Enhancer-Gene Specificity

The Activity-by-Contact (ABC) model predicts which enhancers regulate which genes on a cell type specific basis. This repository contains the code needed to run the ABC model as well as small sample data files, example commands, and some general tips and suggestions. We provide a brief description of the model below, see Fulco, Nasser et al [1] for a full description.

v0.2 is the recommended version for the majority of users. There are some minor methodological differences between v0.2 and the model as described in [1]. These differences are related to Hi-C data processing and were implemented to improve the speed and scalability of the codebase. As such ABC scores computing using v0.2 will not exactly match those published in [1], although they will be very close. The codebase used to generate the results in [1] is available in the NG2019 branch of this repository. The NG2019 branch is no longer maintained.

If you use the ABC model in published research, please cite:

[1] Fulco CP, Nasser J, Jones TR, Munson G, Bergman DT, Subramanian V, Grossman SR, Anyoha R, Doughty BR, Patwardhan TA, Nguyen TH, Kane M, Perez EM, Durand NC, Lareau CA, Stamenova EK, Aiden EL, Lander ES & Engreitz JM. Activity-by-contact model of enhancer–promoter regulation from thousands of CRISPR perturbations. Nat. Genet. 51, 1664–1669 (2019). https://www.nature.com/articles/s41588-019-0538-0


## Requirements
For each cell-type, the inputs to the ABC model are:

 * Required Inputs
 	* bam file for Dnase-Seq or ATAC-Seq (indexed and sorted)
 	* bam file for H3K27ac ChIP-Seq (indexed and sorted)
 * Optional Inputs
 	* Hi-C data (see the Hi-C section below)
 	* A measure of gene expression (see gene expression section)

In addition the following (non-cell-type specific) genome annotation files are required

 * bed file containing gene annotations (may change across cell types if using cell-type specific TSS's)
 * bed file containing chromosome annotations

### Dependencies

The codebase relies on the following dependancies (tested version provided in 
parentheses):

```
Python (3.6.4)
samtools (0.1.19)
bedtools (2.26.0)
Tabix (0.2.5) - Partial dependancy
MACS2 (2.1.1.20160309) - Partial dependancy
Java (1.8) - Partial dependancy
Juicer Tools (1.7.5) - Partial dependancy

Python packages:
pyranges (0.0.55)
numpy (1.15.2)
pandas (0.23.4)
scipy (0.18.1)
pysam (0.15.1)
pyBigWig (0.3.2) - Partial dependancy
```

## Description of the ABC Model

The Activity by Contact (ABC) model is designed to represent a mechanistic model in which enhancers activate gene transcription upon enhancer-promoter contact. In a simple conception of such a model, the quantitative effect of an enhancer depends on the frequency with which it contacts a promoter multiplied by the strength of the enhancer (i.e., the ability of the enhancer to activate transcription upon contacting a promoter). Moreover, the contribution of a specific enhancer to a gene’s expression should depend on the surrounding context (ie, the strength and contact frequency of other enhancers for the gene).

To convert this conceptual framework into a practical score (which can be applied genome-wide), we formulated the ABC score:

ABC score for effect of element E on gene G = Activity of E × Contact frequency between E and G /  Sum of (Activity × Contact Frequency) over all candidate elements within 5 Mb.

Operationally, Activity (A) is defined as the geometric mean of the read counts of DNase-seq and H3K27ac ChIP-seq at an element E, and Contact (C) as the KR normalized Hi-C contact frequency between E and the promoter of gene G. Elements are defined as ~500bp regions centered on DHS peaks. 

Note that the ABC model only considers candidate elements and genes on the same chromosome. It does not make interchromosomal predictions.

 
## Running the ABC Model
Running the ABC model consists of the following steps:

 1. Define candidate enhancer regions
 2. Quantify enhancer activity
 3. Compute ABC Scores

### Step 1. Define candidate elemets

'Candidate elements' are the set of putative enhancer elements for which ABC Scores will be computed. A typical way to define candidate elements is by calling peaks on a DNase-Seq or ATAC-Seq bam file. In this implementation we first call peaks using MACS2 and then process these peaks using ```makeCandidateRegions.py```. 

```makeCandidateRegions.py``` will take as input the narrowPeak file produced by MACS2 and then perform the following processing steps:

 1. Count DNase-seq reads in each peak and retain the top N peaks with the most read counts
 2. Resize each of these N peaks to be a fixed number of base pairs centered on the peak summit
 3. Remove any regions listed in the 'blocklist' and include any regions listed in the 'includelist'
 4. Merge any overlapping regions

See 'Defining Candidate Enhancers' section below for more details.

Activate macs conda environment via: 

```
conda env create -f macs.yml 
```

Sample commands:

#Call peaks
```
macs2 callpeak \
-t example_chr22/input_data/Chromatin/wgEncodeUwDnaseK562AlnRep1.chr22.bam \
-n wgEncodeUwDnaseK562AlnRep1.chr22.macs2 \
-f BAM \
-g hs \
-p .1 \
--call-summits \
--outdir example_chr22/ABC_output/Peaks/ 

#Sort narrowPeak file
bedtools sort -faidx example_chr22/reference/chr22 -i example_chr22/ABC_output/Peaks/wgEncodeUwDnaseK562AlnRep1.chr22.macs2_peaks.narrowPeak > example_chr22/ABC_output/Peaks/wgEncodeUwDnaseK562AlnRep1.chr22.macs2_peaks.narrowPeak.sorted
```

Activate abc conda environment via:

```
conda env create -f abcenv.yml
```

#Call candidate regions
```
python src/makeCandidateRegions.py \
--narrowPeak example_chr22/ABC_output/Peaks/wgEncodeUwDnaseK562AlnRep1.chr22.macs2_peaks.narrowPeak.sorted \
--bam example_chr22/input_data/Chromatin/wgEncodeUwDnaseK562AlnRep1.chr22.bam \
--outDir example_chr22/ABC_output/Peaks/ \
--chrom_sizes example_chr22/reference/chr22 \
--regions_blocklist reference/wgEncodeHg19ConsensusSignalArtifactRegions.bed \
--regions_includelist example_chr22/reference/RefSeqCurated.170308.bed.CollapsedGeneBounds.TSS500bp.chr22.bed \
--peakExtendFromSummit 250 \
--nStrongestPeaks 3000 
```

We recommend using ```--nStrongestPeaks 150000``` when making genome-wide peak calls. ```3000``` is just used for the small example on chr22. 

We have found that on some systems, MACS2 and Python3 are incompatible. It may be necessary to change virtual environments, or installed packages/dotkits after running MACS2. 

Main output files:

* ***macs2_peaks.narrowPeak**: MACS2 narrowPeak file
* ***macs2_peaks.narrowPeak.candidateRegions.bed**: filtered, extended and merged peak calls from MACS2. These are the candidate regions used in downstream scripts.

### Step 2. Quantifying Enhancer Activity: 

```run.neighborhoods.py``` will count DNase-seq (or ATAC-seq) and H3K27ac ChIP-seq reads in candidate enhancer regions. It also makes GeneList.txt, which counts reads in gene bodies and promoter regions.

Replicate epigenetic experiments should be included as comma delimited list of files. Read counts in replicate experiments will be averaged when computing enhancer Activity.

Sample Command:

```
python src/run.neighborhoods.py \
--candidate_enhancer_regions example_chr22/ABC_output/Peaks/wgEncodeUwDnaseK562AlnRep1.chr22.macs2_peaks.narrowPeak.sorted.candidateRegions.bed \
--genes example_chr22/reference/RefSeqCurated.170308.bed.CollapsedGeneBounds.chr22.bed \
--H3K27ac example_chr22/input_data/Chromatin/ENCFF384ZZM.chr22.bam \
--DHS example_chr22/input_data/Chromatin/wgEncodeUwDnaseK562AlnRep1.chr22.bam,example_chr22/input_data/Chromatin/wgEncodeUwDnaseK562AlnRep2.chr22.bam \
--expression_table example_chr22/input_data/Expression/K562.ENCFF934YBO.TPM.txt \
--chrom_sizes example_chr22/reference/chr22 \
--ubiquitously_expressed_genes reference/UbiquitouslyExpressedGenesHG19.txt \
--cellType K562 \
--outdir example_chr22/ABC_output/Neighborhoods/ 
```

Main output files:

  * **EnhancerList.txt**: Candidate enhancer regions with Dnase-seq and H3K27ac ChIP-seq read counts
  * **GeneList.txt**: Dnase-seq and H3K27ac ChIP-seq read counts on gene bodies and gene promoter regions


### Step 3. Computing the ABC Score

Compute ABC scores by combining Activity (as calculated by ```run.neighborhoods.py```) and Hi-C.

Sample Command:

```
python src/predict.py \
--enhancers example_chr22/ABC_output/Neighborhoods/EnhancerList.txt \
--genes example_chr22/ABC_output/Neighborhoods/GeneList.txt \
--HiCdir example_chr22/input_data/HiC/raw/ \
--chrom_sizes example_chr22/reference/chr22 \
--hic_resolution 5000 \
--scale_hic_using_powerlaw \
--threshold .02 \
--cellType K562 \
--outdir example_chr22/ABC_output/Predictions/ \
--make_all_putative
```
### Step 4. Get Prediction Files for Variant Overlap 

Perform filtering strategies to prepare prediction files for downstream variant overlap analysis 
- this step is already being performed in src/predict.py

Sample Command:
```
python src/getVariantOverlap.py \
--all_putative EnhancerPredictionsAllPutative.txt.gz \
--chrom_sizes example_chr22/reference/chr22 \
--outdir . 			
```

The main output files are:

* **EnhancerPredictions.txt**: all element-gene pairs with scores above the provided threshold. Only includes expressed genes and does not include promoter elements. This file defines the set of 'positive' predictions of the ABC model.
* **EnhancerPredictionsFull.txt**: same as above but includes more columns. See <https://docs.google.com/spreadsheets/d/1UfoVXoCxUpMNPfGypvIum1-RvS07928grsieiaPX67I/edit?usp=sharing> for column definitions
* **EnhancerPredictions.bedpe**: Same as above in .bedpe format. Can be loaded into IGV.
* **EnhancerPredictionsAllPutative.txt.gz**: ABC scores for all element-gene pairs. Includes promoter elements and pairs with scores below the threshold. Only includes expressed genes. This file includes both the 'positive' and 'negative' predictions of the model. (use ```--make_all_putative``` to generate this file). 
* **EnhancerPredictionsAllPutativeNonExpressedGenes.txt.gz**: Same as above for non-expressed genes. This file is provided for completeness but we generally do not recommend using these predictions.


The default threshold of 0.02 corresponds to approximately 70% recall and 60% precision [1].

## Defining Candidate Enhancers
'Candidate elements' are the set of putative enhancers; ABC scores will be computed for all 'Candidate elements' within 5Mb of each gene. In computing the ABC score, the product of DNase-seq (or ATAC-seq) and H3K27ac ChIP-seq reads will be counted in each candidate element. Thus the candidate elements should be regions of open (nucleasome depleted) chromatin of sufficient length to capture H3K27ac marks on flanking nucleosomes. In [1], we defined candidate regions to be 500bp (150bp of the DHS peak extended 175bp in each direction). 

Given that the ABC score uses absolute counts of Dnase-seq reads in each region, ```makeCandidateRegions.py ``` selects the strongest peaks as measured by absolute read counts (not by pvalue). In order to do this, we first call peaks using a lenient significance threshold (.1 in the above example) and then consider the peaks with the most read counts. This procedure implicitly assumes that the active karyotype of the cell type is constant.

We recommend removing elements overlapping regions of the genome that have been observed to accumulate anomalous number of reads in epigenetic sequencing experiments (‘block-listed regions’). For convenience, we provide the list of block-listed regions available from <https://sites.google.com/site/anshulkundaje/projects/blacklists>.

We also force the candidate enhancer regions to include gene promoters, even if the promoter is not among the candidate elements with the strongest signals genome-wide in a cell type, by specifying `--region_includelist`. 

## Contact and Hi-C
Given that cell-type specific Hi-C data is more difficult to generate than ATAC-seq or ChIP-seq, we have explored alternatives to using cell-type specific Hi-C data. It has been shown that Hi-C contact frequencies generally follow a powerlaw relationship (with respect to genomic distance) and that many TADs, loops and other structural features of the 3D genome are **not** cell-type specific (Sanborn et al 2015, Rao et al 2014). 


We have found that, for most genes, using an average Hi-C profile in the ABC model gives approximately equally good performance as using a cell-type specific Hi-C profile. To facilitate making ABC predictions in a large panel of cell types, including those without cell type-specific Hi-C data, we have provided an average Hi-C matrix (averaged across 10 cell lines, at 5kb resolution). 

### Format of Hi-C data
The ABC model supports two Hi-C data formats.
 
* Juicer format: three column 'sparse matrix' format representation of a Hi-C matrix.
* bedpe format: More general format which can support variable and arbitrary bin sizes. 

### Average Hi-C
The celltypes used for averaging are: GM12878, NHEK, HMEC, RPE1, THP1, IMR90, HUVEC, HCT116, K562, KBM7. 

Average Hi-C data can be downloaded from: <ftp://ftp.broadinstitute.org/outgoing/lincRNA/average_hic/average_hic.v2.191020.tar.gz> (20 GB)

### Pipeline to Download Hi-C data

The below pipeline will download a Hi-C matrix from Juicebox (in .hic format) and fit a powerlaw distribution.

```
#Download hic matrix file from juicebox
python src/juicebox_dump.py \
--hic_file https://hicfiles.s3.amazonaws.com/hiseq/k562/in-situ/combined_30.hic \
--juicebox "java -jar juicer_tools.jar" \
--outdir example_chr22/input_data/HiC/raw/ \
--chromosomes 22
```

```
#Fit HiC data to powerlaw model and extract parameters
python src/compute_powerlaw_fit_from_hic.py \
--hicDir example_chr22/input_data/HiC/raw/ \
--outDir example_chr22/input_data/HiC/raw/powerlaw/ \
--maxWindow 1000000 \
--minWindow 5000 \
--resolution 5000 \
--chr 'chr22'
```

### Contact data in bedpe formats
Contact data can also be loaded in a generic bedpe format. This supports promoter-capture Hi-C and mixed-resolution Hi-C matrices. Use ```--hic_type bedpe``` in ```predict.py``` to enable this. Note that if contact data is provided in bedpe format, some of the default Hi-C normalizations are not applied. (For example, the code will not adjust the diagonal of the contact matrix)

The bedpe file should be a tab delimited file containing 8 columns (chr1,start1,end1,chr2,start2,end2,name,score) where score denotes the contact frequency. An example is given here: ```input_data/HiC/raw/chr22/chr22.bedpe.gz```

```
python src/predict.py \
--enhancers example_chr22/ABC_output/Neighborhoods/EnhancerList.txt \
--genes example_chr22/ABC_output/Neighborhoods/GeneList.txt \
--HiCdir example_chr22/input_data/HiC/raw/ \
--hic_type bedpe \
--hic_resolution 5000 \
--scale_hic_using_powerlaw \
--threshold .02 \
--cellType K562 \
--outdir example_chr22/ABC_output/Predictions/ \
--make_all_putative
```

### ABC model without experimental contact data
If experimentally derived contact data is not available, one can run the ABC model using the powerlaw estimate only. In this case the ```--HiCdir``` argument should be excluded from ```predict.py``` and the ```--score_column powerlaw.Score``` argument should be included in ```predict.py```. In this case the ```ABC.Score``` column of the predictions file will be set to ```NaN```. The ```powerlaw.Score``` column of the output prediction files will be the relevant Score column to use.

## Gene Expression in ABC
The ABC model is designed to predict the effect of enhancers on expressed genes. If a gene is not expressed in a given cell type (or cell state) then we assume it does not have any activating enhancers (enhancers for which inhibition of the enhancer would lead to decrease in gene expression). Thus we typically only report enhancer-gene connections for expressed genes.

In the absence of expression data, DNase-seq and H3K27ac ChIP-seq at the gene promoter can be used as a proxy for expression. We suggest only considering enhancer-gene connections for genes with sufficiently active promoters (for instance in the top 60% of gene promoters in the cell type)

## Quantile Normalization

The ABC Score uses the quantitative signal of Hi-C, ATAC-Seq and H3K27ac ChIP-Seq. As such it is sensitive to differences in these datasets due to experimental or technical procedures. For example, ABC scores computed on an epigenetic dataset with low signal to noise will be lower than ABC scores computed on an epigenetic dataset with high signal to noise. 

In an effort to make ABC scores comparable across cell types, the ABC model code supports quantile normalizing the epigenetic signal in candidate enhancer regions to some reference. The reference we provide in this repository is computed on ~160,000 DHS peaks in K562. 

Empirically, we have found that applying quantile normalization makes ABC predictions more comparable across cell types (particularly there is substantial variability in the signal to noise ratio of the epigenetic datasets across cell types). However, care should be taken as quantile normalization may not be applicable to all circumstances.

Additionally, the threshold value on the ABC score of .02 is calculated based on the K562 epigenetic data. 

Quantile normalization can be applied using ```--qnorm src/EnhancersQNormRef.K562.txt``` in ```run.neighborhoods.py```

## Tips and Comments

* Accurate transcription start site annotations are critical. The ABC model uses the TSS of the gene in order to assign enhancer-promoter contact frequency. If the TSS annotation is inaccurate (off by >5kb) it will lead to inaccurate predictions.
* We have found that ubiquitously expressed genes appear insensitive to the effects of distal enhancers. For completeness, this code calculates the ABC score for all genes and flags ubiquitously expressed genes.
* The size of candidate enhancer elements is important. For example, if two candidate regions are merged, then the ABC score of the merged region will be approximately the sum of the ABC scores for each individual region.
* In our testing the ABC model typically predicts on average ~3 distal enhancers per expressed gene. If you run the model on a cell type and find a large deviation from this number (say <2 or >5) this may mean the ABC model is not well calibrated in the cell type. Typical remedies are to use quantile normalization, scale Hi-C or to lower/raise the cutoff on the ABC score.

## Contact
Please submit a github issue with any questions or if you experience any issues/bugs. 


