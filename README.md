
# Introduction

The `XeniumIO` package provides functions to import 10X Genomics Xenium
Analyzer data into R. The package is designed to work with the output of
the Xenium Analyzer, which is a software tool that processes Visium
spatial gene expression data. The package provides functions to import
the output of the Xenium Analyzer into R, and to create a `TENxXenium`
object that can be used with other Bioconductor packages.

# Supported Formats

## TENxIO

The 10X suite of packages support multiple file formats. The following
table lists the supported file formats and the corresponding classes
that are imported into R.

| **Extension**       | **Class**     | **Imported as**                    |
|---------------------|---------------|------------------------------------|
| .h5                 | TENxH5        | SingleCellExperiment w/ TENxMatrix |
| .mtx / .mtx.gz      | TENxMTX       | SummarizedExperiment w/ dgCMatrix  |
| .tar.gz             | TENxFileList  | SingleCellExperiment w/ dgCMatrix  |
| peak_annotation.tsv | TENxPeaks     | GRanges                            |
| fragments.tsv.gz    | TENxFragments | RaggedExperiment                   |
| .tsv / .tsv.gz      | TENxTSV       | tibble                             |

## VisiumIO

| **Extension**  | **Class**          | **Imported as**   |
|----------------|--------------------|-------------------|
| spatial.tar.gz | TENxSpatialList    | DataFrame list \* |
| .parquet       | TENxSpatialParquet | tibble \*         |

## XeniumIO

| **Extension** | **Class** | **Imported as** |
|---------------|-----------|-----------------|
| .zarr.zip     | TENxZarr  | (TBD)           |

# Installation

``` r
if (!require("BiocManager", quietly = TRUE))
    install.packages("BiocManager")))

BiocManager::install("XeniumIO")
```

# Loading package

``` r
library(XeniumIO)
```

# XeniumIO

The `TENxXenium` class has a `metadata` slot for the `experiment.xenium`
file. The `resources` slot is a `TENxFileList` or `TENxH5` object
containing the cell feature matrix. The `coordNames` slot is a vector
specifying the names of the columns in the spatial data containing the
spatial coordinates. The `sampleId` slot is a scalar specifying the
sample identifier.

``` r
TENxXenium(
    resources = "path/to/matrix/folder/or/file",
    xeniumOut = "path/to/xeniumOut/folder",
    sample_id = "sample01",
    format = c("mtx", "h5"),
    boundaries_format = c("parquet", "csv.gz"),
    spatialCoordsNames = c("x_centroid", "y_centroid"),
    ...
)
```

The `format` argument specifies the format of the `resources` object,
either “mtx” or “h5”. The `boundaries_format` allows the user to choose
whether to read in the data using the `parquet` or `csv.gz` format.

# Example Folder Structure

Note that the `xeniumOut` unzipped folder must contain the following
files:

        *outs
        ├── cell_feature_matrix.h5
        ├── cell_feature_matrix.tar.gz
        |   ├── barcodes.tsv*
        |   ├── features.tsv*
        |   └── matrix.mtx*
        ├── cell_feature_matrix.zarr.zip
        ├── experiment.xenium
        ├── cells.csv.gz
        ├── cells.parquet
        ├── cells.zarr.zip
        [...]

Note that currently the `zarr` format is not supported as the
infrastructure is currently under development.

## Xenium class

The `resources` slot should either be the `TENxFileList` from the `mtx`
format or a `TENxH5` instance from an `h5` file. The boundaries can
either be a `TENxSpatialParquet` instance or a `TENxSpatialCSV`. These
classes are automatically instantiated by the constructor function.

``` r
showClass("TENxXenium")
#> Class "TENxXenium" [package "XeniumIO"]
#> 
#> Slots:
#>                                                                                                                      
#> Name:                             resources                           boundaries                           coordNames
#> Class:               TENxFileList_OR_TENxH5 TENxSpatialParquet_OR_TENxSpatialCSV                            character
#>                                                                                                                      
#> Name:                              sampleId                              colData                             metadata
#> Class:                            character                   TENxSpatialParquet                           XeniumFile
```

## `import` method

The `import` method for a `TENxXenium` instance returns a
`SpatialExperiment` class object. Dispatch is only done on the `con`
argument. See `?BiocIO::import` for details on the generic. The `import`
function call is meant to be a simple call without much input. For more
details in the package, see `?TENxXenium`.

``` r
getMethod("import", c(con = "TENxXenium"))
#> Method Definition:
#> 
#> function (con, format, text, ...) 
#> {
#>     sce <- import(con@resources, ...)
#>     metadata <- import(con@metadata)
#>     coldata <- import(con@colData)
#>     SpatialExperiment::SpatialExperiment(assays = list(counts = assay(sce)), 
#>         rowData = rowData(sce), mainExpName = mainExpName(sce), 
#>         altExps = altExps(sce), sample_id = con@sampleId, colData = as(coldata, 
#>             "DataFrame"), spatialCoordsNames = con@coordNames, 
#>         metadata = list(experiment.xenium = metadata, polygons = import(con@boundaries)))
#> }
#> <bytecode: 0x622eb1e584c8>
#> <environment: namespace:XeniumIO>
#> 
#> Signatures:
#>         con          format text 
#> target  "TENxXenium" "ANY"  "ANY"
#> defined "TENxXenium" "ANY"  "ANY"
```

# Importing an Example Xenium Dataset

The following code snippet demonstrates how to import a Xenium Analyzer
output into R. The `TENxXenium` object is created by specifying the path
to the `xeniumOut` folder. The `TENxXenium` object is then imported into
R using the `import` method for the `TENxXenium` class.

First, we cache the ~12 MB file to avoid downloading it multiple times
(via
*[BiocFileCache](https://bioconductor.org/packages/3.22/BiocFileCache)*).

``` r
zipfile <- paste0(
    "https://mghp.osn.xsede.org/bir190004-bucket01/BiocXenDemo/",
    "Xenium_Prime_MultiCellSeg_Mouse_Ileum_tiny_outs.zip"
)
destfile <- XeniumIO:::.cache_url_file(zipfile)
```

We then create an output folder for the contents of the zipped file. We
use the same name as the zip file but without the extension (with
`tools::file_path_sans_ext`).

``` r
outfold <- file.path(
    tempdir(), tools::file_path_sans_ext(basename(zipfile))
)
if (!dir.exists(outfold))
    dir.create(outfold, recursive = TRUE)
```

We now unzip the file and use the `outfold` as the `exdir` argument to
`unzip`. The `outfold` variable and folder will be used as the
`xeniumOut` argument in the `TENxXenium` constructor. Note that we use
the `ref = "Gene Expression"` argument in the `import` method to pass
down to the internal `splitAltExps` function call. This will set the
`mainExpName` in the `SpatialExperiment` object.

``` r
unzip(
    zipfile = destfile, exdir = outfold, overwrite = FALSE
)
TENxXenium(xeniumOut = outfold) |>
    import(ref = "Gene Expression")
#> class: SpatialExperiment 
#> dim: 5006 36 
#> metadata(2): experiment.xenium polygons
#> assays(1): counts
#> rownames(5006): ENSMUSG00000052595 ENSMUSG00000030111 ... ENSMUSG00000055670 ENSMUSG00000027596
#> rowData names(3): ID Symbol Type
#> colnames(36): aaamobki-1 aaclkaod-1 ... olbjkpjc-1 omjmdimk-1
#> colData names(13): cell_id transcript_counts ... segmentation_method sample_id
#> reducedDimNames(0):
#> mainExpName: Gene Expression
#> altExpNames(5): Deprecated Codeword Genomic Control Negative Control Codeword Negative Control Probe Unassigned Codeword
#> spatialCoords names(2) : x_centroid y_centroid
#> imgData names(0):
```

Note that you may also use the `swapAltExp` function to set a
`mainExpName` in the `SpatialExperiment` but this is not recommended.
The operation returns a `SingleCellExperiment` which has to be coerced
back into a `SpatialExperiment`. The coercion also loses some metadata
information particularly the `spatialCoords`.

``` r
TENxXenium(xeniumOut = outfold) |>
    import() |>
    swapAltExp(name = "Gene Expression") |>
    as("SpatialExperiment")
#> class: SpatialExperiment 
#> dim: 5006 36 
#> metadata(1): TENxFileList
#> assays(1): counts
#> rownames(5006): ENSMUSG00000052595 ENSMUSG00000030111 ... ENSMUSG00000055670 ENSMUSG00000027596
#> rowData names(3): ID Symbol Type
#> colnames(36): aaamobki-1 aaclkaod-1 ... olbjkpjc-1 omjmdimk-1
#> colData names(13): cell_id transcript_counts ... segmentation_method sample_id
#> reducedDimNames(0):
#> mainExpName: Gene Expression
#> altExpNames(5): Genomic Control Negative Control Codeword Negative Control Probe Unassigned Codeword Deprecated Codeword
#> spatialCoords names(0) :
#> imgData names(0):
```

The dataset was obtained from the 10X Genomics website under the [X0A
v3.0
section](https://www.10xgenomics.com/support/software/xenium-onboard-analysis/latest/resources/xenium-example-data#test-data-v3-0)
and is a subset of the Xenium Prime 5K Mouse Pan Tissue & Pathways
Panel. The link to the data can be seen as the `url` input above and
shown below for completeness.

<https://cf.10xgenomics.com/samples/xenium/3.0.0/Xenium_Prime_MultiCellSeg_Mouse_Ileum_tiny/Xenium_Prime_MultiCellSeg_Mouse_Ileum_tiny_outs.zip>

# Session Info

<details>
<summary>
Click to expand <code>sessionInfo()</code>
</summary>
<pre><code>R version 4.5.0 Patched (2025-04-15 r88148)
Platform: x86_64-pc-linux-gnu
Running under: Ubuntu 24.04.2 LTS
&#10;Matrix products: default
BLAS/LAPACK: /usr/lib/x86_64-linux-gnu/openblas-pthread/libopenblasp-r0.3.26.so;  LAPACK version 3.12.0
&#10;locale:
 [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C               LC_TIME=en_US.UTF-8        LC_COLLATE=en_US.UTF-8    
 [5] LC_MONETARY=en_US.UTF-8    LC_MESSAGES=en_US.UTF-8    LC_PAPER=en_US.UTF-8       LC_NAME=C                 
 [9] LC_ADDRESS=C               LC_TELEPHONE=C             LC_MEASUREMENT=en_US.UTF-8 LC_IDENTIFICATION=C       
&#10;time zone: America/New_York
tzcode source: system (glibc)
&#10;attached base packages:
[1] stats4    stats     graphics  grDevices utils     datasets  methods   base     
&#10;other attached packages:
 [1] BiocStyle_2.37.0            XeniumIO_1.1.1              TENxIO_1.11.1               SingleCellExperiment_1.31.0
 [5] SummarizedExperiment_1.39.0 Biobase_2.69.0              GenomicRanges_1.61.0        GenomeInfoDb_1.45.3        
 [9] IRanges_2.43.0              S4Vectors_0.47.0            BiocGenerics_0.55.0         generics_0.1.3             
[13] MatrixGenerics_1.21.0       matrixStats_1.5.0           colorout_1.3-2             
&#10;loaded via a namespace (and not attached):
 [1] tidyselect_1.2.1         dplyr_1.1.4              blob_1.2.4               arrow_19.0.1.1           filelock_1.0.3          
 [6] fastmap_1.2.0            BiocFileCache_2.99.0     digest_0.6.37            lifecycle_1.0.4          RSQLite_2.3.9           
[11] magrittr_2.0.3           compiler_4.5.0           rlang_1.1.6              tools_4.5.0              utf8_1.2.4              
[16] yaml_2.3.10              knitr_1.50               VisiumIO_1.5.1           askpass_1.2.1            S4Arrays_1.9.0          
[21] bit_4.6.0                curl_6.2.2               DelayedArray_0.35.1      abind_1.4-8              rsconnect_1.3.4         
[26] withr_3.0.2              purrr_1.0.4              sys_3.4.3                grid_4.5.0               cli_3.6.5               
[31] rmarkdown_2.29           crayon_1.5.3             rstudioapi_0.17.1        httr_1.4.7               tzdb_0.5.0              
[36] rjson_0.2.23             BiocBaseUtils_1.11.0     DBI_1.2.3                cachem_1.1.0             assertthat_0.2.1        
[41] parallel_4.5.0           BiocManager_1.30.25      XVector_0.49.0           vctrs_0.6.5              Matrix_1.7-3            
[46] jsonlite_2.0.0           hms_1.1.3                bit64_4.6.0-1            archive_1.1.12           magick_2.8.6            
[51] credentials_2.0.2        glue_1.8.0               codetools_0.2-20         BiocIO_1.19.0            UCSC.utils_1.5.0        
[56] tibble_3.2.1             pillar_1.10.2            rappdirs_0.3.3           htmltools_0.5.8.1        openssl_2.3.2           
[61] R6_2.6.1                 dbplyr_2.5.0             httr2_1.1.2              gert_2.1.5               vroom_1.6.5             
[66] evaluate_1.0.3           lattice_0.22-7           readr_2.1.5              SpatialExperiment_1.19.0 memoise_2.0.1           
[71] Rcpp_1.0.14              SparseArray_1.9.0        whisker_0.4.1            xfun_0.52                pkgconfig_2.0.3         </code></pre>
</details>
