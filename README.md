
<!-- README.md is generated from README.Rmd. Please edit that file -->

# <a href='https://mattpm.github.io/dsb'><img src='man/figures/sticker.png' align="right" width="200" /></a> dsb: an R package for normalizing and denoising CITE-seq protein data

**Documentation below includes installation instructions, usage
tutorials for standard or sample multiplexed experiments, a workflow for
using dsb with Seurat, and integration with Bioconductor and Scanpy.
Also see FAQ
section.**

<!-- badges: start -->

<!-- [![Travis build status](https://travis-ci.org/MattPM/dsb.svg?branch=master)](https://travis-ci.org/MattPM/dsb) -->

<!-- badges: end -->

<a href='https://mattpm.github.io/dsb'><img src='man/figures/dsb_overview.png' height = "150"
/></a>

dsb (**d**enoised and **s**caled by **b**ackground) is a lightweight R
package developed in [John Tsang’s
Lab](https://www.niaid.nih.gov/research/john-tsang-phd) (NIH-NIAID) by
Matt Mulè, Andrew Martins and John Tsang. Through control experiments
and computational modeling designed to interrogate sources of noise in
antibody-based protein data from droplet single cell experiments, we
developed and validated dsb for removing noise and normalizing protein
data from single cell methods such as CITE-seq, REAP-seq, and Mission
Bio Tapestri.

[**See the dsb *Biorxiv*
preprint**](https://www.biorxiv.org/content/10.1101/2020.02.24.963603v1)
for details on the method.  
[A complete reproducible workflow
repository](https://github.com/niaid/dsb_manuscript) links to all data
and code for reproducing results/figures in the
paper.

<a href='https://github.com/niaid/dsb'><img src='man/figures/dsb_v_other.png' /></a>  
Follow tutorial below to use dsb in a workflow with Seurat or integrate
with Bioconductor or Scanpy.
<a href='https://mattpm.github.io/dsb'><img src='man/figures/dsb_protdist.png'/></a>

### Background

CITE-seq protein data suffers from substantial background noise of
unknown origin (for example, see supplementary fig 5a in [Stoeckius *et.
al.* 2017 *Nat.
Methods*](https://static-content.springer.com/esm/art%3A10.1038%2Fnmeth.4380/MediaObjects/41592_2017_BFnmeth4380_MOESM1_ESM.pdf))

1)  Based on unstained control experiments and modeling we found a major
    source of protein background noise comes from ambient, unbound
    antibody encapsulated in droplets.

2)  “Empty” droplets (containing ambient mRNA and antibody reads but no
    cell), outnumbering cell-containing droplets by 10-100 fold, capture
    the *ambient component* of protein background noise.

3)  Cell-to-cell technical variations such as stochastic differences in
    cell lysis/capture, RT efficiency, sequencing depth, and
    non-specific antibody binding can be estimated and removed by
    defining *the “technical component” of each cell’s* protein library.

[**Please consider citing the dsb
preprint**](https://www.biorxiv.org/content/10.1101/2020.02.24.963603v1)
if you use this package or find the protein noise modeling results
useful. If you have a question, please open an issue on the github site
<https://github.com/niaid/dsb> so the community can benefit. Package
maintainer / code: Matt Mulè (matthew.mule at nih.gov, permanent:
mattmule at gmail. General contact: John Tsang (john.tsang at nih.gov)

## Installation and quick overview with pre-loaded package data

The `DSBNormalizeProtein()` function  
1\) normalizes raw protein counts of cells (columns) by proteins (rows)
matrix `cells_citeseq_mtx` estimating the ambient component of noise
with `empty_drop_citeseq_mtx`, a matrix of background/empty droplets  
2\) models and removes the ‘technical component’ of each cell’s protein
library by setting `denoise.counts = TRUE`

``` r
# install dsb package directly from github: 
require(devtools); devtools::install_github(repo = 'MattPM/dsb')
library(dsb)

adt_norm = DSBNormalizeProtein(
  # remove ambient protien noise reflected in counts from empty droplets 
  cell_protein_matrix = cells_citeseq_mtx, # cell-containing droplet raw protein count matrix
  empty_drop_matrix = empty_drop_citeseq_mtx, # empty/background droplet raw protein counts
  
  # recommended step II: model and remove the technical component of each cell's protein library
  denoise.counts = TRUE, # (default = TRUE); run step II  
  use.isotype.control = TRUE, # (default = TRUE): use isotype controls to define technical components. 
  isotypecontrol.name.vec = rownames(cells_citeseq_mtx)[67:70] # vector of isotype control names
  )
```

## Tutorial with public 10X genomics data

Download **RAW (*not* filtered\!)** *feature / cellmatrix raw* from
[public 10X Genomics CITE-seq data
here](https://support.10xgenomics.com/single-cell-gene-expression/datasets/3.0.2/5k_pbmc_protein_v3).
Steps below use R 3.6 and Seurat 3. We emphasize normalized protein data
and raw RNA data from this workflow at step III can be used with Seurat,
Bioconductor or Python’s AnnData class in scanpy. We provide a suggested
workflow below that is similar to the CITE-seq workflow used in our
paper on baseline immune system states: [Kotliarov *et. al.* 2020 *Nat.
Medicine*](https://doi.org/10.1038/s41591-020-0769-8)

### Step Ia: load *raw* count alignment (e.g. Cell Ranger) output and define cell variables

Below we load the **raw** output from the Cell Ranger count alignment.
The raw output is a sparse matrix of **all possible cell barcodes** vs
proteins / mRNA. The number of cell barcodes ranges 500k-6M depending on
the kit/chemistry version. We define QC variables for each barcode. *If
you loaded multiple “lanes” of 10X data with the same pool of antibody
stained cells, we recommend combining all raw outputs together to
normalize (see FAQ section for code). If you used cell hashing and
superloading you can also consider an alternative workflow at the end of
this tutorial.*

``` r
library(dsb)
library(Seurat) # not a dsb dependency, only used for example workflow below
library(tidyverse) # not a dsb dependency, only used for visualizations below

# read raw data using the Seurat function "Read10X" 
raw = Read10X("data/10x_data/10x_pbmc5k_V3/raw_feature_bc_matrix/")

# Read10X detects the different assays and formats the output as a list: split the list
prot = raw$`Antibody Capture`
rna = raw$`Gene Expression`

# create a dataframe of simple qc stats for each droplet 
rna_size = log10(Matrix::colSums(rna))
prot_size = log10(Matrix::colSums(prot))
ngene = Matrix::colSums(rna > 0)
mtgene = grep(pattern = "^MT-", rownames(rna), value = TRUE)
propmt = Matrix::colSums(rna[mtgene, ]) / Matrix::colSums(rna)
md = as.data.frame(cbind(propmt, rna_size, ngene, prot_size))
md$bc = rownames(md)
```

### Step Ib: Define cell-containing and background droplets

Now that we loaded the raw output, we define ells and background
droplets. The number of cells you define should be in line with the
expected cell loading/recovery from your particular experient. If the
experiment used multiplexing and “superloading” and your raw output was
merged from multiple 10X lanes (see FAQ for code), we may expect to
recover \>50,000 cells + doublets; a population of droplets with higher
library size than the background peak(s) would therefore be observed in
the plots below 50,000 plus droplets. In this experiment, ~5000 cells
were loaded, and therefore *The rightmost peaks \> 3 are the
cell-containing
droplets*

``` r
p1 = ggplot(md[md$rna_size > 0, ], aes(x = rna_size)) + geom_histogram(fill = "dodgerblue") + ggtitle("RNA library size \n distribution")
p2 = ggplot(md[md$prot_size> 0, ], aes(x = prot_size)) + geom_density(fill = "firebrick2") + ggtitle("Protein library size \n distribution")
cowplot::plot_grid(p1, p2, nrow = 1)
```

<a href='https://github.com/niaid/dsb'><img src='man/figures/libsize.png' /></a>

Below we define background droplets as the major peak in the background
distribution between 1.4 and 2.5 log total protein counts. One could
also use the entire population of droplets from 0 to 2.5 with little
impact on normalized values (see the dsb paper for details). In
addition, we add mRNA quality control based filters to remove potential
low quality cells from both the background drops and the cells as we
would in any scRNAseq workflow (e.g. see Seurat tutorials or [Luecken
*et. al.* 2019 *Mol Syst
Biol*](https://www.embopress.org/doi/full/10.15252/msb.20188746)).

``` r
# define a vector of background / empty droplet barcodes based on protein library size and mRNA content
background_drops = md[md$prot_size > 1.4 & md$prot_size < 2.5 & md$ngene < 80, ]$bc
negative_mtx_rawprot = prot[ , background_drops] %>%  as.matrix()

# define a vector of cell-containing droplet barcodes based on protein library size and mRNA content 
positive_cells = md[md$prot_size > 2.8 & md$ngene < 3000 & md$ngene > 200 & propmt <0.2, ]$bc
cells_mtx_rawprot = prot[ , positive_cells] %>% as.matrix()

# check - are the number of cells in line with the expected cell recovery from your experiment? 
length(positive_cells)
```

\[1\] 3481  
3481 matches the number of cells we expected to recover; the protein
library size distribution of the cells (orange) and background droplets
(blue) used for normalization are highlighted below. \>67,000 negative
droplets are used to estimate the ambient background for normalizing the
3,481 single cells. (see FAQ for code to generate plot)
<a href='https://mattpm.github.io/dsb'><img src='man/figures/library_size_10x_dsb_distribution.png' height="200"  /></a>

### Step II: normalize protein data with the DSBNormalizeProtein Function.

``` r
#normalize protein data for the cell containing droplets with the dsb method. 
dsb_norm_prot = DSBNormalizeProtein(
                           cell_protein_matrix = cells_mtx_rawprot,
                           empty_drop_matrix = negative_mtx_rawprot,
                           denoise.counts = TRUE,
                           use.isotype.control = TRUE,
                           isotype.control.name.vec = rownames(cells_mtx_rawprot)[30:32])
```

The function returns a matrix of normalized protein values can be
integrated with any single cell analysis software, we provide an example
with Seurat, Bioconductor and Scanpy
below.

### Step III (option a): integration with Seurat

``` r
# filter raw protein, RNA and metadata to only include cell-containing droplets 
cells_rna = rna[ ,positive_cells]
md = md[positive_cells, ]

# create Seurat object(note min.cells is an mRNA filter not a cell filter, cells were filtered above)
s = CreateSeuratObject(counts = cells_rna, meta.data = md, assay = "RNA", min.cells = 20)

# add DSB normalized "dsb_norm_prot" protein data to the seurat object 
s[["CITE"]] = CreateAssayObject(data = dsb_norm_prot)
```

### Suggested Step IV: Protein based clustering + cluster annotation

  - This is similar to the workflow used in our paper [Kotliarov *et.
    al.* 2020 *Nat.
    Medicine*](https://doi.org/10.1038/s41591-020-0769-8) where we
    analyzed mRNA states *within* the interpretable clusters defined by
    dsb normalized protein
data.

<!-- end list -->

``` r
# define Euclidean distance matrix on dsb normalized protein data (without isotype controls)
dsb = s@assays$CITE@data[1:29, ]
p_dist = dist(t(dsb))
p_dist = as.matrix(p_dist)

# Cluster using Seurat 
s[["p_dist"]] = FindNeighbors(p_dist)$snn
s = FindClusters(s, resolution = 0.5, graph.name = "p_dist")
```

## Suggested Step V: cluster annotation based on average dsb normalized protein values

dsb normalized values aid in interpreting clustering results since the
background cell population is centered around 0 for each protein and the
positive values lie on an interpretable scale. The values for each cell
represent the log number of standard deviations of a given protein from
the expected noise as reflected by the protein distribution in empty
droplets +/- the residual of the fitted model to the cell-intrinsic
technical component.

``` r
# calculate the average of each protein separately for each cluster 
prots = rownames(s@assays$CITE@data)
adt_plot = adt_data %>% 
  group_by(seurat_clusters) %>% 
  summarize_at(.vars = prots, .funs = mean) %>% 
  column_to_rownames("seurat_clusters") 
# plot a heatmap of the average dsb normalized values for each cluster
pheatmap::pheatmap(t(adt_plot), color = viridis::viridis(25, option = "B"), 
                   fontsize_row = 8, border_color = NA)
```

<a href='https://mattpm.github.io/dsb'><img src='man/figures/dsb_heatmap.png' height="400"  /></a>

### Visualization of single cell protein levels on the interpretable dsb scale

Dimensionality reduction plots can sometimes be useful visualization
tools, for example to guide cluster annotation similar to the summarized
values above. Below we calculate UMAP embeddings for each cell (this can
also be done in *Seurat* or with *scater*) and make a combined dataframe
with UMAP embeddings dsb normalized protein values and all cell metadata
variables for visualization.

``` r
library(reticulate); use_virtualenv("r-reticulate")
library(umap)

# set umap config
config = umap.defaults
config$n_neighbors = 40
config$min_dist = 0.4

# run umap
ump = umap(t(s2_adt3), config = config)
umap_res = ump$layout %>% as.data.frame() 
colnames(umap_res) = c("UMAP_1", "UMAP_2")

# save results dataframe 
df_dsb = cbind(s@meta.data, umap_res, as.data.frame(t(s@assay$CITE@data)))
# or Seurat::RunUMAP(s, assay = "CITE") #then: cbind(s@meta.data, reductions$UMAP@cell_embeddings ...)
# visualizatons below were made directly from the data frame df_dsb above with ggplot
```

<a href='https://mattpm.github.io/dsb'><img src='man/figures/dsb_protdist.png'/></a>

# Integration with Bioconductor and Scanpy

### Step III (option b): integration with Bioconductor’s SingleCellExperiment class

Rather than Seurat you may wish to use the SingleCellExperiment class -
this can be accomplished with this alternative to step III above using
the following code:

``` r
library(SingleCellExperiment)
sce = SingleCellExperiment(assays = list(counts = count_rna), colData = md)
dsb_adt = SummarizedExperiment(as.matrix(count_prot))
altExp(sce, "CITE") = dsb_adt
# downstream analysis using bioconductor ... 
```

### Step III (option c): integration with Scanpy using python’s AnnData class

It is also simple to use dsb with the AnnData class in Python. [see here
for documentation on interoperability between scanpy Bioconductor and
Seurat](https://theislab.github.io/scanpy-in-R/). Since dsb is an R
package, to integrate with Scanpy, one would follow this normalization
tutorial steps 1 and 2 to read raw 10X data and normalize with dsb, then
the simplest option (since you are already in an R environment) is to
use reticulate to create scanpy’s AnnData class from raw RNA and dsb
denoised and normalized protein values. You can import that into a
python session or use in R. Anndata are not structured with different
asssays for multimodal data; we therefore need to merge the RNA and
protein data. See the current [scanpy CITE-seq
workflow](https://scanpy-tutorials.readthedocs.io/en/latest/cite-seq/pbmc5k.html)

``` r
library(reticulate); sc = import("scanpy")

# merge dsb-normalized protein and raw RNA data 
combined_dat = rbind(count_rna, dsb_norm_prot)
s[["combined_data"]] = CreateAssayObject(data = combined_dat)

# create Anndata Object 
adata_seurat = sc$AnnData(
    X   = t(GetAssayData(s,assay = "combined_data")),
    obs = seurat@meta.data,
    var = GetAssay(seurat)[[]]
)
```

## Quickstart V2 correcting ambient background without cell denoising (if isotype controls not measured)

We only recommend setting `denoise.counts = FALSE` If isotype controls
were not included in the experiment. This results in *not* defining the
technical component of each cell’s protein library. The values of the
normalized matrix returned are the number of standard deviations above
the expected ambient noise captured by empty droplets.

### no isotype controls - option 1 (recommended)

``` r
# suggested workflow if isotype controls are not included 
dsb_rescaled = DSBNormalizeProtein(cell_protein_matrix = cells_citeseq_mtx,
                                   empty_drop_matrix = empty_drop_citeseq_mtx, 
                                   # do not denoise each cell's technical component
                                   denoise.counts = FALSE)
```

### Quickstart V2, option II (for experiments without isotype controls)

*This is **not** recommended unless you confirm that µ1 and µ2 are not
correlated (see below).* The background mean for each cell inferred via
a per-cell gaussian mixture model (µ1) can theoretically be used alone
to define the cell’s technical component, however this assumes the
background mean has no expected biological variation. In our paper, we
found the background mean had weak but significant correlation with the
foreground mean (µ2) across single cells. The eigenvector through the
background mean and isotype controls (which defines the technical
component in the default dsb method) anchors the component of the
background mean associated with noise. The pipeline below allows you to
first check if µ1 and µ2 from the Gaussian mixture are correlated, in
which case one should use option 1 above. If µ1 and µ2 are not
correlated, this suggests (but does not prove) that µ1 alone does not
capture biological variaiton. **See FAQ for code to manually check this
correlation**

``` r

dsb_rescaled = dsb::DSBNormalizeProtein(cell_protein_matrix = cells_citeseq_mtx,
                                   empty_drop_matrix = empty_drop_citeseq_mtx, 
                                   # denoise with background mean only 
                                   denoise.counts = TRUE, use.isotype.control = FALSE)
```

## For experiments using sample barcoding / “cell hashing” antibodies

With sample multiplexing experiments, demultiplexing functions define a
“negative” cell population which can then be used to define background
droplets for dsb. In our data, dsb normalized values were nearly
identically distributed when dsb was run with background defiend by
demultiplexing functions or protein library size (see the paper).
[HTODemux function in
Seurat](https://satijalab.org/seurat/v3.1/hashing_vignette.html)
[deMULTIplex function from
Multiseq](https://github.com/chris-mcginnis-ucsf/MULTI-seq)
[demuxlet](https://github.com/statgen/demuxlet)

## Example workflow with *Seurat* using dsb in conjunction with background defined by sample demultiplexing functions

As in step Ia we load the **raw** output from cell ranger and use
*partial thresholding* (shown below) prior to demultiplexing. This has
multiple benefits: including more empty drops robustifies the background
population (e.g. as estimated by the k-medoids function implemented in
Seurat::HTODemux). More negatives will also result in more droplets
defined by the function as “negative” which will increase the confidence
in the estimate of background used by dsb. \#\#\# Alternative Step Ia:
load *raw* count alignment and define background with multiplexing
functions

``` r
# raw = Read10X see above -- path to cell ranger outs/raw_feature_bc_matrix ; 

# partial thresholding to slightly subset negative drops include all with 5 unique mRNAs
seurat_object = CreateSeuratObject(raw, min.genes = 5)

# demultiplex (positive.quantile can be tuned to dataset depending on size)
seurat_object = HTODemux(seurat_object, assay = "HTO", positive.quantile = 0.99)
Idents(seurat_object) = "HTO_classification.global"

# subset empty drop/background and cells 
neg_object = subset(seurat_object, idents = "Negative")
singlet_object = subset(seurat_object, idents = "Singlet")

# non sparse CITEseq data store more efficiently in a regular matrix
neg_adt_matrix = GetAssayData(neg_object, assay = "CITE", slot = 'counts') %>% as.matrix()
positive_adt_matrix = GetAssayData(singlet_object, assay = "CITE", slot = 'counts') %>% as.matrix()

# normalize the data with dsb
dsb_norm_prot = DSBNormalizeProtein(
                           cell_protein_matrix = cells_mtx_rawprot,
                           empty_drop_matrix = negative_mtx_rawprot,
                           denoise.counts = TRUE,
                           use.isotype.control = TRUE,
                           isotype.control.name.vec = rownames(cells_mtx_rawprot)[30:32])

# now add the normalized dat back to the object (the singlets defined above as "object")
singlet_object[["CITE"]] = CreateAssayObject(data = dsb_norm_prot)
```

# Frequently Asked Questions

**I get the error “Error in quantile.default(x, seq(from = 0, to = 1,
length = n)): missing values and NaN’s not allowed if ‘na.rm’ is FALSE”
What should I do?** - (see issue 6) this error occurs during denoising,
(denoise = TRUE) when you have antibodies with 0 counts or close to 0
across *all cells*. To get rid of this error, check the distributions of
the antibodies with e.g. `apply(cells_protein_matrix, 1, quantile)` to
find the protein(s) with basically no counts, then remove these from the
empty drops and the cells. (see issue 5)

**I get a "problem too large or memory exhausted error when I try to
convert to a regular R matrix** - (see issue 10) CITE-seq protein counts
don’t need a sparse representation-very likely this error is because
there are too many negative droplets defined (i.e. over 1 million). You
should be able to normalize datasets with 100,000+ cells and similar
numbers of negative droplets (or less) on a normal 16GB laptop. By
further narrowing in on the major background distribution, one should be
able to convert the cells and background to a normal R matrix which
should run successfully.

**I have multiple “lanes” of 10X data from the same pool of cells, how
should I run the workflow above?**

Droplets derived from the same pool of stained cells partitioned across
multiple lanes should be normalized together. To do this, you should
merge the raw output of each lane, then run step 1 in the workflow-note
that since the cell barcode names are the same for each lane in the raw
output, you need to add a string to each barcode to identify the lane of
origin to make the barcodes have unique names; here is one way to do
that: First, add each 10X lane *raw* output from Cell Ranger into a
separate directory in a folder “data”  
data  
|\_10xlane1  
  |\_outs  
    |\_raw\_feature\_bc\_matrix  
|\_10xlane2  
  |\_outs  
    |\_raw\_feature\_bc\_matrix

``` r
library(Seurat) # for Read10X helper function

# path_to_reads = here("data/")
umi.files = list.files(path_to_reads, full.names=T, pattern = "10x" )
umi.list = lapply(umi.files, function(x) Read10X(data.dir = paste0(x,"/outs/raw_feature_bc_matrix/")))
prot = rna = list()
for (i in 1:length(umi.list)) {
  prot[[i]] = umi.list[[i]]`Antibody Capture`
  rna[[i]] = umi.list[[i]]`Gene Expression`
  colnames(prot[[i]]) = paste0(colnames(prot[[i]]),"_", i )
  colnames(rna[[i]]) = paste0(colnames(rna[[i]]),"_", i )
}  
prot = do.call(cbind, prot)
rna = do.call(cbind, rna)
# proceed with step 1 in tutorial - define background and cell containing drops for dsb
```

**I have 2 batches, should I combine them into a single batch or
normalize each batch separately?** - (See issue 12) How much batch
variation there is depends on how much experiment-specific and expected
biological variability there is between the batches. In the dataset used
in the preprint, if we normalized with all background drops and cells in
a single normalization, the resulting dsb normalized values were highly
concordant with when we normalized each batch separately, this held true
with either definition of background drops used (i.e. based on
thresholding with the library size or based on hashing-see below). One
could try both and see which mitigates the batch variation the most. See
**issue 12** for example code to do
this.  
<a href='https://mattpm.github.io/dsb'><img src='man/figures/mbatch.png' height="300" /></a>

**How do I know whether I should set the denoise.counts argument to TRUE
vs FALSE?**  
In nearly all cases this argument should be set to TRUE and we highly
recommend that use.isotype.control is also set to TRUE when using
denoise.counts feature (this is the package default). The denoise.counts
argument specifies whether to remove cell-intrinsic technical noise by
defining and regressing out *cell-intrinsic technical factors* such as
efficiency of oligo tag capture, RT, sequencing depth and non-specific
antibody binding among other variables that contribute technical
variations not captured by protein counts in background droplets used in
dsb. The only reason not to use this argument is if the model
assumptions used to define the technical component are not expected to
be met by the particular experiment: denoise.counts models the negative
protein population (µ1) for each cell with a two-component Gaussian
mixture, making the conservative assumption that cells in the experiment
should be negative for a subset of the measured proteins. If you expect
all cells in your experiment express all / a vast majority of proteins
measured, this may not be an optimal assumption. In the paper we show
that the k=2 model provided an optimal fit (first plot below) compared
to other values of k in all 5 external datasets which measured between
just 14 to 86 proteins. The technical component is defined as the
primary latent component (λ) i.e. PC1 through each isotype control value
and µ1 for each cell. In all datasets, correlations between µ1 and 1)
each individual isotype control (second plot) and 2) the average of all
four isotype controls (third plot) were higher than those between the
isotype control themselves suggesting shared variation between the
independently inferred µ1 and isotype controls captured unobserved,
latent factors contributing to technical noise. λ was associated with
the protein library size (last plot, this is also true within protein
defined cell type-see paper) suggesting that the shared component of
variation in these variables reflect technical noise, but the library
size alone should not be used as a normalization factor (as is typical
for mRNA data) due to potential biological contributions and bias in the
small subset of proteins measured relative to the surface proteome (see
paper for a more detailed
discussion).

<a href='https://mattpm.github.io/dsb'><img src='man/figures/10k v3 model data.png' /></a>

**Code for generating the count and density histogram of positive and
negative droplet protein distributions?** assuming at step II, after
setting thresholds:

``` r
# at step 3 
pv = md[positive_cells, ]; pv$class = "cell_containing"
nv = md[background_drops, ]; nv$class = "background"
ddf = rbind(pv, nv)
p = ggplot(ddf, aes(x = prot_size, fill = class, color = class )) + theme_bw() + 
  geom_histogram(aes(y=..count..), alpha=0.5, bins = 50,position="identity")+ 
  ggtitle(paste0("theoretical max barcodes = ", nrow(md), "\n", "cell containing drops after QC = ", nrow(pv), "\n", "negative droplets = ", nrow(nv))) + theme(legend.position = c(0.8, 0.7))
xtop = cowplot::axis_canvas(p, axis = "x") + geom_density(data = ddf, aes(x = prot_size, fill = class), alpha = 0.5)
p2 = cowplot::ggdraw(cowplot::insert_xaxis_grob(p, xtop, grid::unit(.4, "null"), position = "top"))
```

<a href='https://mattpm.github.io/dsb'><img src='man/figures/distribs.png' height = "200" /></a>

**Checking the correlation between µ1 and µ2 for denoising without
isotype controls** This refers to Quickstart V2 Option 2 above. To
define each cell’s technical component its background mean alone without
anchoring using isotype controls, confirm µ1 and µ2 have a low
correlation:

``` r
# step 1: confirm low correlation between µ1 and µ2 from the Gaussian mixture.
adtu_log1 = log(empty_drop_citeseq_mtx + 10) 
adt_log1 = log(cells_citeseq_mtx + 10)

# rescale 
mu_u1 = apply(adtu_log1, 1 , mean)
sd_u1 = apply(adtu_log1, 1 , sd)
norm_adt = apply(adt_log1, 2, function(x) (x  - mu_u1) / sd_u1) 

# run per-cellgaussian mixture 
library(mclust)
cm = apply(norm_adt, 2, function(x) {
            g = Mclust(x, G=2, warn = TRUE, verbose = FALSE)  
            return(g) 
        })
# tidy model fit data 
mu1 = lapply(cm, function(x){ x$parameters$mean[1] }) %>% base::unlist(use.names = FALSE)
mu2 = lapply(cm, function(x){ x$parameters$mean[2] }) %>% base::unlist(use.names = FALSE)
# test correlation 
cor.test(mu1, mu2)
```

### NIAID statement

A review of this code has been conducted, no critical errors exist, and
to the best of the authors knowledge, there are no problematic file
paths, no local system configuration details, and no passwords or keys
included in this code. This open source software comes as is with
absolutely no warranty.

### Terms of Use

By using this software, you agree this software is to be used for
research purposes only. Any presentation of data analysis using dsb will
acknowledge the software according to the guidelines below. How to
cite:  
Please cite this software as: 1. Mulè MP, Martins AJ, Tsang JS.
Normalizing and denoising protein expression data from droplet-based
single cell profiling. bioRxiv. 2020;2020.02.24.963603.

Primary author(s): Matt Mulè Organizational contact information:
General: john.tsang AT nih.gov, code: mulemp AT nih.gov  
Date of release: Oct 7 2020  
Version: 0.1.0 License details: see package  
Description: dsb:an R package for normalizing and denoising CITEseq
protein data  
Usage instructions: Provided in this markdown  
Example(s) of usage: Provided in this markdown and vignettes  
Proper attribution to others, when applicable: NA

### code check

``` r
# checked repo for PII and searched for strings with paths 
# code check 
library(lintr)
library(here)
fcn = suppressMessages(list.files(here("R/"), ".r", full.names = TRUE))
vignette = list.files(path = here("vignettes/dsb_normalizing_CITEseq_data.R"))
# code check 
scp = c(fcn, vignette) %>% as.list()
lt = suppressMessages((lapply(scp, lintr::lint)))
```
