Using VisCello for Single Cell Data Visualization
================
Qin Zhu, Kim Lab & Tan Lab, University of Pennsylvania


## Playground

Here's an example app with data from Paul et al. (2015): https://cello.shinyapps.io/base/. It shows basic features offered by VisCello.
Here's another app based on VisCello for interactive exploration of C. elegans embryogenesis data: https://cello.shinyapps.io/celegans/

Install VisCello with code below:

``` r
install.packages("devtools") 
devtools::install_github("qinzhu/VisCello") # install
library(VisCello) # load
cello() # launch with example data
```

To put in your own dataset into VisCello for visualization, follow guidance below.

## General data requirement

`VisCello` requires two main data object - an **`ExpressionSet`** object and a **`Cello`** object (or list of Cello objects), plus one configuration file.

The **`ExpressionSet`** object is a general class from Bioconductor. See <https://bioconductor.org/packages/release/bioc/vignettes/Biobase/inst/doc/ExpressionSetIntroduction.pdf> for details. The expression matrix holds 3 key datasets: the expression matrix and the normalized count matrix, the meta data for the samples (cells), and the meta data for the genes.

The **`Cello`** object is an S4 class specifically designed for visualizing subsets of the single cell data - by storing dimension reduction results of (subsets of) cells that are present in the global ExpressionSet, and any local meta information about the cells, such as clustering results.

Lastly, a simple configuration file needs to be editted by user to let VisCello know where's `ExpressionSet` and `Cello` list of your data, and what's the name of study, etc.

This vignette will describe the preprocessing step for inputing data into VisCello.

## Prepare ExpressionSet object

The `data-raw` folder contains an example dataset and associated meta information from Paul et al. (2015). The files in the folder are:

-   `subset_GSE72857.txt`: Raw gene expression matrix, with column as cells and rows as genes, rownames must be unique.
-   `subset_GSE72857_log2norm.txt`: Log2 normalized expression matrix, same dimension as raw matrix. You can also input any other type of normalized data as long as it matches the dimension of raw data.
-   `subset_GSE72857_pmeta.txt`: Meta data for the cells.

Load the data into R and convert them into ExpressionSet using the following code:

``` r
library(Matrix)
library(Biobase)
raw_df <- read.table("data-raw/GSE72857_raw.txt")
norm_df <- read.table("data-raw/GSE72857_log2norm.txt")
pmeta <- read.table("data-raw/GSE72857_pmeta.txt")
fmeta <- read.table("data-raw/GSE72857_fmeta.txt")

# Feature meta MUST have rownames the same as the rownames of expression matrix - this can be gene name or gene id (but must not have duplicates). 
rownames(fmeta) <- fmeta$id

eset <- new("ExpressionSet",
             assayData = assayDataNew("environment", exprs=Matrix(as.matrix(raw_df), sparse = T), norm_exprs = Matrix(as.matrix(norm_df), sparse = T)),
             phenoData =  new("AnnotatedDataFrame", data = pmeta),
             featureData = new("AnnotatedDataFrame", data =fmeta))
```

A few additional work needs to be done here: some columns in pmeta, such as cell state should be treated as factors rather than numeric values.

``` r
factor_cols <- c("Mouse_ID", "cluster", "State")
for(c in factor_cols) {
    pData(eset)[[c]] <- factor(pData(eset)[[c]])
}

saveRDS(eset, "inst/app/data/eset.rds") 
```

Note that you can save `eset.rds` to any place if you don't want to deploy VisCello on ShinyServer. But if it is for deploy, you need to save to `inst/app/data/`.

Now the expression data and meta information required by VisCello is in place.

### [Alternative] Convert common objects to ExpressionSet

* Seurat object

``` r
fmeta <- data.frame(symbol = rownames(seurat@data)) # Seurat does not seem to store feature meta data as of version 3.0
rownames(fmeta) <- fmeta$symbol
eset <- new("ExpressionSet",
             assayData = assayDataNew("environment", exprs=Matrix(as.matrix(seurat@raw.data), sparse = T), 
             norm_exprs = Matrix(as.matrix(seurat@data), sparse = T)),
             phenoData =  new("AnnotatedDataFrame", data = seurat@meta.data),
             featureData = new("AnnotatedDataFrame", data = fmeta))
```

* Monocle object

``` r
fmeta <- fData(cds)
fmeta[[1]] <- fmeta$gene_short_name # Make first column gene name
eset <- new("ExpressionSet",
             assayData = assayDataNew("environment", exprs=Matrix(as.matrix(exprs(cds)), sparse = T),  
             norm_exprs = Matrix(log2(Matrix::t(Matrix::t(FM)/sizeFactors(cds))+1), sparse = T)), # Note this is equivalent to 'log' normalization method in monocle, you can use other normalization function
             phenoData =  new("AnnotatedDataFrame", data = pData(cds)),
             featureData = new("AnnotatedDataFrame", data = fmeta))
```

* SingleCellExperiment/SummarizedExperiment object

``` r
fmeta <- rowData(sce)
# if rowData empty, you need to make a new fmeta with rownames of matrix: fmeta <- data.frame(symbol = rownames(sce)); rownames(fmeta) <- fmeta[[1]]
fmeta[[1]] <- fmeta$symbol # Make sure first column is gene name
eset <- new("ExpressionSet",
             assayData = assayDataNew("environment", exprs=Matrix(sce@assays$data$counts, sparse = T), # Change 'counts' to raw count matrix
             norm_exprs = Matrix(sce@assays$data$norm_counts, sparse = T)), # Change 'norm_counts' to raw count matrix
             phenoData =  new("AnnotatedDataFrame", data = colData(sce)),
             featureData = new("AnnotatedDataFrame", data = fmeta))
```

## Prepare Cello object

`Cello` object allows embedding of multiple dimension reduction results for different subsets of cells. This allows "zoom-in" analysis on subset of cells as well as differential expression analysis on locally defined clusters. The basic structure of `Cello` is as follows:

-   `name`: Name of the cell subset
-   `idx` : The index of the cell subset in global expression set.
-   `proj` : Named list of projections such as PCA, t-SNE and UMAP.
-   `pmeta` : (Optional) local meta data containing the meta data that's specific to this local cell subset. For example, in global meta data, only one "Cluster" column is allowed. But what if you have different clustering results for different cell subsets? You can store this subset-dependent result inside the local pmeta slot of cello.
-   `notes` : (Optional) information about the cell subset to display to the user

Create `Cello` for VisCello, note at least one dimension reduction result must be computed and put in `cello@proj`:

``` r
library(irlba)
source("data-raw/preprocess_scripts/cello.R")
source("data-raw/preprocess_scripts/cello_dimR.R")
# Creating a cello for all the cells
cello <- new("Cello", name = "All Data", idx = 1:ncol(eset)) # Index is basically the column index, here all cells are included 
# Code for computing dimension reduction, not all of them is necessary, and you can input your own dimension reduction result into the cello@proj list.
# It is also recommended that you first filter your matrix to remove low expression genes and cells, and input a matrix with variably expressed genes
cello <- compute_pca_cello(eset, cello, num_dim = 50) # Compute PCA 
cello <- compute_tsne_cello(eset, cello, use_dim = 30, n_component = 2, perplexity = 30) # Compute t-SNE
cello <- compute_umap_cello(eset, cello, use_dim = 30, n_component = 2, umap_path = "data-raw/preprocess_scripts/python/umap.py") # Compute UMAP, need reticulate and UMAP (python package) to be installed
cello <- compute_umap_cello(eset, cello, use_dim = 30, n_component = 3, umap_path = "data-raw/preprocess_scripts/python/umap.py") # 3D UMAP
```

If you already computed your dimension reduction result and wants to make a cello for it, use following R code:

``` r
cello <- new("Cello", name = "My cell subset", idx = cell_index)  # cell_index is which cells from ExpressionSet are used for the dimension reduction
my_tsne_proj <- read.table("path_to_tsne_projection")
my_umap_proj <- read.table("path_to_umap_projection") # You can put in as many dimension reduction result as you want
cello@proj <- list("My t-SNE [2D]" = my_tsne_proj, "My UMAP [3D]" = my_umap_proj) # Put a 2D or 3D in the name to tell VisCello if you want to visualize this as a 2D plot or as a 3D rotatable plot, if 2D there must be 2 columns for each dimension, if 3D there must be 3 columns.
```


Create a list to store `Cello` objects and save to data location.

``` r
clist <- list()
clist[["All Data"]] <- cello
saveRDS(clist, "inst/app/data/clist.rds") 
```

You can create multiple `Cello` hosting visualization information of different subsets of the cells. Say people want to zoom in onto GMP for further analysis:

``` r
zoom_type <- "GMP"
cur_idx <-  which(eset$cell_type == zoom_type)
cello <- new("Cello", name = zoom_type, idx = cur_idx) 

# Refilter gene by variation if necessary
cello <- compute_pca_cello(eset[,cur_idx], cello, num_dim = 50) # Compute PCA 
cello <- compute_tsne_cello(eset[,cur_idx], cello, use_dim = 15, n_component = 2, perplexity = 30) # Compute t-SNE
cello <- compute_umap_cello(eset[,cur_idx], cello, use_dim = 15, n_component = 2, umap_path = "data-raw/preprocess_scripts/python/umap.py") # Compute UMAP
cello <- compute_umap_cello(eset[,cur_idx], cello, use_dim = 15, n_component = 3, umap_path = "data-raw/preprocess_scripts/python/umap.py") # 3D UMAP
clist[[zoom_type]] <- cello
saveRDS(clist, "inst/app/data/clist.rds") 
```

Note that you can save `clist.rds` to any place (recommend same folder with eset.rds and config.yml for each study) if you don't want to deploy VisCello on ShinyServer. But if it is for deploy, you need to save to `inst/app/data/`.

Now all the data required to run cello has been preprocessed.


## Prepare configure file and launch

Download example configure file from: https://github.com/qinzhu/VisCello/blob/master/inst/app/data/config.yml

-   `study_name`: Appear as title for the app.
-   `study_description` : Appear as footer for the app.
-   `eset_path` : Where you save eset.rds
-   `clist_path` : Where you save clist.rds
-   `organism` : support mouse, human and c. elegans.
-   `feature_name_column` : important! What's the column name of fData(eset) that corresponds to gene symbol.
-   `feature_id_column` : Optional, What's the column name of fData(eset) that corresponds to gene id, set same as `feature_name_column` if you don't have gene id.

Now VisCello is ready to go! To launch Viscello, in R:

``` r
library(VisCello)
cello(config_file = "~/path_to_config.yml")
```


## Host VisCello on a server

To host VisCello on a server, you need either ShinyServer (<https://www.rstudio.com/products/shiny/shiny-server/>) or use the shinyapps.io service (<https://www.shinyapps.io/>). 

STEP 1: Install VisCello from github
``` r
install_github("qinzhu/VisCello") # STEP 1
```

STEP 2: **IMPORTANT** Git clone VisCello from github, replace `inst/app/data/eset.rds`, `inst/app/data/clist.rds`, `inst/app/data/config.yml` with your own data. Do not change path for eset.rds and clist.rds in config.yml.

STEP 3: Set the repositories to bioconductor in R, and then only deploy the inst/app/ folder that contains your own data.
``` r
options(repos = BiocManager::repositories()) 
rsconnect::deployApp("inst/app/", account = "cello", appName = "base") # change account to your own account
```


Please cite VisCello properly if you use VisCello to host your dataset.

## Cite VisCello

Packer, J.S., Zhu, Q., Huynh, C., Sivaramakrishnan, P., Preston, E., Dueck, H., Stefanik, D., Tan, K., Trapnell, C., Kim, J. and Waterston, R.H., 2019. A lineage-resolved molecular atlas of C. elegans embryogenesis at single cell resolution. BioRxiv, p.565549.



Reference
---------

Paul, Franziska, Ya’ara Arkin, Amir Giladi, Diego Adhemar Jaitin, Ephraim Kenigsberg, Hadas Keren-Shaul, Deborah Winter, et al. 2015. “Transcriptional Heterogeneity and Lineage Commitment in Myeloid Progenitors.” *Cell* 163 (7). Elsevier: 1663–77.
