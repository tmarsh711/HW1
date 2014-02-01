BIOST 578 Winter 2014: HW#1
========================================================
width: 1440
height: 900
transition: none
font-family: 'Helvetica'
css: my_style.css
author: Tracey Marsh
date: January 31, 2014

<a rel="license" href="http://creativecommons.org/licenses/by-sa/3.0/deed.en_US"><img alt="Creative Commons License" style="border-width:0" src="http://i.creativecommons.org/l/by-sa/3.0/88x31.png" /></a><br /><tiny>This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-sa/3.0/deed.en_US">Creative Commons Attribution-ShareAlike 3.0 Unported License</tiny></a>.

Goal: 
========================
Find all HCV gene expression data (Illumina platform) submitted by an investigator at Yale. 

Obtain: title, accession number, GPL accession, manufacturer, and platform description. 

Set Up Common Environment
========================

```r
#helper function for later
trim <- function (x) gsub("^\\s+|\\s+$", "", x)

source("http://bioconductor.org/biocLite.R")
#assumed already done in environment
#Install all core packages and update all installed packages
#biocLite()
#getSQLiteFile()
#biocLite("GEOmetadb","GEOquery")
library(RSQLite)

#make a connection ***MODIFY PATH AS NEEDED***
geoCon <- dbConnect(SQLite(),'/Users/tracey/Documents/Stats/BIOST578/GEOmetadb.sqlite')
#dbListTables(geoCon)
```


Approach #1 - Use GEOmetabd package
===================================

```r
res1<-dbGetQuery(geoCon, 
  "SELECT gse.gse, gse.title, gpl.gpl, gpl.manufacturer, gpl.description
   FROM (gse JOIN gse_gpl ON gse.gse=gse_gpl.gse) j 
        JOIN gpl ON j.gpl=gpl.gpl
   WHERE gpl.manufacturer like '%illumina%'    
     AND gse.contact like '%yale%'  
     AND gpl.organism like '%homo sapien%'
     AND (gse.summary like '%HCV%' 
         OR gse.summary like '%hepatitis C%')        
   ;")
```


Approach #1 - Results
===================================

```r
#dim(res1)
names(res1)<-c('gse','title','gpl','manufacturer','description')
res1[,c('gse','gpl','manufacturer')]
```

```
       gse      gpl  manufacturer
1 GSE40223 GPL10558 Illumina Inc.
2 GSE40812 GPL10558 Illumina Inc.
```

```r
substring(trim(res1$title),1,100)
```

```
[1] "The blood transcriptional signature of chronic HCV [Illumina data]"                                  
[2] "Impaired TLR3-mediated immune responses from macrophages of patients chronically infected with Hepat"
```

```r
substring(trim(res1$description),1,100)
```

```
[1] "The HumanHT-12 v4 Expression BeadChip provides high throughput processing of 12 samples per BeadChip"
[2] "The HumanHT-12 v4 Expression BeadChip provides high throughput processing of 12 samples per BeadChip"
```


Approach #2 - Use data.table package
====================================
First set-up environment as if relational database were all data tables.


```r
library(data.table)
dbListTables(geoCon)
```

```
 [1] "gds"               "gds_subset"        "geoConvert"       
 [4] "geodb_column_desc" "gpl"               "gse"              
 [7] "gse_gpl"           "gse_gsm"           "gsm"              
[10] "metaInfo"          "sMatrix"          
```

```r
gse<-data.table(dbGetQuery(geoCon, "SELECT gse, title, summary, contact FROM gse;"))
setkey(gse,gse)
#one to many
gse_gpl<-data.table(dbGetQuery(geoCon, "SELECT * FROM gse_gpl;"))
setkeyv(gse_gpl,c('gpl','gse'))
gpl<-data.table(dbGetQuery(geoCon, "SELECT gpl, manufacturer, description FROM gpl;"))
setkey(gpl,gpl)
```


Approach #2 - Use data.table package
====================================

```r
res2<-merge(gpl[gse_gpl,nomatch=0][toupper(manufacturer) %like% 'ILLUMINA',],
            gse[(toupper(contact) %like% "YALE")&
                (toupper(summary) %like% 'HCV' | toupper(summary) %like% 'HEPATITIS C'),],
            by='gse',all=FALSE)[,c('contact','summary'):=c(NULL,NULL)]
```

    
Approach #2 - Results
===================================

```r
#dim(res2)
res2[,c('gse','gpl','manufacturer'),with=FALSE]
```

```
        gse      gpl  manufacturer
1: GSE40223 GPL10558 Illumina Inc.
2: GSE40812 GPL10558 Illumina Inc.
```

```r
res2[,substring(trim(title),1,100)]
```

```
[1] "The blood transcriptional signature of chronic HCV [Illumina data]"                                  
[2] "Impaired TLR3-mediated immune responses from macrophages of patients chronically infected with Hepat"
```

```r
res2[,substring(trim(description),1,100)]
```

```
[1] "The HumanHT-12 v4 Expression BeadChip provides high throughput processing of 12 samples per BeadChip"
[2] "The HumanHT-12 v4 Expression BeadChip provides high throughput processing of 12 samples per BeadChip"
```


Clean Up Common Overhead
========================

```r
#disconnect
dbDisconnect(geoCon)
```

```
[1] TRUE
```

