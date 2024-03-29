# CS598 Deep Learning for Healthcare Spring 2023 Paper Reproduction Project
##### Gilberto Ramirez and Jay Kakwani 
##### {ger6, kakwani2}@illinois.edu
##### Group ID: 27 | Paper ID: 181
---

## Original Paper

**Citation to the original paper**

Suresh, Harini, Gong, Jen J, Guttag, John V. Learning Tasks for Multitask Learning: Heterogenous Patient Populations in the ICU. In Proceedings of the 24th ACM SIGKDD International Conference on Knowledge Discovery & Data Mining (KDD '18). Association for Computing Machinery, New York, NY, USA, 802–810. https://doi.org/10.1145/3219819.3219930

**Repository Link**

https://github.com/mit-ddig/multitask-patients


## Data

This paper uses the publicly available [MIMIC-III database](https://www.nature.com/articles/sdata201635) which contains clinical data in a critical care setting. After reviewing the paper in detail, we decided to use [MIMIC-Extract](https://arxiv.org/abs/1907.08322), an open source pipeline by (Wang et al., 2020) for transforming the raw EHR data into usable Pandas dataframes containing hourly time series of vitals and laboratory measurements after performing unit conversion, outlier handling, and aggregation of semantically similar features.

There are three data files required -
``` 
all_hourly_data.h5 (an HDF File resulting from MIMIC-Extract pipeline)
code_status.csv (MIMIC-III concept table)
sapsii.csv (MIMIC-III concept table)
```
MIMIC-III Concept tables can be generated following instructions in this [GitHub link](https://github.com/MIT-LCP/mimic-code/tree/main/mimic-iii/concepts#generating-the-concepts-in-postgresql)

## Dependencies

You'll need a working Python environment to run the code.
The recommended way to set up your environment is through the
[Anaconda Python distribution](https://www.anaconda.com/download/) which
provides the `conda` package manager.
Anaconda can be installed in your user directory and does not interfere with
the system Python installation.
The required dependencies are specified in the file `environment.yml`.

We use `conda` virtual environments to manage the project dependencies in
isolation.
Thus, you can install our dependencies without causing conflicts with your
setup (even with different Python versions).

Run the following command in the repository folder (where `environment.yml`
is located) to create a separate environment and install all required
dependencies in it:

    conda env create


## Code

1. [mtl_patients.py](https://github.com/ger6-illini/dl4h-sp23-team27-project/blob/main/code/mtl_patients.py) has methods to pre-process data, generate summaries, discover cohorts and run mortality predictions.
2. [cs598-dl4h-team27-paper181.ipynb](https://github.com/ger6-illini/dl4h-sp23-team27-project/blob/main/notebooks/cs598-dl4h-team27-paper181.ipynb) notebook has steps and results for 24 hours and 48 hours initial period of ICU stay experiments.

## Method

In this paper, the authors propose a novel two-step pipeline to predict in-hospital mortality across patient populations with different characteristics. 

-    **Step 1. Discovering Patient Cohorts** by dividing patients into relevant non-overlapping cohorts in an unsupervised way using a long short-term memory (LSTM) autoencoder followed by a Gaussian Mixture Model (GMM).
-    **Step 2. Predicts in-hospital mortality** for each patient cohort identified in the previous step using an LSTM based multi-task learning model where every cohort is considered a different task.

### Step 1 Discovering Patient Cohorts : 

In order to identify meaningful patient cohorts, the paper proposes to process the raw patient data in such a way that the result is a 3D matrix of shape $(P \times T \times F)$ where $P$ represents the number of patients, $T$ the number of timesteps, and $F$ the number of features as depicted in the figure below (in blue) which is partially based on Figure 2 of the original paper. All numbers shown in the figure below correspond to a specific experiment published in the paper in which the observation window is limited to the first $24$ hours (cutoff period) after a patient goes into a careunit and there is a gap of $12$ hours (gap period) between the end of the observation window and the beginning of the prediction window where the prediction task is in-hospital mortality.

Preparation of the data to get the 3D (blue) matrix is performed by a function called `prepare_data()` inside the `mtl_patients.py` module. This preparation consists of the following transformations taken from the paper and the author's code reference implementation:
1. Calculation of the mortality flag (prediction label) and mortality time for every patient in the dataset using an *extended* definition of mortality: death, a note of "Do Not Resuscitate" (DNR), or a note of "Comfort Measures Only" (CMO). In case any of these conditions are met for a patient, the corresponding mortality label is set to *True* and the corresponding mortality time is considered as the earliest time of any of the three conditions. After reviewing in detail the author's code implementation it seems mortality is based on deathtime and a CMO note but not DNR. However, the calculation of the time of death is based on the earliest time of the three conditions.
2. Data used for the prediction is only limited to the first certain amount of hours after a patient goes into the ICU. This amount of hours is called inside the code "a cutoff period" (observation window) and defines the period of data used to train all models. In addition, there is another number of hours called inside the code "the gap period" which represents the time between the end of the observation window and the beginning of the prediction window to prevent label leakage. All patients that died under the *extended* definition before the cutoff period plus the gap period or stayed less than the cutoff period are excluded from the experiment as part of this step. Also, all patients under the age of 15 are excluded (this is already part of the exclusion criteria of the MIMIC-Extract pipeline).
3. There are 29 vitals/labs timeseries selected by the paper. Only data within the cutoff period for vitals/labs is kept and rest is removed. This will be used for the rest of the machine learning pipeline.
4. All vitals/labs values are converted to z-scores so they all have zero mean and unit standard deviation. Those z-scores are rounded to the closest integer and clipped between $-4$ and $4$ or set to $9$ in case of `NaN`. This allows to map every vital/lab measurement (a float) to one of ten possible values $[-4, -3, -2, -1, 0, 1, 2, 3, 4, 9]$, so they can be converted to dummy values. After dummifying the vitals/labs, column for the $9$ values (`NaN`) is removed, and the resulting matrix is sparse and containing either $0$s or $1$s.
5. Every patient is padded with rows of zeroes for those hours that are missed. For example, if a patient only has vitals/labs for the first ten hours and the cutoff period is 24, code adds fourteen hours (rows) with zeroes for that patient. In the end, the matrix will have a size of $P \times T \times F$ as expected by the subsequent models.
6. Finally, static data (gender, age, and ethnicity) is converted to integers representing categories and dummified. In case of age, there are four buckets established; $(10, 30), (30, 50), (50, 70), (70, \infty)$; while ethnicity is broken into five buckets (asian, white, hispanic, black, other).
7. Cohort assignments based on first careunit or Simplified Acute Physiology Score (SAPS) II score quartile is calculated for each patient and returned as well.

<img src="img/paper-181-fig-1.png" alt="Fig-1" title="Discover Patient Cohorts">

The `discover_cohorts()` function inside the `mtl_patients.py` module is the one implementing the pipeline shown in the figure above and then calling the `prepare_data()` function detailed previously as the first step. Once data has been processed, the function will break the data in training, validation, and test data sets in a $70\%/10\%/20\%$ proportion.

The training data is used to train an LSTM autoencoder. The main purpose of the LSTM autoencoder is to generate a fixed-length dense representation (embedding) of the sparse inputs trying to retain the most important parts of the inputs. The paper selected embeddings of size $50$ as the optimal dimension (hyperparameter). The purple box in the middle of the diagram above (a 2D matrix) represents the embeddings after the LSTM autoencoder learned the representation of the original 3D matrix of shape $(32537 \times 24 \times 232)$ where every row corresponds to a patient.

Once the embeddings are calculated, a Gaussian Mixture Model is applied using $3$ clusters (the value the authors considered optimal). The result are the three green boxes representing three cohorts discovered in an unsupervised way and grouping similar patients based on the three static and the 29 time-varying vitals/labs selected from the MIMIC-III database.


### Step 2 Predicting In-Hospital Mortality : 


As mentioned in the previous section, the paper uses a two-step pipeline to: 1) identify relevant patient cohorts, and 2) use those relevant cohorts as separate tasks in a multi-lask learning framework to predict in-hospital mortality. In this section, we will focus on the second step of the pipeline, i.e., use multi-task learning to make in-hospital mortality predictions for different patient cohorts.

The second step uses as input the result from the first step which is a series of 3D matrices, one per discovered cohort, of shape $(P \times T \times F)$ where $P$ represents the number of patients, $T$ the number of timesteps, and $F$ the number of features. As an example, the 24 hour experiment described by the authors in the paper and reproduced in the previous section resulted in three cohorts (clusters) called group 0, group 1, and group 2 where the shapes of the corresponding 3D matrices are:
* $14120 \times 24 \times 232$ for group 0,
* $10841 \times 24 \times 232$ for group 1, and
* $7752 \times 24 \times 232$ for group 2.

To convert these matrices into predictions, the paper proposes an LSTM for all model configurations including the baseline. In particular, the paper shows results from two specific models: a baseline model that is called *global* and using single-task learning and the multi-task learning model the authors claim as superior to the baseline.

A diagram of the baseline (*global*) model proposed by the authors is shown below. As it can be seen, this model consists of an LSTM layer of 16 cells using a RELU activation function followed by a *single* dense layer with a sigmoid activation function. The result of the dense (fully-connected) layer is an estimate of the probability of in-hospital mortality for a given patient. This baseline model is trained with all patient samples regardless the cohort, hence the name *global*, and used for per cohort predictions.

![Figure 2](img/paper-181-fig-2.png)

Moving to the second model and the one the authors claim it provides benefits against the baseline is the so called *multi-task learning model*. This model consists of an LSTM layer with same number of cells (16) as the baseline model, to ensure the comparison is fair, connected to as many dense layers as population subgroups (cohorts). Each of these cohorts is considered a *task* and authors propose training these models on multiple tasks simultaneously in contrast to the baseline model with just one dense layer. The benefit of this approach according to the authors is the ability to share knowledge learned from one task (cohort) to rest of tasks under the assumption that the subpopulations used are distinct enough with relation to the outcome learned (mortality) that such shared knowledge truly exists. A representation of the multi-task learning model is shown below:


![Figure 3](img/paper-181-fig-3.png)

For benchmarking purposes of the entire pipeline, the authors compared the results from running the pipeline using unsupervised cohort discovery (step one) against cohorts created using the first careunit the patient went into which can be considered an engineered feature. We will show those results in the next subsections.

The overall performance of this model is measured using both macro and micro metrics (section 4.3 in the paper) where:
* In *micro* metrics all predicted probabilities for all patients are treated as if they come from a single classifier: $\text{Metric}_\text{Micro} = \text{Metric}([\hat{y}_0, ..., \hat{y}_k], [y_0, ..., y_K])$.
* In *macro* metrics probabilities are evaluated on a *per cohort* basis, and then averaged: $Metric_{\text{Macro}} =\dfrac{1}{K} \displaystyle\sum_{k=0}^k Metric(\hat{y}_k,y_k)$.

Paper suggests that, although micro metrics are the ones typically chosen in the literature, evaluating performance on different subpopulations will benefit from macro metrics instead of micro metrics specially when there is class imbalance in every cohort. All results show macro and micro versions of the metrics for the aggregate performance of the models.

All results being used for comparison between models by the paper will use three metrics:
* AUC (Area Under the ROC Curve) for every cohort and, for the aggregate performance, macro and micro.
* PPV (Positive Predictive Value which is same as Precision) for every cohort and, for the aggregate performance, macro and micro. This PPV is calculated at a sensitivity of 80%, a value selected by the paper authors.
* Specificity for every cohort and, for the aggregate performance, macro and micro. This specificity is calculated at a sensitivity of 80%, a value selected by the paper authors.

All in-hospital mortality prediction tasks are implemented using the function `run_mortality_prediction_task()`. This function will call other functions to prepare the data, split the data in training/validation/test data sets, train the corresponding model, predict using the resulting model, and calculate the metrics of the model.

## Results


### Paper Reproduction Result

24 Hours and 48 hours Mortality Prediction experiment shows performance differences between multi-task and global models on specific cohorts.
Significant differences (p < 0.01) are shown in bold.

<table>
<tr><th> Paper Reproduction Results: 24 Hours and 48 Hours Mortality Predictions </th></tr>
<tr><td>    

|            |              |        || AUC    |            |              |  | PPV    |            |              | |Specificity |            |              |
|------------|--------------|--------|---|--------|------------|--------------|---|--------|------------|--------------|---|-------------|------------|--------------|
| Experiment |  Cohort type | Cohort |\| | Global | Multi-task |      p-value |\|  | Global | Multi-task |      p-value |\||      Global | Multi-task |      p-value |
|   24 hours |    Careunits |    CCU | | **0.865** |      0.857 |8.3 x 10<sup>-7</sup> ||  **0.206** |      0.195 | 1.03 x 10<sup>-3</sup> ||       0.785 |      0.778 | 1.48 x 10<sup>-1</sup> |
|            |              |   CSRU | | 0.889 |      **0.903** | 4.05 x 10<sup>-10</sup> ||  **0.098** |      0.093 | 5.46 x 10<sup>-2</sup> ||       0.846 |      0.848 | 3.11 x 10<sup>-1</sup> |
|            |              |   MICU | | 0.828 |      **0.833** | 5.07 x 10<sup>-9</sup> ||  0.227 |      **0.243** | 1.28 x 10<sup>-14</sup> ||       0.681 |      **0.709** | 8.44 x 10<sup>-15</sup> |
|            |              |   SICU | | **0.855** |      0.842 | 1.29 x 10<sup>-16</sup> ||  0.209 |      0.210 | 6.51 x 10<sup>-1</sup> ||      0.742 |      0.745 | 6.5 x 10<sup>-1</sup> |
|            |              |  TSICU | | 0.865 |      **0.871** | 2.5 x 10<sup>-7</sup> ||  **0.222** |      0.211 | 2.85 x 10<sup>-3</sup> ||       **0.797** |      0.788 | 4.07 x 10<sup>-3</sup> |
|            |              |  Macro | | 0.860 |      0.861 | 2.77 x 10<sup>-1</sup> ||  **0.192** |      0.190 | 6.46 x 10<sup>-2</sup> ||       0.770 |      0.774 | 3.59 x 10<sup>-1</sup> |
|            |              |  Micro | | 0.862 |      **0.864** | 8.84 x 10<sup>-4</sup> ||  0.202 |      **0.216** | 4.91 x 10<sup>-17</sup> ||       0.759 |      **0.778** | 5.0 x 10<sup>-17</sup> |
|            | Unsupervised |      0 | | **0.871** |      0.857 | 4.23 x 10<sup>-18</sup> ||  **0.194** |      0.176 | 5.4 x 10<sup>-10</sup> ||       **0.777** |      0.744 | 2.86 x 10<sup>-11</sup> |
|            |              |      1 | | 0.828 |      **0.832** | 3.08 x 10<sup>-8</sup> ||  0.217 |      **0.223** | 3.06 x 10<sup>-6</sup> ||       0.701 |      **0.713** | 2.02 x 10<sup>-6</sup> |
|            |              |      2 | | 0.896 |      **0.903** | 1.61 x 10<sup>-13</sup> ||  0.171 |      **0.193** | 1.02 x 10<sup>-13</sup> ||       0.807 |      **0.830** | 6.98 x 10<sup>-13</sup> |
|            |              |  Macro | |**0.865** |      0.864 | 2.61 x 10<sup>-3</sup> ||  0.194 |      **0.197** | 9.65 x 10<sup>-3</sup> ||       0.762 |      0.762 | 7.7 x 10<sup>-1</sup> |
|            |              |  Micro | | 0.865 |      0.865 | 4.94 x 10<sup>-1</sup> ||  0.204 |      **0.210** | 8.03 x 10<sup>-7</sup> ||       0.762 |      **0.770** | 1.74 x 10<sup>-6</sup> |
|   48 hours |    Careunits |    CCU | | 0.837 |      **0.850** | 2.3 x 10<sup>-9</sup> ||  0.170 |      0.172 | 1.47 x 10<sup>-1</sup> ||       0.738 |      **0.749** | 5.96 x 10<sup>-2</sup> |
|            |              |   CSRU | | **0.926** |      0.914 | 5.01 x 10<sup>-12</sup> ||  **0.099** |      0.092 | 3.04 x 10<sup>-2</sup> ||       **0.880** |      0.868 | 3.78 x 10<sup>-3</sup> |
|            |              |   MICU | | 0.824 |      **0.829** | 4.2 x 10<sup>-6</sup> || 0.188 |      **0.193** | 1.73 x 10<sup>-3</sup> ||       0.689 |      **0.702** | 5.14 x 10<sup>-4</sup> |
|            |              |   SICU | | 0.861 |     **0.872** | 5.79 x 10<sup>-11</sup> ||  0.211 |      **0.238** | 1.3 x 10<sup>-9</sup> ||       0.805 |      **0.828** | 4.12 x 10<sup>-7</sup> |
|            |              |  TSICU | | 0.797 |     **0.832** | 1.66 x 10<sup>-17</sup> ||  0.109 |      **0.148** | 4.18 x 10<sup>-18</sup> ||       0.651 |      **0.756** | 4.39 x 10<sup>-18</sup> |
|            |              |  Macro | | 0.849 |      **0.859** | 3.51 x 10<sup>-16</sup> ||  0.156 |      **0.169** | 4.95 x 10<sup>-11</sup> ||       0.753 |      **0.781** | 3.91 x 10<sup>-15</sup> |
|            |              |  Micro | | 0.852 |      **0.862** | 1.09 x 10<sup>-17</sup> ||  0.157 |      **0.175** | 5.37 x 10<sup>-18</sup> ||       0.740 |      **0.773** | 5.48 x 10<sup>-18</sup> |
|            | Unsupervised |      0 | | 0.847 |      **0.849** | 1.12 x 10<sup>-5</sup> ||  0.145 |      **0.157** | 6.36 x 10<sup>-16</sup> ||       0.730 |      **0.754** | 4.56 x 10<sup>-16</sup> |
|            |              |      1 | | 0.863 |      **0.871** | 8.71 x 10<sup>-15</sup> ||  0.192 |      **0.200** | 1.76 x 10<sup>-5</sup> ||       0.779 |      **0.792** | 2.22 x 10<sup>-6</sup> |
|            |              |  Macro | | 0.855 |      **0.860** | 1.04 x 10<sup>-15</sup> ||  0.169 |      **0.179** | 1.11 x 10<sup>-10</sup> ||       0.754 |     **0.773** | 2.53 x 10<sup>-14</sup> |
|            |              |  Micro | | 0.853 |      **0.857** | 1.41 x 10<sup>-15</sup> ||  0.158 |      **0.168** | 4.06 x 10<sup>-16</sup> ||       0.743 |      **0.760** | 3.45 x 10<sup>-16</sup> |

</td>                
</tr> 
</table>


### Results Comparison 

#### 1. Original Paper (Table 4) - 24 Hours Mortality Prediction 

<table>
<tr><th>Original Paper Result (24 Hours) </th><th>Reproduction Paper Result (24 Hours)</th></tr>
<tr><td>

|             |        | AUC       |            | PPV       |            | Specificity |            |
|-------------|--------|-----------|------------|-----------|------------|-------------|------------|
| Cohort type | Cohort | Global    | Multi-task | Global    | Multi-task | Global      | Multi-task |
| Careunits   | CCU    | **0.862** | 0.861      | **0.248** | 0.229      | **0.834**   | 0.819      |
|             | CSRU   | 0.849     | **0.867**  | 0.107     | **0.117**  | 0.893       | **0.898**  |
|             | MICU   | 0.814     | **0.832**  | 0.261     | **0.262**  | 0.764       | **0.766**  |
|             | SICU   | 0.839     | **0.855**  | 0.226     | **0.238**  | 0.781       | **0.796**  |
|             | TSICU  | 0.846     | **0.869**  | 0.183     | **0.192**  | 0.823       | **0.818**  |
|             | Macro  | 0.842     | **0.857**  | 0.205     | **0.208**  | 0.819       | 0.819      |
|             | Micro  | 0.852     | **0.866**  | 0.231     | **0.233**  | 0.817       | **0.821**  |
| Unsupervised| 0      | 0.803     | **0.819**  | 0.083     | **0.103**  | 0.732       | **0.786**  |
|             | 1      | 0.811     | **0.829**  | 0.120     | **0.126**  | 0.916       | 0.915      |
|             | 2      | 0.814     | **0.821**  | 0.276     | **0.288**  | 0.734       | **0.742**  |
|             | Macro  | 0.809     | **0.823**  | 0.159     | **0.172**  | 0.794       | **0.814**  |
|             | Micro  | 0.852     | **0.858**  | **0.231** | 0.228      | **0.817**   | 0.814      |

</td><td>

|              |        | AUC       |            | PPV       |            | Specificity |            |
|--------------|--------|-----------|------------|-----------|------------|-------------|------------|
| Cohort type  | Cohort | Global    | Multi-task | Global    | Multi-task | Global      | Multi-task |
| Careunits    | CCU    | **0.865** | 0.857      | **0.206** | 0.195      | 0.785       | 0.778      |
|              | CSRU   | 0.889     | **0.903**  | **0.098** | 0.093      | 0.846       | 0.848      |
|              | MICU   | 0.828     | **0.833**  | 0.227     | **0.243**  | 0.681       | **0.709**  |
|              | SICU   | **0.855** | 0.842      | 0.209     | 0.210      | 0.742       | 0.745      |
|              | TSICU  | 0.865     | **0.871**  | **0.222** | 0.211      | **0.797**   | 0.788      |
|              | Macro  | 0.860     | 0.861      | **0.192** | 0.190      | 0.770       | 0.774      |
|              | Micro  | 0.862     | **0.864**  | 0.202     | **0.216**  | 0.759       | **0.778**  |
| Unsupervised | 0      | **0.871** | 0.857      | **0.194** | 0.176      | **0.777**   | 0.744      |
|              | 1      | 0.828     | **0.832**  | 0.217     | **0.223**  | 0.701       | **0.713**  |
|              | 2      | 0.896     | **0.903**  | 0.171     | **0.193**  | 0.807       | **0.830**  |
|              | Macro  | **0.865** | 0.864      | 0.194     | **0.197**  | 0.762       | 0.762      |
|              | Micro  | 0.865     | 0.865      | 0.204     | **0.210**  | 0.762       | **0.770**  |

</td></tr> </table>


#### 2. Original Paper (Table 5) - 48 Hours Mortality Prediction 

<table>
<tr><th>Original Paper Result (48 Hours) </th><th>Reproduction Paper Result (48 Hours)</th></tr>
<tr><td>

|              |        | AUC       |            | PPV       |            | Specificity |            |
|--------------|--------|-----------|------------|-----------|------------|-------------|------------|
| Cohort type  | Cohort | Global    | Multi-task | Global    | Multi-task | Global      | Multi-task |
| Careunits    | Macro  | **0.859** | 0.839      | **0.187** | 0.170      | **0.833**   | 0.826      |
|              | Micro  | **0.865** | 0.856      | **0.206** | 0.198      | **0.833**   | 0.832      |
| Unsupervised | Macro  | 0.834     | 0.833      | 0.154     | 0.154      | **0.789**   | 0.775      |
|              | Micro  | **0.865** | 0.861      | 0.206     | 0.191      | **0.833**   | 0.812      |

</td><td>

|              |        | AUC    |            | PPV    |            | Specificity |            |
|--------------|--------|--------|------------|--------|------------|-------------|------------|
| Cohort type  | Cohort | Global | Multi-task | Global | Multi-task | Global      | Multi-task |
| Careunits    | Macro  | 0.849  | **0.859**  | 0.156  | **0.169**  | 0.753       | **0.781**  |
|              | Micro  | 0.852  | **0.862**  | 0.157  | **0.175**  | 0.740       | **0.773**  |
| Unsupervised | Macro  | 0.855  | **0.860**  | 0.169  | **0.179**  | 0.754       | **0.773**  |
|              | Micro  | 0.853  | **0.857**  | 0.158  | **0.168**  | 0.743       | **0.760**  |

</td></tr> </table>





