# The Geopolitics of Repressions
This repository contains code for my bachelor thesis project titled The Geopolitics of Repressions. 
The final version of the manuscript is available [here](https://martin-kosiik.github.io/Geopolitics_of_Repressions.pdf).

## Instructions for Replication
To succesfully replicate the analysis, follow the steps below:

1. Downolad the repository by clicking on **Clone or download**  under the repository name. 
2. Create a folder named `memo_list` on the same level as the folders `LaTex`, `code`, etc.
3. Download the Memorial data from [here](https://github.com/MemorialInternational/memorial_data_FULL_DB/blob/master/data/lists.memo.ru-disk/lists.memo.ru-disk.zip) and unzip the content into `memo_list`. 
4. Download the Zhukov and Talibova (2018) dataset from [here](https://www.prio.org/utility/DownloadFile.ashx?id=8&type=replicationfile) and copy file `eventsClean_v1.RData` into `data` folder of your cloned `Geopolitics-of-Repressions` repository.
5. Open R project file `Geopolitics-of-Repressions.Rproj` and run the RMarkdown scripts in  `code` folder starting with `01_ethnicity_imputation_and_data_summary.Rmd` and ending with `11_effect_size_calculations.Rmd`. 
6. Additionally, if you want to reproduce the manuscript itself, set the file `Bachelor_or_Master_thesis.tex` as the main file and run pdfLaTeX compiler. 

If you have any troubles with replication or any other questions, email me at  [martin.kosiik@gmail.com](mailto:martin.kosiik@gmail.com).

## Abstract
  This thesis studies how geopolitical concerns influence attitudes of a state toward its ethnic minorities.
    Using  data digitized from archival sources on  more than 2 million individual arrests by the Soviet secret police, I apply difference-in-differences and synthetic control method to estimate how changing German-Soviet relations influenced repressions of Germans in the Soviet Union. 
   The results of both methods show that
   there was large and statistically significant increase in arrests of Germans following the German invasion into the Soviet Union in 1941.
   Furthermore, the impact of war  appears to be highly persistent since there is almost  no decline in the estimated effect on repressions for nearly 10 years after the end of the war.
