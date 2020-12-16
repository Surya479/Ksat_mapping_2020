Global prediction of soil saturated hydraulic conductivity using random
forest in a Covariate-based Geo Transfer Functions (CoGTF) framework
================
Surya Gupta, Peter Lehmann, Sara Bonetti, Andreas Papritz and Dani Or

  - [Selection of random grids for spatial
    cross-validation](#selection-of-random-grids-for-spatial-cross-validation)
  - [Selection of Covaraites](#selection-of-covaraites)
  - [Model fitting](#model-fitting)

Soil saturated hydraulic conductivity (Ksat) is one of the prominent
soil hydraulic properties used in the modeling of land surface
processes. Ksat is often derived using limited dataset and soil basic
properties likely soil texture, bulk density) by means pedotransfer
functions (PTFs). We propose here an integrated Predictive Soil Modeling
(PSM) framework where soil variables are combined with RS-based
covariates using the Random Forest method. We refer to this approach as
the “Covariate-based Geo Transfer Functions” (CoGTF). Here, the
objective of this report to show the methods used to develop the CoGTF
with R code and stepwise description.

SoilKsatDB [link](https://doi.org/10.5281/zenodo.3752721)

CoGTF global Ksat maps [link](https://doi.org/10.5281/zenodo.3934853)

To cite this maps please use:

Gupta, S., Hengl, T., Lehmann, P., Bonetti, S., Papritz, A. and Or, D.:
[Global prediction of soil saturated hydraulic conductivity using random
forest in a Covariate-based Geo Transfer Functions (CoGTF)
framework](https://www.essoar.org/doi/10.1002/essoar.10503663.1).
manuscript submitted to Journal of Advances in Modeling Earth Systems
(JAMES).

``` r
library(caret)
```

    ## Loading required package: lattice

    ## Loading required package: ggplot2

``` r
library(randomForest)
```

    ## randomForest 4.6-14

    ## Type rfNews() to see new features/changes/bug fixes.

    ## 
    ## Attaching package: 'randomForest'

    ## The following object is masked from 'package:ggplot2':
    ## 
    ##     margin

``` r
library(ranger)
```

    ## 
    ## Attaching package: 'ranger'

    ## The following object is masked from 'package:randomForest':
    ## 
    ##     importance

``` r
library(mlr)
```

    ## Loading required package: ParamHelpers

    ## 'mlr' is in maintenance mode since July 2019. Future development
    ## efforts will go into its successor 'mlr3' (<https://mlr3.mlr-org.com>).

    ## 
    ## Attaching package: 'mlr'

    ## The following object is masked from 'package:caret':
    ## 
    ##     train

``` r
library(tibble)
library(raster)
```

    ## Loading required package: sp

    ## 
    ## Attaching package: 'raster'

    ## The following object is masked from 'package:mlr':
    ## 
    ##     resample

    ## The following object is masked from 'package:ParamHelpers':
    ## 
    ##     getValues

``` r
library(sp)
library(rgdal)
```

    ## rgdal: version: 1.5-12, (SVN revision 1018)
    ## Geospatial Data Abstraction Library extensions to R successfully loaded
    ## Loaded GDAL runtime: GDAL 3.0.4, released 2020/01/28
    ## Path to GDAL shared files: C:/Users/guptasu.D/Documents/R/win-library/3.6/rgdal/gdal
    ## GDAL binary built with GEOS: TRUE 
    ## Loaded PROJ runtime: Rel. 6.3.1, February 10th, 2020, [PJ_VERSION: 631]
    ## Path to PROJ shared files: C:/Users/guptasu.D/Documents/R/win-library/3.6/rgdal/proj
    ## Linking to sp version:1.4-2
    ## To mute warnings of possible GDAL/OSR exportToProj4() degradation,
    ## use options("rgdal_show_exportToProj4_warnings"="none") before loading rgdal.

``` r
library(hexbin)
library(lattice)
library(RColorBrewer)
library(viridis)
```

    ## Loading required package: viridisLite

``` r
library(Metrics)
```

    ## 
    ## Attaching package: 'Metrics'

    ## The following objects are masked from 'package:caret':
    ## 
    ##     precision, recall

``` r
ksat_df<-read.csv("E:/Ksat_dataset_mapping.csv")

## Unique IDs for 5degrees by 5 degrees



source("E:/OpenLandMap/R/saveRDS_functions.R")
source("E:/OpenLandMap/R/LandGIS_functions.R")

## saveRDS_functions.R and LandGIS_functions.R available at https://github.com/Envirometrix/LandGISmaps/tree/477460d1d0099646c508f65e68769b9edf050ce8/functions

## 3D modeling (see Hengl, T., & MacMillan, R. A. (2019). Predictive soil mapping with R. Lulu. com.)

dfs <- hor2xyd(ksat_df, U="hzn_top", L="hzn_bot")


I.vars = make.names(unique(unlist(sapply(c("s.no","clm_", "dtm_", "lcv", "veg_", "olm_c", "olm_s", "olm_bd", "FID_Fish_", "DEPTH"), function(i){names(dfs)[grep(i, names(dfs))]}))))

t.vars = c("log_ksat")
sel.n <- c(t.vars,I.vars)
sel.r <- complete.cases(dfs[,sel.n])
PTF_temp2 <- dfs[sel.r,sel.n]
```

# Selection of random grids for spatial cross-validation

``` r
## we selected 3 sets

##Set1
set.seed(16)
chosen <- sample(unique(PTF_temp2$FID_Fish_n), 40)

ff<-subset(PTF_temp2, FID_Fish_n %in% chosen)

final<-PTF_temp2[!(PTF_temp2$FID_Fish_n %in% ff$FID_Fish_n),]

set.seed(10)
chosen <- sample(unique(final$FID_Fish_n), 28)

ff1<-subset(final, FID_Fish_n %in% chosen)

final1<-final[!(final$FID_Fish_n %in% ff1$FID_Fish_n),]


set.seed(16)
chosen <- sample(unique(final1$FID_Fish_n), 16)

ff2<-subset(final1, FID_Fish_n %in% chosen)

final2<-final1[!(final1$FID_Fish_n %in% ff2$FID_Fish_n),]


set.seed(17)
chosen <- sample(unique(final2$FID_Fish_n), 53)

ff3<-subset(final2, FID_Fish_n %in% chosen)

final3<-final2[!(final2$FID_Fish_n %in% ff3$FID_Fish_n),]

##Set2

set.seed(34)
chosen <- sample(unique(PTF_temp2$FID_Fish_n), 58)

ff<-subset(PTF_temp2, FID_Fish_n %in% chosen)

final<-PTF_temp2[!(PTF_temp2$FID_Fish_n %in% ff$FID_Fish_n),]

set.seed(39)
chosen <- sample(unique(final$FID_Fish_n), 29)

ff1<-subset(final, FID_Fish_n %in% chosen)

final1<-final[!(final$FID_Fish_n %in% ff1$FID_Fish_n),]


set.seed(57)
chosen <- sample(unique(final1$FID_Fish_n), 27)

ff2<-subset(final1, FID_Fish_n %in% chosen)

final2<-final1[!(final1$FID_Fish_n %in% ff2$FID_Fish_n),]


set.seed(71)
chosen <- sample(unique(final2$FID_Fish_n), 12)

ff3<-subset(final2, FID_Fish_n %in% chosen)

final3<-final2[!(final2$FID_Fish_n %in% ff3$FID_Fish_n),]

##Set3

set.seed(79)
chosen <- sample(unique(PTF_temp2$FID_Fish_n), 29)

ff<-subset(PTF_temp2, FID_Fish_n %in% chosen)

final<-PTF_temp2[!(PTF_temp2$FID_Fish_n %in% ff$FID_Fish_n),]

set.seed(85)
chosen <- sample(unique(final$FID_Fish_n), 36)

ff1<-subset(final, FID_Fish_n %in% chosen)

final1<-final[!(final$FID_Fish_n %in% ff1$FID_Fish_n),]


set.seed(100)
chosen <- sample(unique(final1$FID_Fish_n), 31)

ff2<-subset(final1, FID_Fish_n %in% chosen)

final2<-final1[!(final1$FID_Fish_n %in% ff2$FID_Fish_n),]


set.seed(117)
chosen <- sample(unique(final2$FID_Fish_n), 27)

ff3<-subset(final2, FID_Fish_n %in% chosen)

final3<-final2[!(final2$FID_Fish_n %in% ff3$FID_Fish_n),]



df1<-ff
df2<-ff1
df3<-ff2
df4<-ff3
df5<-final3



Train1<- rbind(ff, ff1, ff2, ff3)

Train2<- rbind (ff1, ff2, ff3,final3)

Train3<- rbind(ff2, ff3,final3, ff)

Train4<- rbind(ff3,final3, ff,ff1)

Train5<- rbind(final3, ff,ff1, ff2)
```

# Selection of Covaraites

``` r
grid <- list.files("E:/maps_tests/new_layers/layers_RS/" , pattern = "*.tif$")
All_cov <- raster::stack(paste0("E:/maps_tests/new_layers/layers_RS/", grid))

set.seed(2) 
fm.ksat <- as.formula(paste("log_ksat~ ",paste(names(All_cov), collapse = "+")))
fm.ksat
```

    ## log_ksat ~ clm_bioclim.var_chelsa.1_m_1km_s0..0cm_1979..2013_v1.0 + 
    ##     clm_bioclim.var_chelsa.12_m_1km_s0..0cm_1979..2013_v1.0 + 
    ##     clm_bioclim.var_chelsa.13_m_1km_s0..0cm_1979..2013_v1.0 + 
    ##     clm_bioclim.var_chelsa.14_m_1km_s0..0cm_1979..2013_v1.0 + 
    ##     clm_bioclim.var_chelsa.4_m_1km_s0..0cm_1979..2013_v1.0 + 
    ##     clm_bioclim.var_chelsa.5_m_1km_s0..0cm_1979..2013_v1.0 + 
    ##     clm_bioclim.var_chelsa.6_m_1km_s0..0cm_1979..2013_v1.0 + 
    ##     clm_cloud.fraction_earthenv.modis.annual_m_1km_s0..0cm_2000..2015_v1.0 + 
    ##     clm_diffuse.irradiation_solar.atlas.kwhm2.100_m_1km_s0..0cm_2016_v1 + 
    ##     clm_direct.irradiation_solar.atlas.kwhm2.10_m_1km_s0..0cm_2016_v1 + 
    ##     clm_lst_mod11a2.annual.day_m_1km_s0..0cm_2000..2017_v1.0 + 
    ##     clm_lst_mod11a2.annual.day_sd_1km_s0..0cm_2000..2017_v1.0 + 
    ##     clm_precipitation_sm2rain.annual_m_1km_s0..0cm_2007..2018_v0.2 + 
    ##     DEPTH + dtm_aspect.cosine_merit.dem_m_250m_s0..0cm_2018_v1.0 + 
    ##     dtm_elevation_merit.dem_m_250m_s0..0cm_2017_v1.0 + dtm_lithology_usgs.ecotapestry.acid.plutonics_p_250m_s0..0cm_2014_v1.0 + 
    ##     dtm_slope_merit.dem_m_1km_s0..0cm_2017_v1.0 + dtm_twi_merit.dem_m_1km_s0..0cm_2017_v1.0 + 
    ##     lcv_landsat.nir_wri.forestwatch_m_250m_s0..0cm_2014_v1.0 + 
    ##     lcv_landsat.red_wri.forestwatch_m_250m_s0..0cm_2018_v1.2 + 
    ##     lcv_landsat.swir2_wri.forestwatch_m_250m_s0..0cm_2018_v1.2 + 
    ##     lcv_snow_probav.lc100_p_250m_s0..0cm_2017_v1.0 + lcv_wetlands.regularly.flooded_upmc.wtd_p_250m_b0..200cm_2010..2015_v1.0 + 
    ##     olm_bd + olm_clay + olm_sand + veg_fapar_proba.v.annual_d_250m_s0..0cm_2014..2017_v1.0

# Model fitting

``` r
set.seed(2) 
rm.ksat <- Train1[complete.cases(Train1[,all.vars(fm.ksat)]),]
m.ksat <- ranger(fm.ksat, rm.ksat, num.trees=200, mtry=6, quantreg = TRUE)
m.ksat
```

    ## Ranger result
    ## 
    ## Call:
    ##  ranger(fm.ksat, rm.ksat, num.trees = 200, mtry = 6, quantreg = TRUE) 
    ## 
    ## Type:                             Regression 
    ## Number of trees:                  200 
    ## Sample size:                      14330 
    ## Number of independent variables:  28 
    ## Mtry:                             6 
    ## Target node size:                 5 
    ## Variable importance mode:         none 
    ## Splitrule:                        variance 
    ## OOB prediction error (MSE):       0.5449819 
    ## R squared (OOB):                  0.593953

``` r
df5$prediction<- predict(m.ksat,df5)$predictions

RMSE(df5$prediction, df5$log_ksat)
```

    ## [1] 1.540183

``` r
## Ist_part is computed

rm.ksat1 <- Train2[complete.cases(Train2[,all.vars(fm.ksat)]),]
m.ksat1 <- ranger(fm.ksat, rm.ksat1, num.trees=200, mtry=6, quantreg = TRUE)
m.ksat1
```

    ## Ranger result
    ## 
    ## Call:
    ##  ranger(fm.ksat, rm.ksat1, num.trees = 200, mtry = 6, quantreg = TRUE) 
    ## 
    ## Type:                             Regression 
    ## Number of trees:                  200 
    ## Sample size:                      14340 
    ## Number of independent variables:  28 
    ## Mtry:                             6 
    ## Target node size:                 5 
    ## Variable importance mode:         none 
    ## Splitrule:                        variance 
    ## OOB prediction error (MSE):       0.5914379 
    ## R squared (OOB):                  0.6161231

``` r
df1$prediction<- predict(m.ksat1,df1)$predictions

RMSE(df1$prediction, df1$log_ksat)
```

    ## [1] 1.271529

``` r
## 2nd_part is computed
rm.ksat2 <- Train3[complete.cases(Train3[,all.vars(fm.ksat)]),]
m.ksat2 <- ranger(fm.ksat, rm.ksat2, num.trees=200, mtry=6, quantreg = TRUE)
m.ksat2
```

    ## Ranger result
    ## 
    ## Call:
    ##  ranger(fm.ksat, rm.ksat2, num.trees = 200, mtry = 6, quantreg = TRUE) 
    ## 
    ## Type:                             Regression 
    ## Number of trees:                  200 
    ## Sample size:                      14510 
    ## Number of independent variables:  28 
    ## Mtry:                             6 
    ## Target node size:                 5 
    ## Variable importance mode:         none 
    ## Splitrule:                        variance 
    ## OOB prediction error (MSE):       0.5091291 
    ## R squared (OOB):                  0.7000296

``` r
df2$prediction<- predict(m.ksat2,df2)$predictions

RMSE(df2$prediction, df2$log_ksat)
```

    ## [1] 0.9671728

``` r
## 3rd_part is computed

rm.ksat3 <- Train4[complete.cases(Train4[,all.vars(fm.ksat)]),]
m.ksat3 <- ranger(fm.ksat, rm.ksat3, num.trees=200, mtry=6, quantreg = TRUE)
m.ksat3
```

    ## Ranger result
    ## 
    ## Call:
    ##  ranger(fm.ksat, rm.ksat3, num.trees = 200, mtry = 6, quantreg = TRUE) 
    ## 
    ## Type:                             Regression 
    ## Number of trees:                  200 
    ## Sample size:                      14334 
    ## Number of independent variables:  28 
    ## Mtry:                             6 
    ## Target node size:                 5 
    ## Variable importance mode:         none 
    ## Splitrule:                        variance 
    ## OOB prediction error (MSE):       0.520718 
    ## R squared (OOB):                  0.673602

``` r
df3$prediction<- predict(m.ksat3,df3)$predictions

RMSE(df3$prediction, df3$log_ksat)
```

    ## [1] 1.205074

``` r
## 4th_part is computed
rm.ksat4 <- Train5[complete.cases(Train5[,all.vars(fm.ksat)]),]
m.ksat4 <- ranger(fm.ksat, rm.ksat4, num.trees=200, mtry=6, quantreg = TRUE)
m.ksat4
```

    ## Ranger result
    ## 
    ## Call:
    ##  ranger(fm.ksat, rm.ksat4, num.trees = 200, mtry = 6, quantreg = TRUE) 
    ## 
    ## Type:                             Regression 
    ## Number of trees:                  200 
    ## Sample size:                      14402 
    ## Number of independent variables:  28 
    ## Mtry:                             6 
    ## Target node size:                 5 
    ## Variable importance mode:         none 
    ## Splitrule:                        variance 
    ## OOB prediction error (MSE):       0.4789683 
    ## R squared (OOB):                  0.7237078

``` r
df4$prediction<- predict(m.ksat4,df4)$predictions

RMSE(df4$prediction, df4$log_ksat)
```

    ## [1] 1.009909

``` r
Final_data<- rbind(df1,df2,df3,df4,df5)

dd<- aggregate(Final_data[, 1:33], list(Final_data$s_no), mean)




RMSE(dd$prediction,dd$log_ksat)
```

    ## [1] 1.197835

``` r
ccc = DescTools::CCC(dd$prediction,dd$log_ksat, ci = "z-transform", conf.level = 0.95, na.rm=TRUE)$rho.c
ccc
```

    ##         est    lwr.ci    upr.ci
    ## 1 0.1384592 0.1271947 0.1496881

``` r
dd$log_ksat1<- 10^dd$log_ksat
dd$prediction1<- 10^dd$prediction

hexbinplot(log_ksat1~ prediction1, 
           panel = function(x, y, ...){
             panel.hexbinplot(x, y, ...)
             panel.loess(x, y,span = 2/3, col.line = "blue",type="l", lty=2, lwd = 4)
             panel.abline(c(0, 1),lwd = 2)
           },
           data = dd,xlab = "Predicted Ksat [cm/day]", ylab = "Measured Ksat [cm/day]",cex.axis = 4, aspect="1", xbins=30, colramp = function(n) {viridis (8,  alpha = 1, begin = 0, end = 1, direction = -1,option = "C")},xlim=c(0.1,10000), ylim=c(0.1,10000),
           scales=list(
             x = list(log = 10, equispaced.log = FALSE), 
             y = list(log = 10, equispaced.log = FALSE)
           ),
           font.lab= 6, cex.labels = 1.2,font.axis = 2,colorcut=c(0,0.01,0.03,0.07,0.15,0.25,0.5,0.75,1) )
```

![](README_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

``` r
##Fitting final model

#rm.ksat <- PTF_temp2[complete.cases(PTF_temp2[,all.vars(fm.ksat)]),]
#m.ksat <- ranger(fm.ksat, rm.ksat, num.trees=200, mtry=6, quantreg = TRUE)
#m.ksat


##Importance variable
#m.ksat <- randomForest(fm.ksat, rm.ksat, num.trees=200, mtry=6, quantreg = TRUE)

#varImpPlot(m.ksat, sort=TRUE, n.var=min(30, nrow(m.ksat$importance)))

## Then Produced the final CoGTF map

#p2 = predict(All_cov,m.ksat, progress='window',type = "response",fun = function(model, ...) predict(model, ...)$predictions)

#writeRaster(p2, "/home/step/data/OpenLandMap/Final_selected_covariates/New_raster_layer/Final_RF_0cm.tif")
```
