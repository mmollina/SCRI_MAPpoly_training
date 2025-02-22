Introduction to MAPpoly 0.3.0
================
Marcelo Mollinari
2022-01-12

``` r
library(mappoly)
```

### Load file

``` r
file.name <- system.file("extdata/potato_example.csv", package = "mappoly")
dat <- read_geno_csv(file.in = file.name, ploidy = 4)
print(dat, detailed = T)
plot(dat)
```

### Data quality control

``` r
dat <- filter_individuals(dat)
dat <- filter_missing(dat, type = "marker", filter.thres = .05)
dat <- filter_missing(dat, type = "individual", filter.thres = .05)
seq.filt <- filter_segregation(dat, chisq.pval.thres = 0.05/dat$n.mrk)
seq.filt <- make_seq_mappoly(seq.filt)
seq.red  <- elim_redundant(seq.filt)
seq.init <- make_seq_mappoly(seq.red)
plot(seq.init)
```

### Genomic order

``` r
go <- get_genomic_order(input.seq = seq.init) ## get genomic order of the sequence
plot(go)
```

### Two-points for the whole dataset

``` r
ncores <- parallel::detectCores() - 1
tpt <- est_pairwise_rf2(seq.init, ncpus = ncores)
m <- rf_list_to_matrix(tpt) ## converts rec. frac. list into a matrix 
sgo <- make_seq_mappoly(go) ## creates a sequence of markers in the genome order
plot(m, ord = sgo, fact = 5) ## plots a rec. frac. matrix using the genome order, averaging neighbor cells in a 5 x 5 grid 
```

### Group

``` r
g <- group_mappoly(m, expected.groups = 12, comp.mat = TRUE)
plot(g)
g
```

### Ordering and Phasing

``` r
id <- as.numeric(colnames(g$seq.vs.grouped.snp)[-13])
MAPs <- vector("list", 12)
for(i in id){
  s <- make_seq_mappoly(g, i, genomic.info = 1) 
  tpt <- est_pairwise_rf(s, ncpus = ncores)
  gen.o <- get_genomic_order(s) ## Using genomic order
  s.gen <- make_seq_mappoly(gen.o)
  MAPs[[i]] <- est_rf_hmm_sequential(s.gen,twopt = tpt, 
                                     sub.map.size.diff.limit = 5, 
                                     extend.tail = 200)
  MAPs[[i]]$info$chrom <- rep(paste0("ST4.03ch", stringr::str_pad(i, 2, pad = "0")),
                               MAPs[[i]]$info$n.mrk)
}
plot_map_list(MAPs, horiz = FALSE, col = "ggstyle")
```

### Updating map distances

``` r
MAPs.up <- vector("list", 12)
for(i in 1:12)
  MAPs.up[[i]] <- est_full_hmm_with_global_error(MAPs[[i]], error = 0.05)
plot_map_list(MAPs.up, horiz = FALSE, col = "ggstyle")
```

### Plot map vs. genome

``` r
plot_genome_vs_map(MAPs.up, same.ch.lg = TRUE)
```

### Map summary and export map to CSV

``` r
summary_maps(MAPs.up)
export_map_list(MAPs.up, file = "output_map.csv")
```

### Calculate homolog probability profiles

``` r
geno.prob <- lapply(MAPs.up, calc_genoprob_error, error = 0.05)
h <- calc_homologprob(geno.prob)
plot(h, lg = 3, ind = 12)
plot(h, lg = c(1,3,5,7), ind = 12)
```

### Preferential pairing profiles

``` r
pp <- calc_prefpair_profiles(geno.prob)
plot(pp)
```
