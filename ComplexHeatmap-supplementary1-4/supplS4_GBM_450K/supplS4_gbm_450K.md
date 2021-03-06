
Supplementary S4. Visualize Methylation Profile with Complex Annotations
==============================================

**Author**: Zuguang Gu ( z.gu@dkfz.de )

**Date**: 2016-05-13

----------------------------------------



<style type="text/css">
h1 {
    line-height: 120%;
}
</style>

In this supplementary, Figure 1 in [Strum et al., 2012](http://dx.doi.org/10.1016/j.ccr.2012.08.024)
is re-implemented with some adjustments.

**Note that ~ 3GB of memory are required to run this example.**

To successfully run this example, **ComplexHeatmap** >= 1.10.1. is required.
The newest version can be obtained by:


```r
library(devtools)
install_github("jokergoo/Complexheatmap")
```

Load packages.


```r
library(ComplexHeatmap)
library(matrixStats)
library(circlize)
library(RColorBrewer)
library(GetoptLong)
library(GenomicRanges)
```

The methylation profiles have been measured by Illumina HumanMethylation450 BeadChip arrays.
First,  load probe data via the **IlluminaHumanMethylation450kanno.ilmn12.hg19** package.


```r
data("IlluminaHumanMethylation450kanno.ilmn12.hg19", 
	package = "IlluminaHumanMethylation450kanno.ilmn12.hg19")
probe = IlluminaHumanMethylation450kanno.ilmn12.hg19 # change to a short name
```

Methylation profiles can be download from [GEO database](http://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE36278).
The [**GEOquery** package](http://bioconductor.org/packages/release/bioc/html/GEOquery.html) is used to retrieve expression data from GEO.


```r
library(GEOquery)
gset = getGEO("GSE36278")
```



Adjust row names in the matrix to be the same as the probes. 


```r
mat = exprs(gset[[1]])
colnames(mat) = phenoData(gset[[1]])@data$title
mat = mat[rownames(probe@data$Locations), ]	
```

`probe` contains locations of probes and also information whether the CpG sites overlap
with SNPs. Here we remove probes that are on sex chromosomes and probes that overlap with SNPs.


```r
l = probe@data$Locations$chr %in% paste0("chr", 1:22) & 
	is.na(probe@data$SNPs.137CommonSingle$Probe_rs)
mat = mat[l, ]
```

Get subsets for locations of probes and the annotation to CpG Islands accordingly.


```r
cgi = probe@data$Islands.UCSC$Relation_to_Island[l]
loc = probe@data$Locations[l, ]
```

Separate the matrix into a matrix for tumor samples and a matrix for normal samples.
Also modify column names for the tumor samples to be consistent with the phenotype data 
which we will read later.


```r
mat1 = as.matrix(mat[, grep("GBM", colnames(mat))])   # tumor samples
mat2 = as.matrix(mat[, grep("CTRL", colnames(mat))])  # normal samples
colnames(mat1) = gsub("GBM", "dkfz", colnames(mat1))
```

Phenotype data is from [Sturm et al., 2012](http://dx.doi.org/10.1016/j.ccr.2012.08.024),
supplementary table S1.

The rows of phenotype data are adjusted to be the same as the columns of the methylation matrix.


```r
phenotype = read.table("450K_annotation.txt", header = TRUE, sep = "\t", row.names = 1, 
	check.names = FALSE, comment.char = "", stringsAsFactors = FALSE)
phenotype = phenotype[colnames(mat1), ]
```

Please note that we only use the 136 samples which are from DKFZ, while in [Sturm et al., 2012](http://dx.doi.org/10.1016/j.ccr.2012.08.024), additional 74 TCGA samples have been used.

Extract the top 8000 probes with most variable methylation in the tumor samples, and also subset other
information correspondingly.


```r
ind = order(rowVars(mat1, na.rm = TRUE), decreasing = TRUE)[1:8000]
m1 = mat1[ind, ]
m2 = mat2[ind, ]
cgi2 = cgi[ind]
cgi2 = ifelse(grepl("Shore", cgi2), "Shore", cgi2)
cgi2 = ifelse(grepl("Shelf", cgi2), "Shelf", cgi2)
loc = loc[ind, ]
```

For each probe, find the distance to the closest TSS. `pc_tx_tss.bed` contains positions
of TSS from protein coding genes.


```r
gr = GRanges(loc[, 1], ranges = IRanges(loc[, 2], loc[, 2]+1))
tss = read.table("pc_tx_tss.bed", stringsAsFactors = FALSE)
tss = GRanges(tss[[1]], ranges = IRanges(tss[, 2], tss[, 3]))

tss_dist = distanceToNearest(gr, tss)
tss_dist = tss_dist@elementMetadata$distance
```

Because there are a few `NA` in the matrix (`sum(is.na(m1))/length(m1) = 0.0011967`) 
which will break the `cor()` function, we replace `NA` to the intermediate methylation (0.5).
Note that although **ComplexHeatmap** allows `NA` in the matrix, removal of `NA` will speed up the clustering.


```r
m1[is.na(m1)] = 0.5
m2[is.na(m2)] = 0.5
```

The following annotations will be added to the columns of the methylation matrix:

1. age
2. subtype classification by DKFZ
3. subtype classification by TCGA
4. subtype classification by TCGA, based on expression profile
5. IDH1 mutation
6. H3F3A mutation
7. TP53 mutation
8. chr7 gain
9. chr10 loss
10. CDKN2A deletion
11. EGFR amplification
12. PDGFRA amplification

In following code we define the column annotation in the `ha` variable. Also we customize colors, legends and height of the annotations.


```r
ha = HeatmapAnnotation(age = anno_points(phenotype[[13]], gp = gpar(col = ifelse(phenotype[[13]] > 20, "black", "red")), axis = TRUE),
	dkfz_cluster = phenotype[[1]],
	tcga_cluster = phenotype[[2]],
	tcga_expr = phenotype[[3]],
	IDH1 = phenotype[[5]],
	H3F3A = phenotype[[4]],
	TP53 = phenotype[[6]],
	chr7_gain = phenotype[[7]],
	chr10_loss = phenotype[[8]],
	CDKN2A_del = phenotype[[9]],
	EGFR_amp = phenotype[[10]],
	PDGFRA_amp = phenotype[[11]],
	col = list(dkfz_cluster = structure(names = c("IDH", "K27", "G34", "RTK I PDGFRA", "Mesenchymal", "RTK II Classic"), brewer.pal(6, "Set1")),
		tcga_cluster = structure(names = c("G-CIMP+", "Cluster #2", "Cluster #3"), brewer.pal(3, "Set1")),
		tcga_expr = structure(names = c("Proneural", "Classical", "Mesenchymal"), c("#377EB8", "#FFFF33", "#FF7F00")),
		IDH1 = structure(names = c("MUT", "WT"),c("black", "white")),
		H3F3A = structure(names = c("MUT", "WT", "G34R", "G34V", "K27M"), c("black", "white", "#4DAF4A", "#4DAF4A", "#377EB8")),
		TP53 = structure(names = c("MUT", "WT"), c("black", "white")),
		chr7_gain = structure(names = c(0, 1), c("white", "#E41A1C")),
		chr10_loss = structure(names = c(0, 1), c("white", "#377EB8")),
		CDKN2A_del = structure(names = c(0, 1), c("white", "#377EB8")),
		EGFR_amp = structure(names = c(0, 1), c("white", "#E41A1C")),
		PDGFRA_amp = structure(names = c(0, 1), c("white", "#E41A1C"))),
	na_col = "grey",
	show_legend = c(TRUE, TRUE, TRUE, FALSE, TRUE, FALSE, FALSE, FALSE, FALSE, FALSE, FALSE),
	annotation_height = unit(c(30, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5), "mm"),
	annotation_legend_param = list(
		dkfz_cluster = list(title = "DKFZ Methylation"),
		tcga_cluster = list(title = "TCGA Methylation"),
		tcga_expr = list(title = "TCGA Expression"),
		H3F3A = list(title = "Mutations"))
)
```

In the final plot, there are four heatmaps added. From left to right, there are

1. heatmap for methylation in tumor samples
2. methylation in normal samples
3. distance to nearest TSS
4. CpG Island (CGI) annotation.

The heatmaps are split by rows according to CGI annotations. 

After the heatmaps are plotted, additional graphics such as labels for annotations are added by `decorate_*()` functions.


```r
col_fun = colorRamp2(c(0, 0.5, 1), c("#377EB8", "white", "#E41A1C"))
ht_list = Heatmap(m1, col = col_fun, name = "Methylation",
	clustering_distance_rows = "euclidean", row_dend_reorder = TRUE,
	clustering_distance_columns = "spearman", column_dend_reorder = TRUE,
	show_row_dend = FALSE, show_column_dend = FALSE,
	show_row_names = FALSE, show_column_names = FALSE,
	bottom_annotation = ha, column_title = qq("GBM samples (n = @{ncol(m1)})"),
	split = factor(cgi2, levels = c("Island", "Shore", "Shelf", "OpenSea")), 
	row_title_gp = gpar(col = "#FFFFFF00")) + 
Heatmap(m2, col = col_fun, show_row_names = FALSE, show_column_names = FALSE, 
	show_column_dend = FALSE, column_title = "Controls",
	show_heatmap_legend = FALSE, width = unit(1, "cm")) +
Heatmap(tss_dist, name = "tss_dist", col = colorRamp2(c(0, 2e5), c("white", "black")), 
	show_row_names = FALSE, show_column_names = FALSE, width = unit(5, "mm"),
	heatmap_legend_param = list(at = c(0, 1e5, 2e5), labels = c("0kb", "100kb", "200kb"))) + 
Heatmap(cgi2, name = "CGI", show_row_names = FALSE, show_column_names = FALSE, width = unit(5, "mm"),
	col = structure(names = c("Island", "Shore", "Shelf", "OpenSea"), c("red", "blue", "green", "#CCCCCC")))
draw(ht_list, annotation_legend_side = "left", heatmap_legend_side = "left")

annotation_titles = c(dkfz_cluster = "DKFZ Methylation",
	tcga_cluster = "TCGA Methylation",
	tcga_expr = "TCGA Expression",
	IDH1 = "IDH1",
	H3F3A = "H3F3A",
	TP53 = "TP53",
	chr7_gain = "Chr7 gain",
	chr10_loss = "Chr10 loss",
	CDKN2A_del = "Chr10 loss",
	EGFR_amp = "EGFR amp",
	PDGFRA_amp = "PDGFRA amp")
for(an in names(annotation_titles)) {
	decorate_annotation(an, {
		grid.text(annotation_titles[an], unit(-2, "mm"), just = "right")
		grid.rect(gp = gpar(fill = NA, col = "black"))
	})
}
decorate_annotation("age", {
	grid.text("Age", unit(-12, "mm"), just = "right")
	grid.rect(gp = gpar(fill = NA, col = "black"))
	grid.lines(unit(c(0, 1), "npc"), unit(c(20, 20), "native"), gp = gpar(lty = 2))
})
decorate_annotation("IDH1", {
	grid.lines(unit(c(-40, 0), "mm"), unit(c(1, 1), "npc"))
})
decorate_annotation("chr7_gain", {
	grid.lines(unit(c(-40, 0), "mm"), unit(c(1, 1), "npc"))
})
decorate_heatmap_body("tss_dist", slice = 4, {
	grid.text("tss_dist", y = unit(-2, "mm"), rot = 90, just = "right")
})
decorate_heatmap_body("CGI", slice = 4, {
	grid.text("CGI_annotation", y = unit(-2, "mm"), rot = 90, just = "right")
})
decorate_heatmap_body("Methylation", slice = NULL, {
	grid.text(qq("DNA methylation probes (n = @{nrow(m1)})"), x = unit(-2, "mm"), 
		rot = 90, just = "bottom")
})
```

![plot of chunk unnamed-chunk-16](figure/unnamed-chunk-16-1.png)

## Session info


```r
sessionInfo()
```

```
## R version 3.2.3 (2015-12-10)
## Platform: x86_64-apple-darwin13.4.0 (64-bit)
## Running under: OS X 10.11.4 (El Capitan)
## 
## locale:
## [1] C/en_US.UTF-8/C/C/C/C
## 
## attached base packages:
##  [1] stats4    parallel  methods   grid      stats     graphics  grDevices
##  [8] utils     datasets  base     
## 
## other attached packages:
##  [1] RColorBrewer_1.1-2    circlize_0.3.7        GEOquery_2.34.0      
##  [4] Biobase_2.28.0        GenomicRanges_1.20.8  GenomeInfoDb_1.4.3   
##  [7] IRanges_2.2.9         S4Vectors_0.6.6       BiocGenerics_0.14.0  
## [10] GetoptLong_0.1.3      hash_2.2.6            matrixStats_0.50.2   
## [13] ComplexHeatmap_1.10.1
## 
## loaded via a namespace (and not attached):
##  [1] knitr_1.13           whisker_0.3-2        XVector_0.8.0       
##  [4] magrittr_1.5         colorspace_1.2-6     rjson_0.2.15        
##  [7] stringr_1.0.0        tools_3.2.3          formatR_1.4         
## [10] bitops_1.0-6         GlobalOptions_0.0.10 RCurl_1.95-4.8      
## [13] dendextend_1.1.8     shape_1.4.2          evaluate_0.9        
## [16] stringi_1.0-1        XML_3.98-1.4
```
