# Benchmark result of R jobs

## Code sources

In this benchmark, we use two R scripts to do the same data transformation:
- recuperation_contrats_mmo.R: Origin code provided by Dares
- sparlyr_pliu_test.R : update version

Below code is the content of  `sparlyr_pliu_test.R`

```R
# spark_install_tar("S:\Spark\spark-3.3.2-bin-hadoop3.tgz")
# Sys.setenv(SPARK_HOME="")

library(sparklyr)
library(dplyr)

conf <- spark_config()

conf["sparklyr.shell.driver-memory"] <- "64g"
conf["sparklyr.connect.cores.local"] <- 8

sc <- spark_connect(master="local", config = conf)

ODD_p <- spark_read_parquet(sc,
                            path= "C:/Users/Public/Documents/MiDAS_parquet/Vague 2/FNA/odd.parquet",
                            memory=FALSE)

head(ODD_p, n=5)

MMO_p <- spark_read_parquet(sc,
                            path= "C:/Users/Public/Documents/MiDAS_parquet/Vague 2/MMO/mmo.parquet",
                            memory=FALSE)

head(MMO_p, n=5)

start_time <- Sys.time()

droits_spark <- ODD_p %>% distinct(id_midas, KROD1) 

test1 <- MMO_p %>%
  right_join(droits_spark, by = c("id_midas")) %>%
  filter(DebutCTT != "" & FinCTT != "") %>%
  mutate(DebutCTT = as.Date(DebutCTT),FinCTT = as.Date(FinCTT)) %>%
  group_by(id_midas) %>%
  filter(DebutCTT == min(DebutCTT)) %>%
  ungroup() %>%
  mutate(duree_ctt = DATEDIFF(FinCTT, DebutCTT) + 1) %>%
  group_by(Nature) %>%
  summarise(duree_moy = mean(duree_ctt, na.rm = TRUE))
  
head(test1,n=5)

duration <- Sys.time() - start_time

duration

spark_disconnect(sc)

```


```R
##############
### ENTETE ###
##############

###Vider l'environnement de la session avant de commencer
rm(list=ls())
gc()


###Packages nécessaires
library(dplyr)
library(arrow)
#library(haven)
#library(lubridate)
#library(reshape2)
#library(duckdb)
#library(dbplyr)
#library(ggplot2)
library(sparklyr)


### Fichier de configuration Spark

conf <- spark_config()
conf$`sparklyr.cores.local` <- 4   # nombre de coeurs
conf$`sparklyr.shell.driver-memory` <- "80G"# mémoire allouée à l'unique exécuteur en local
conf$spark.storage.memoryFraction <- "0.05"    # fraction de la mémoire allouée au stockage
conf$spark.sql.shuffle.partitions <- "200"     
conf$spark.driver.maxResultSize <- 0           

### Lancement de la connexion à spark

sc <- spark_connect(master="local",config=conf)


###Chargement des données en spark
ODD_p <- spark_read_parquet(sc,
                            path= "C:/Users/Public/Documents/MiDAS_parquet/Vague 2/FNA/odd.parquet",
                            memory=FALSE)

MMO_p <- spark_read_parquet(sc,
                            path= "C:/Users/Public/Documents/MiDAS_parquet/Vague 2/MMO/mmo.parquet",
                            memory=FALSE)





####################################
### TEST A LANCER SIMULTANEMENET ###
####################################


time_spark_odd <- Sys.time()
droits_spark <- ODD_p %>%
  distinct(id_midas, KROD1) 

MMO_ODD <- MMO_p %>%
  right_join(droits_spark, by = c("id_midas")) %>%
  filter(DebutCTT != "" & FinCTT != "") %>%
  mutate(DebutCTT = as.Date(DebutCTT),FinCTT = as.Date(FinCTT)) %>%
  group_by(id_midas) %>%
  filter(DebutCTT == min(DebutCTT)) %>%
  ungroup() %>%
  mutate(duree_ctt = DATEDIFF(FinCTT, DebutCTT) + 1) %>%
  group_by(Nature) %>%
  summarise(duree_moy = mean(duree_ctt, na.rm = TRUE)) %>%
  ungroup() %>%
  collect()
time_s_final_odd <- Sys.time() - time_spark_odd
time_s_final_odd 

```

Below code is the content of  `recuperation_contrats_mmo.R`

## Benchmark result

| source_name                 | execution_time | data shuffle size |
|-----------------------------|----------------|-------------------|
| recuperation_contrats_mmo.R | 4.800279 min   | 4.4GB             |
| sparlyr_pliu_test.R         | 1.981864 min   | 123.7MB           |



## Use spark UI to benchmark a spark job

Use spark UI to view the job duration, memory usage and GC time.
http://localhost:4040

Unable to view DAG of the spark job for now. We will correct this ASAP

## Suggestions

### Don't use collect or df_collect unless you have no choice

The `collect()` and `df_collect()` function collect data from all worker node and send them to the driver node 
(a full shuffle), then convert `the Spark DataFrame into an R data frame or list`.

**Memory usage**

As they fetch the entire content of the Spark DataFrame and brings it into memory as an R data frame or list,
the memory consumption will be doubled. For large datasets, it can be problematic, because it may exceed the available
memory.

**Calculation time**
The full shuffle and dataframe conversion consume lots of time too.

A spark dataframe is similar to R dataframe. All the spark transformation and action can be applied on the dataframe.
So there is no need to do the conversion. 

The case you need to do the conversion
- Another function need to use the dataframe as input, but the function only accept the R dataframe as input.
- Data Visualization
