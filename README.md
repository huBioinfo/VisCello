Using VisCello for Single Cell Data Visualization
================
Qin Zhu, Kim Lab, University of Pennsylvania

General data requirement
------------------------

`VisCello` requires two main data object - an `ExpressionSet` object and a `Cello` object (or list of Cello objects).

The `ExpressionSet` object is a general class from Bioconductor. See <https://bioconductor.org/packages/release/bioc/vignettes/Biobase/inst/doc/ExpressionSetIntroduction.pdf> for details. The expression matrix holds 3 key datasets: the expression matrix and the normalized count matrix, the meta data for the samples (cells), and the meta data for the genes.

The `Cello` object is an S4 class specifically designed for visualizing subsets of the single cell data - by storing dimension reduction results of (subsets of) cells that are present in the global ExpressionSet, and any local meta information about the cells, such as clustering results.

This vignette will describe the preprocessing step for inputing data into VisCello.

Prepare ExpressionSet object
----------------------------

The `data-raw` folder contains an example dataset and associated meta information from Paul et al. (2015). The files in the folder are:

-   `subset_GSE72857.txt`: Raw gene expression matrix, with column as cells and rows as genes.
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
# feature meta Must contain a column id - ensemble id, and correspoinding symbol, the matrix rownames should be ensemble id.
colnames(fmeta) <- c("id", "symbol") 
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

Now the expression data and meta information required by VisCello is in place.

Prepare Cello object
--------------------

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
saveRDS(clist, "inst/app/data/clist.rds") # Note if you change the file name from clist.rds to other name, you need to change the readRDS code in global.R
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

Now all the data required to run cello has been preprocessed.

Customize visualization paramaters if necessary
-----------------------------------------------

Finnally, you can set which meta data you want to show to the user and what should be their default color palatte and data scale in `global.R`.

For example, change this code to exclude meta columns that you don't want to show the user.

``` r
meta_order <- meta_order[!meta_order %in% c("Amp_batch_ID", "well_coordinates", "Number_of_cells", "Plate_ID", "Batch_desc", "Pool_barcode", "Cell_barcode", "RMT_sequence", "no_expression")]
```

Set default color palatte and data scale by editting this chunk of code

``` r
# Edit if necessary, Which meta to show 
pmeta_attr$dpal <- ifelse(pmeta_attr$is_numeric, "RdBu", "Set1")
pmeta_attr$dscale <- ifelse(pmeta_attr$is_numeric, "log10", NA)
pmeta_attr$dscale[which(pmeta_attr$meta_id %in% c("Size_Factor"))] <- "identity"
```

Compile (install) the VisCello package
--------------------------------------

Open R and do the following:

``` r
setwd("/Users/yourname/Download/") # Replace the path with path to PARENT folder of VisCello
install.packages("devtools")
if (!requireNamespace("BiocManager", quietly = TRUE))
    install.packages("BiocManager")
BiocManager::install("monocle", update = F, ask=F)
BiocManager::install("clusterProfiler", update = F, ask=F)
BiocManager::install("org.Mm.eg.db", update = F, ask=F)
BiocManager::install("org.Hs.eg.db", update = F, ask=F)
devtools::install_local("VisCello.base")
```

Now VisCello is ready to go! To launch Viscello, in R:

``` r
library(VisCello.base)
viscello()
```

Host VisCello on a server
-------------------------

To host VisCello on a server, you need either ShinyServer (<https://www.rstudio.com/products/shiny/shiny-server/>) or use the shinyapps.io service (<https://www.shinyapps.io/>). Start Viscello on server by calling above code.

Reference
---------

Paul, Franziska, Ya’ara Arkin, Amir Giladi, Diego Adhemar Jaitin, Ephraim Kenigsberg, Hadas Keren-Shaul, Deborah Winter, et al. 2015. “Transcriptional Heterogeneity and Lineage Commitment in Myeloid Progenitors.” *Cell* 163 (7). Elsevier: 1663–77.
