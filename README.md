PCAtools: everything Principal Components Analysis
================
Kevin Blighe
2019-04-30

-   [Introduction](#introduction)
-   [Installation](#installation)
    -   [1. Download the package from Bioconductor](#download-the-package-from-bioconductor)
    -   [2. Load the package into R session](#load-the-package-into-r-session)
-   [Quick start](#quick-start)
    -   [A SCREE plot](#a-scree-plot)
    -   [A bi-plot](#a-bi-plot)
    -   [A pairs plot](#a-pairs-plot)
    -   [A loadings plot](#a-loadings-plot)
    -   [An eigencor plot](#an-eigencor-plot)
-   [Advanced features](#advanced-features)
    -   [Determine PCs accounting for 80% explained variation](#determine-pcs-accounting-for-80-explained-variation)
    -   [Modify bi-plots](#modify-bi-plots)
        -   [Colour by a factor from the metadata, add lines through center, and add legend](#colour-by-a-factor-from-the-metadata-add-lines-through-center-and-add-legend)
        -   [Supply custom colours, add more lines, and increase legend size](#supply-custom-colours-add-more-lines-and-increase-legend-size)
        -   [Change shape based on tumour grade, remove connectors, and add titles](#change-shape-based-on-tumour-grade-remove-connectors-and-add-titles)
        -   [Remove labels, modify line types, remove gridlines, and increase point size](#remove-labels-modify-line-types-remove-gridlines-and-increase-point-size)
        -   [Colour by a continuous variable (colour controlled by ggplot2 engine); plot other PCs](#colour-by-a-continuous-variable-colour-controlled-by-ggplot2-engine-plot-other-pcs)
    -   [Quickly explore potentially informative PCs via a pairs plot](#quickly-explore-potentially-informative-pcs-via-a-pairs-plot)
    -   [Determine the variables that drive variation among each PC](#determine-the-variables-that-drive-variation-among-each-pc)
    -   [Correlate the principal components back to the clinical data](#correlate-the-principal-components-back-to-the-clinical-data)
    -   [Plot the entire project on a single panel](#plot-the-entire-project-on-a-single-panel)
-   [Acknowledgments](#acknowledgments)
-   [Session info](#session-info)
    -   [References](#references)

Introduction
============

Principal Components Analysis (PCA) is a very powerful technique that has wide applicability in data science, bioinformatics, and further afield. It was initially developed to analyse large volumes of data in order to tease out the differences/relationships between the logical entities being analysed. It extracts the fundamental structure of the data without the need to build any model to represent it. This 'summary' of the data is arrived at through a process of reduction that can transform the large number of variables into a lesser number that are uncorrelated (i.e. the ‘principal components'), whilst at the same time being capable of easy interpretation on the original data (Blighe 2013) (Blighe and Lewis 2018).

Installation
============

1. Download the package from Bioconductor
-----------------------------------------

``` r
    if (!requireNamespace('BiocManager', quietly = TRUE))
        install.packages('BiocManager')
        BiocManager::install('PCAtools')
```

Note: to install development version:

``` r
    devtools::install_github('kevinblighe/PCAtools')
```

2. Load the package into R session
----------------------------------

``` r
    library(PCAtools)
```

Quick start
===========

For this vignette, we will load breast cancer gene expression data with recurrence free survival (RFS) from [Gene Expression Profiling in Breast Cancer: Understanding the Molecular Basis of Histologic Grade To Improve Prognosis](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE2990).

First, let's read in and prepare the data:

``` r
  library(Biobase)
  library(GEOquery)

  # load series and platform data from GEO
  gset <- getGEO('GSE2990', GSEMatrix =TRUE, getGPL=FALSE)

  x <- exprs(gset[[1]])

  # remove Affymetrix control probes
  x <- x[-grep('^AFFX', rownames(x)),]

  # extract information of interest from the phenotype data (pdata)
  idx <- which(colnames(pData(gset[[1]])) %in%
    c('age:ch1', 'distant rfs:ch1', 'er:ch1',
      'ggi:ch1', 'grade:ch1', 'size:ch1',
      'time rfs:ch1'))

  metadata <- data.frame(pData(gset[[1]])[,idx],
    row.names = rownames(pData(gset[[1]])))

  # tidy column names
  colnames(metadata) <- c('Age', 'Distant.RFS', 'ER', 'GGI', 'Grade',
    'Size', 'Time.RFS')

  # prepare certain phenotypes
  metadata$Age <- as.numeric(gsub('^KJ', NA, metadata$Age))
  metadata$Distant.RFS <- factor(metadata$Distant.RFS, levels=c(0,1))
  metadata$ER <- factor(gsub('\\?', NA, metadata$ER), levels=c(0,1))
  metadata$ER <- factor(ifelse(metadata$ER == 1, 'ER+', 'ER-'), levels = c('ER-', 'ER+'))
  metadata$GGI <- as.numeric(metadata$GGI)
  metadata$Grade <- factor(gsub('\\?', NA, metadata$Grade), levels=c(1,2,3))
  metadata$Grade <- gsub(1, 'Grade 1', gsub(2, 'Grade 2', gsub(3, 'Grade 3', metadata$Grade)))
  metadata$Grade <- factor(metadata$Grade, levels = c('Grade 1', 'Grade 2', 'Grade 3'))
  metadata$Size <- as.numeric(metadata$Size)
  metadata$Time.RFS <- as.numeric(gsub('^KJX|^KJ', NA, metadata$Time.RFS))

  # remove samples from the pdata that have any NA value
  discard <- apply(metadata, 1, function(x) any(is.na(x)))

  metadata <- metadata[!discard,]

  # filter the expression data to match the samples in our pdata
  x <- x[,which(colnames(x) %in% rownames(metadata))]

  # check that sample names match exactly between pdata and expression data 
  all(colnames(x) == rownames(metadata))
```

    ## [1] TRUE

Conduct principal components analysis (PCA)

``` r
  p <- pca(x, metadata = metadata, removeVar = 0.1)
```

A SCREE plot
------------

``` r
  screeplot(p)
```

![Figure 1: A SCREE plot to show the proportion of explained variance by PC](README_files/figure-markdown_github/ex1-1.png)

A bi-plot
---------

``` r
  biplot(p)
```

![Figure 2: A bi-plot of PC1 versus PC2](README_files/figure-markdown_github/ex2-1.png)

A pairs plot
------------

``` r
  pairsplot(p)
```

![Figure 3: A pairs plot, comparing PC1 - PC5 on a pairwise basis](README_files/figure-markdown_github/ex3-1.png)

A loadings plot
---------------

``` r
  plotloadings(p)
```

![Figure 4: Plot the component loadings and label genes most responsible for variation](README_files/figure-markdown_github/ex4-1.png)

An eigencor plot
----------------

``` r
  eigencorplot(p,
    metavars = c('Age','Distant.RFS','ER','GGI','Grade','Size','Time.RFS'))
```

![Figure 5: Correlate PCs to metadata variables](README_files/figure-markdown_github/ex5-1.png)

Advanced features
=================

All plots in PCAtools are highly configurable and should cover virtually all general usage requirements. The following sections take a look at some of these advanced features, and form a somewhat practical example of how one can use PCAtools to make a clinical interpretation of data.

Determine PCs accounting for 80% explained variation
----------------------------------------------------

By analysing the SCREE plot, we can easily gauge how many PCs are required to satisfy a certain threshold for expalined variation in a dataset.

Horizontal and vertical lines can be added to the plot via 'hline' and 'vline'. We can add a text label with geom\_text().

``` r
  library(ggplot2)

  screeplot(p,
    components = getComponents(p, 1:30),
    hline = 80, vline = 27) +
    geom_text(aes(20, 80, label = '80% explained variation', vjust = -1))
```

![Figure 6: Advanced SCREE plot illustrating number of PCs comprising at least 80% explained variation](README_files/figure-markdown_github/ex6-1.png)

So, at least 27 PCs contribute to &gt;80% explained variation.

Modify bi-plots
---------------

The bi-plot comparing PC1 versus PC2 is the most characteristic plot of PCA. However, PCA is much more than the bi-plot and much more than PC1 and PC2. This said, PC1 and PC2, by the very nature of PCA, are indeed usually the most important parts of PCA.

In a bi-plot, we can shade the points by different groups and add many more features.

### Colour by a factor from the metadata, add lines through center, and add legend

``` r
  biplot(p,
    colby = 'ER',
    hline = 0, vline = 0,
    legendPosition = 'right')
```

![Figure 7: adding lines and a legend to a bi-plot](README_files/figure-markdown_github/ex7-1.png)

### Supply custom colours, add more lines, and increase legend size

``` r
  biplot(p,
    colby = 'ER', colkey = c('ER+'='forestgreen', 'ER-'='purple'),
    hline = 0, vline = c(-25, 0, 25),
    legendPosition = 'top', legendLabSize = 16, legendIconSize = 8.0)
```

![Figure 8: supplying custom colours to a bi-plot](README_files/figure-markdown_github/ex8-1.png)

### Change shape based on tumour grade, remove connectors, and add titles

``` r
  biplot(p,
    colby = 'ER', colkey = c('ER+'='forestgreen', 'ER-'='purple'),
    hline = 0, vline = c(-25, 0, 25),
    legendPosition = 'top', legendLabSize = 16, legendIconSize = 8.0,
    shape = 'Grade', shapekey = c('Grade 1'=15, 'Grade 2'=17, 'Grade 3'=8),
    drawConnectors = FALSE,
    title = 'PCA bi-plot',
    subtitle = 'PC1 versus PC2',
    caption = '27 PCs == 80%')
```

![Figure 9: supplying custom colours and shapes to a bi-plot, removing connectors, and modifying titles](README_files/figure-markdown_github/ex9-1.png)

### Remove labels, modify line types, remove gridlines, and increase point size

``` r
  biplot(p,
    lab = FALSE,
    colby = 'ER', colkey = c('ER+'='royalblue', 'ER-'='red3'),
    hline = 0, vline = c(-25, 0, 25),
    vlineType = c('dotdash', 'solid', 'dashed'),
    gridlines.major = FALSE, gridlines.minor = FALSE,
    pointSize = 5,
    legendPosition = 'left', legendLabSize = 16, legendIconSize = 8.0,
    shape = 'Grade', shapekey = c('Grade 1'=15, 'Grade 2'=17, 'Grade 3'=8),
    drawConnectors = FALSE,
    title = 'PCA bi-plot',
    subtitle = 'PC1 versus PC2',
    caption = '27 PCs == 80%')
```

![Figure 10: removing labels, modifying line types, removing gridlines, and increasing point size in a bi-plot](README_files/figure-markdown_github/ex10-1.png)

### Colour by a continuous variable (colour controlled by ggplot2 engine); plot other PCs

``` r
  biplot(p, x = 'PC10', y = 'PC50',
    lab = FALSE,
    colby = 'Age',
    hline = 0, vline = 0,
    hlineWidth = 1.0, vlineWidth = 1.0,
    gridlines.major = FALSE, gridlines.minor = TRUE,
    pointSize = 5,
    legendPosition = 'left', legendLabSize = 16, legendIconSize = 8.0,
    shape = 'Grade', shapekey = c('Grade 1'=15, 'Grade 2'=17, 'Grade 3'=8),
    drawConnectors = FALSE,
    title = 'PCA bi-plot',
    subtitle = 'PC10 versus PC50',
    caption = '27 PCs == 80%')
```

![Figure 11: colouring by a continuous variable and plotting other PCs in a bi-plot](README_files/figure-markdown_github/ex11-1.png)

Quickly explore potentially informative PCs via a pairs plot
------------------------------------------------------------

The pairs plot in PCA unfortunately suffers from a lack of use; however, for those who love exploring data and squeezing every last ounce of information out of data, a pairs plot provides for a relatively quick way to explore useful leads for other downstream analyses.

As the number of pairwise plots increases, however, space becomes limited. We can shut off titles and axis labeling to save space. Reducing point size and colouring by a variable of interest can additionally help us to rapidly skim over the data.

``` r
  pairsplot(p,
    components = getComponents(p, c(1:10)),
    triangle = TRUE, trianglelabSize = 12,
    hline = 0, vline = 0,
    pointSize = 0.4,
    gridlines.major = FALSE, gridlines.minor = FALSE,
    colby = 'Grade',
    title = 'Pairs plot', plotaxes = FALSE,
    margingaps = unit(c(-0.01, -0.01, -0.01, -0.01), 'cm'))
```

![Figure 12: Maximising available space in a pairs plot](README_files/figure-markdown_github/ex12-1.png)

We can arrange these in a way that makes better use of the screen space by setting 'triangle = FALSE'. In this case, we can further control the layout with the 'ncol' and 'nrow' parameters, although, the function will automatically determine these based on your input data.

``` r
  pairsplot(p,
    components = getComponents(p, c(1,5,10,15,20,25,30,35,70)),
    triangle = FALSE,
    hline = 0, vline = 0,
    pointSize = 0.8,
    gridlines.major = FALSE, gridlines.minor = FALSE,
    colby = 'ER',
    title = 'Pairs plot', plotaxes = TRUE,
    margingaps = unit(c(0.1, 0.1, 0.1, 0.1), 'cm'))
```

![Figure 13: arranging a pairs plot horizontally](README_files/figure-markdown_github/ex13-1.png)

Determine the variables that drive variation among each PC
----------------------------------------------------------

If, on the bi-plot or pairs plot, we encounter evidence that 1 or more PCs are segregating a factor of interest, we can explore further the genes that are driving these differences along each PC.

For each PC of interest, 'plotloadings' determines the variables falling within the top/bottom 5% of the loadings range, and then creates a final consensus list of these. These variables are then plotted.

The loadings plot, like all others, is highly configurable. To modify the cut-off for inclusion / exclusion of variables, we use 'rangeRetain', where 0.01 equates to the top/bottom 1% of the loadings range per PC. We can also add a title, subtitle, and caption, and alter the shape and colour scheme.

``` r
  plotloadings(p,
    rangeRetain = 0.01,
    labSize = 3.0,
    title = 'Loadings plot',
    subtitle = 'PC1, PC2, PC3, PC4, PC5',
    caption = 'Top 1% variables',
    shape = 24,
    col = c('limegreen', 'black', 'red3'),
    drawConnectors = TRUE)
```

![Figure 14: modifying cut-off for labeling in a loadings plot](README_files/figure-markdown_github/ex14-1.png)

We can check the genes to which these relate by using biomaRt:

``` r
  library(biomaRt)

  mart <- useMart('ENSEMBL_MART_ENSEMBL')
  mart <- useDataset('hsapiens_gene_ensembl', mart)

  getBM(mart = mart,
    attributes = c('affy_hg_u133a', 'ensembl_gene_id',
      'gene_biotype', 'external_gene_name'),
    filter = 'affy_hg_u133a',
    values = c('215281_x_at', '214464_at', '211122_s_at', '205225_at',
      '202037_s_at', '204540_at', '215176_x_at', '205044_at', '208650_s_at',
      '205380_at'),
    uniqueRows = TRUE)
```

    ##    affy_hg_u133a ensembl_gene_id                       gene_biotype
    ## 1    208650_s_at ENSG00000185275               processed_pseudogene
    ## 2    208650_s_at ENSG00000272398                     protein_coding
    ## 3      205225_at ENSG00000091831                     protein_coding
    ## 4    208650_s_at ENSG00000261333               processed_pseudogene
    ## 5      205044_at ENSG00000094755                     protein_coding
    ## 6    215176_x_at ENSG00000242371                          IG_V_gene
    ## 7    215176_x_at ENSG00000251546                          IG_V_gene
    ## 8    202037_s_at ENSG00000104332                     protein_coding
    ## 9    211122_s_at ENSG00000169248                     protein_coding
    ## 10   215281_x_at ENSG00000143442                     protein_coding
    ## 11     205380_at ENSG00000215859 transcribed_unprocessed_pseudogene
    ## 12     214464_at ENSG00000143776                     protein_coding
    ## 13     205380_at ENSG00000174827                     protein_coding
    ## 14     204540_at ENSG00000101210                     protein_coding
    ## 15   215176_x_at ENSG00000282120                          IG_V_gene
    ##    external_gene_name
    ## 1              CD24P4
    ## 2                CD24
    ## 3                ESR1
    ## 4              CD24P2
    ## 5               GABRP
    ## 6            IGKV1-39
    ## 7           IGKV1D-39
    ## 8               SFRP1
    ## 9              CXCL11
    ## 10               POGZ
    ## 11            PDZK1P1
    ## 12           CDC42BPA
    ## 13              PDZK1
    ## 14             EEF1A2
    ## 15           IGKV1-39

At least one interesting finding is 205225\_at (*ESR1*), which is by far the gene most responsible for variation along PC2. The previous bi-plots showed that this PC also segregated ER+ from ER- patients. The other results could be explored.

With the loadings plot, in addition, we can instead plot absolute values and modify the point sizes to be proportional to the loadings. We can also switch off the line connectors and plot the loadings for any PCs

``` r
  plotloadings(p,
    components = getComponents(p, c(1,5,10,15,20,25,30,35,70)),
    rangeRetain = 0.1,
    labSize = 3.0,
    absolute = TRUE,
    title = 'Loadings plot',
    subtitle = 'Misc PCs',
    caption = 'Top 10% variables',
    shape = 23, shapeSizeRange = c(1, 16),
    col = c('white', 'pink'),
    drawConnectors = FALSE)
```

![Figure 15: plotting absolute component loadings](README_files/figure-markdown_github/ex15-1.png)

Correlate the principal components back to the clinical data
------------------------------------------------------------

Further exploration of the PCs can come through correlations with clinical data. This is also a mostly untapped resource in the era of 'big data' and can help to guide an analysis down a particular path (or not!).

We may wish, for example, to correlate all PCs that account for 80% variation in our dataset and then explore further the PCs that have statistically significant correlations.

'eigencorplot' is built upon another function by the *PCAtools* developers, namely [CorLevelPlot](https://github.com/kevinblighe/CorLevelPlot). Further examples can be found there.

``` r
  eigencorplot(p,
    components = getComponents(p, 1:27),
    metavars = c('Age','Distant.RFS','ER','GGI','Grade','Size','Time.RFS'),
    col = c('darkblue', 'blue2', 'black', 'red2', 'darkred'),
    cexCorval = 0.7,
    colCorval = 'white',
    fontCorval = 2,
    posLab = 'bottomleft',
    rotLabX = 45,
    posColKey = 'top',
    cexLabColKey = 1.5,
    scale = TRUE,
    main = 'PC1-27 clinical correlations',
    colFrame = 'white',
    plotRsquared = FALSE)
```

![Figure 16: correlating PCs that account for at least 80% variation to clinical variables](README_files/figure-markdown_github/ex16-1.png)

We can also supply different cut-offs for statistical significance, plot R-squared values, and specify correlation method:

``` r
  eigencorplot(p,
    components = getComponents(p, 1:10),
    metavars = c('Age','Distant.RFS','ER','GGI','Grade','Size','Time.RFS'),
    col = c('white', 'cornsilk1', 'gold', 'forestgreen', 'darkgreen'),
    cexCorval = 1.2,
    fontCorval = 2,
    posLab = 'all',
    rotLabX = 45,
    scale = TRUE,
    main = bquote(Principal ~ component ~ Pearson ~ r^2 ~ clinical ~ correlates),
    plotRsquared = TRUE,
    corFUN = 'pearson',
    corUSE = 'pairwise.complete.obs',
    signifSymbols = c('****', '***', '**', '*', ''),
    signifCutpoints = c(0, 0.0001, 0.001, 0.01, 0.05, 1))
```

![Figure 17: modifying cut-offs and symbols for statistical significance in eigencorplot](README_files/figure-markdown_github/ex17-1.png)

Clearly, PC2 is coming across as the most interesting PC in this experiment, with highly statistically significant correlation (p&lt;0.0001) to ER status, tumour grade, and GGI (genomic Grade Index), an indicator of response. It comes as no surprise that the gene driving most variationn along PC2 is *ESR1*, identified from our loadings plot.

This information is, of course, not new, but shows how PCA is much more than just a bi-plot used to identify outliers!

Plot the entire project on a single panel
-----------------------------------------

``` r
  pscree <- screeplot(p, components = getComponents(p, 1:30),
    hline = 80, vline = 27, axisLabSize = 10, returnPlot = FALSE) +
    geom_text(aes(20, 80, label = '80% explained variation', vjust = -1))

  ppairs <- pairsplot(p, components = getComponents(p, c(1:3)),
    triangle = TRUE, trianglelabSize = 12,
    hline = 0, vline = 0,
    pointSize = 0.8, gridlines.major = FALSE, gridlines.minor = FALSE,
    colby = 'Grade',
    title = '', titleLabSize = 16, plotaxes = FALSE,
    margingaps = unit(c(0.01, 0.01, 0.01, 0.01), 'cm'),
    returnPlot = FALSE)

  pbiplot <- biplot(p, lab = FALSE,
    colby = 'ER', colkey = c('ER+'='royalblue', 'ER-'='red3'),
    hline = 0, vline = c(-25, 0, 25), vlineType = c('dotdash', 'solid', 'dashed'),
    gridlines.major = FALSE, gridlines.minor = FALSE,
    pointSize = 2, axisLabSize = 12,
    legendPosition = 'left', legendLabSize = 10, legendIconSize = 3.0,
    shape = 'Grade', shapekey = c('Grade 1'=15, 'Grade 2'=17, 'Grade 3'=8),
    drawConnectors = FALSE,
    title = 'PCA bi-plot', subtitle = 'PC1 versus PC2',
      caption = '27 PCs == 80%',
    returnPlot = FALSE)

  ploadings <- plotloadings(p, rangeRetain = 0.01, labSize = 2.5,
    title = 'Loadings plot', axisLabSize = 12,
    subtitle = 'PC1, PC2, PC3, PC4, PC5',
    caption = 'Top 1% variables',
    shape = 24, shapeSizeRange = c(4, 4),
    col = c('limegreen', 'black', 'red3'),
    legendPosition = 'none',
    drawConnectors = FALSE,
    returnPlot = FALSE)

  peigencor <- eigencorplot(p,
    components = getComponents(p, 1:10),
    metavars = c('Age','Distant.RFS','ER','GGI','Grade','Size','Time.RFS'),
    #col = c('royalblue', '', 'gold', 'forestgreen', 'darkgreen'),
    cexCorval = 0.6,
    fontCorval = 2,
    posLab = 'all', 
    rotLabX = 45,
    scale = TRUE,
    main = "PC clinical correlates",
    cexMain = 1.5,
    plotRsquared = FALSE,
    corFUN = 'pearson',
    corUSE = 'pairwise.complete.obs',
    signifSymbols = c('****', '***', '**', '*', ''),
    signifCutpoints = c(0, 0.0001, 0.001, 0.01, 0.05, 1),
    returnPlot = FALSE)

    library(cowplot)
    library(ggplotify)

    top_row <- plot_grid(pscree, ppairs, pbiplot,
      ncol = 3,
      labels = c('A', 'B  Pairs plot', 'C'),
      label_fontfamily = 'serif',
      label_fontface = 'bold',
      label_size = 22,
      align = 'h',
      rel_widths = c(1.05, 0.9, 1.05))

    bottom_row <- plot_grid(ploadings,
      as.grob(peigencor),
      ncol = 2,
      labels = c('D', 'E'),
      label_fontfamily = 'serif',
      label_fontface = 'bold',
      label_size = 22,
      align = 'h',
      rel_widths = c(1.5, 1.5))

    plot_grid(top_row, bottom_row, ncol = 1, rel_heights = c(1.0, 1.0))
```

![Figure 18: a merged panel of all PCAtools plots](README_files/figure-markdown_github/ex18-1.png)

Acknowledgments
===============

The development of *PCAtools* has benefited from contributions and suggestions from:

Krushna Chandra Murmu

Session info
============

``` r
sessionInfo()
```

    ## R version 3.6.0 (2019-04-26)
    ## Platform: x86_64-pc-linux-gnu (64-bit)
    ## Running under: Ubuntu 16.04.6 LTS
    ## 
    ## Matrix products: default
    ## BLAS:   /usr/lib/atlas-base/atlas/libblas.so.3.0
    ## LAPACK: /usr/lib/atlas-base/atlas/liblapack.so.3.0
    ## 
    ## locale:
    ##  [1] LC_CTYPE=pt_BR.UTF-8       LC_NUMERIC=C              
    ##  [3] LC_TIME=en_GB.UTF-8        LC_COLLATE=pt_BR.UTF-8    
    ##  [5] LC_MONETARY=en_GB.UTF-8    LC_MESSAGES=pt_BR.UTF-8   
    ##  [7] LC_PAPER=en_GB.UTF-8       LC_NAME=C                 
    ##  [9] LC_ADDRESS=C               LC_TELEPHONE=C            
    ## [11] LC_MEASUREMENT=en_GB.UTF-8 LC_IDENTIFICATION=C       
    ## 
    ## attached base packages:
    ## [1] parallel  stats     graphics  grDevices utils     datasets  methods  
    ## [8] base     
    ## 
    ## other attached packages:
    ##  [1] ggplotify_0.0.3     biomaRt_2.39.4      GEOquery_2.51.6    
    ##  [4] Biobase_2.43.1      BiocGenerics_0.29.2 PCAtools_0.99.13   
    ##  [7] cowplot_0.9.4       lattice_0.20-38     reshape2_1.4.3     
    ## [10] ggrepel_0.8.0       ggplot2_3.1.1       knitr_1.22         
    ## 
    ## loaded via a namespace (and not attached):
    ##  [1] progress_1.2.0       tidyselect_0.2.5     xfun_0.6            
    ##  [4] purrr_0.3.2          colorspace_1.4-1     htmltools_0.3.6     
    ##  [7] stats4_3.6.0         yaml_2.2.0           gridGraphics_0.3-0  
    ## [10] blob_1.1.1           XML_3.98-1.19        rlang_0.3.4         
    ## [13] pillar_1.3.1         glue_1.3.1           withr_2.1.2         
    ## [16] DBI_1.0.0            bit64_0.9-7          rvcheck_0.1.3       
    ## [19] plyr_1.8.4           stringr_1.4.0        munsell_0.5.0       
    ## [22] gtable_0.3.0         memoise_1.1.0        evaluate_0.13       
    ## [25] labeling_0.3         IRanges_2.17.5       curl_3.3            
    ## [28] AnnotationDbi_1.45.1 highr_0.8            Rcpp_1.0.1          
    ## [31] readr_1.3.1          scales_1.0.0         limma_3.39.19       
    ## [34] S4Vectors_0.21.24    bit_1.1-14           hms_0.4.2           
    ## [37] digest_0.6.18        stringi_1.4.3        dplyr_0.8.0.1       
    ## [40] grid_3.6.0           tools_3.6.0          bitops_1.0-6        
    ## [43] magrittr_1.5         lazyeval_0.2.2       RCurl_1.95-4.12     
    ## [46] tibble_2.1.1         RSQLite_2.1.1        crayon_1.3.4        
    ## [49] tidyr_0.8.3          pkgconfig_2.0.2      prettyunits_1.0.2   
    ## [52] xml2_1.2.0           httr_1.4.0           assertthat_0.2.1    
    ## [55] rmarkdown_1.12       R6_2.4.0             compiler_3.6.0

References
----------

Blighe (2013)

Blighe and Lewis (2018)

Blighe, Kevin. 2013. “Haplotype classification using copy number variation and principal components analysis.” The Open Bioinformatics Journal 7:19-24.

Blighe, Kevin, and Myles Lewis. 2018. “PCAtools: everything Principal Components Analysis.” <https://github.com/kevinblighe>.
