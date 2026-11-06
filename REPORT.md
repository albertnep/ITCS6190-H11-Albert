# Hands-on L11: Report

**Name: Albert Bastakoti**
**Student ID:** 801291837
**Email: abastako@charlotte.edu**

---

## Seed and commands

Seed used (your student ID):
801291837

The commands you ran, in order. If you deviated from the steps in the README, say where and
why.

```bash
docker desktop start
docker compose up -d
docker cp main.py spark-master:/opt/spark/work-dir/
docker exec spark-master /opt/spark/bin/spark-submit --master spark://spark-master:7077 /opt/spark/work-dir/main.py task1 /opt/spark/work-dir/shared 801291837
docker exec spark-master /opt/spark/bin/spark-submit --master spark://spark-master:7077 /opt/spark/work-dir/main.py task2 /opt/spark/work-dir/shared 801291837
```

---

## Results

### Task 1: baseline and full pipeline

Paste the two lines printed by `evaluate()` (also in `shared-folder/output/task1/metrics`),
the five predictions shown and the "saved to" line.

```
baseline                         RMSE   316.2   R2  0.151
full                             RMSE   298.2   R2  0.246
saved to /opt/spark/work-dir/shared/models/full
```

How many training and test rows did your seed give? The slides show 5,780 and 1,366 with
seed 42; why is yours different?

My seed gave 5,735 training rows and 1,411 test rows. randomSplit assigns rows randomly, so a different seed produces a different split size and different rows.

### Task 2: scoring with the saved model

Paste the stage list, the RMSE of the loaded model and the five predictions.

```
stages: StringIndexerModel, OneHotEncoderModel, VectorAssembler, LinearRegressionModel
full                             RMSE   298.2   R2  0.246
Bayview,Entire home/apt,1.0,88.0,103.1
Bayview,Entire home/apt,1.0,105.0,63.8
Bayview,Entire home/apt,1.0,109.0,104.5
Bayview,Entire home/apt,1.0,219.0,71.1
Bayview,Entire home/apt,1.0,118.0,60.9
```

---

## Questions

**1. Baseline versus full.** With seed 42 the lecture gets 221.6 / 0.152 for the baseline
and 220.6 / 0.160 for the full pipeline. What did you get, and how much did 97 more
features buy? The price goes from 10 to 10,000 dollars a night (the `summary()` in the
lecture): how does that affect an RMSE measured in dollars, and why do the numbers change
from seed to seed? Name one change to the pipeline you would try first (slides "Running
Example" with the model table, and "Tuning").

The baseline had an RMSE of 316.2 and an R² of 0.151, while the full pipeline had an RMSE of 298.2 and an R² of 0.246. This means the additional features lowered the RMSE by $18 and improved R² by 0.095. RMSE is measured in dollars, so mistakes on expensive listings have a bigger impact. The wide price range also makes the results sensitive to outliers. Results may change with different seeds because the training and testing data change. I would try adjusting regParam and elasticNetParam to improve the linear regression model.




**2. What is in `shared-folder/models/full`?** List its folders (`find shared-folder/models/full
-maxdepth 2`) and say what each holds. Why do we save the PipelineModel rather than the
LinearRegressionModel alone? Which stages of the saved model are fitted models and which
are plain transformers?

The saved model contains metadata, stages, and these stage folders:
`stages/0_StringIndexer_5d86f42d82ef`, `stages/1_OneHotEncoder_1fb9f0892548`,
`stages/2_VectorAssembler_123b2e702445`, and
`stages/3_LinearRegression_84a2e1accf2e`. The metadata records the PipelineModel
format and stage metadata. Each stage folder stores that stage's metadata and model
data where applicable. Saving the PipelineModel preserves the complete feature
preprocessing sequence and the regression model, so scoring applies exactly the same
category mappings and feature vector construction used during training. The fitted
stages are StringIndexerModel, OneHotEncoderModel, and LinearRegressionModel.
VectorAssembler is a plain transformer.



**3. Only transformers at scoring time.** Task 2 ran no `fit()`. Compare the number of
predictions written with the number of test rows: if they differ, where did the rows go,
and which parameter of which stage decides it? What would you have to change in task 2 to
score a stream of listings instead of a Parquet file (slide "Scoring a Stream")?

The test set has 1,411 rows, but only 1,410 predictions were made. One row was dropped because it had a category the model had not seen before. The parameter handleInvalid="skip" in the StringIndexer caused that row to be removed. To score a stream instead of a Parquet file, I would use spark.readStream to read the data, apply the saved model with transform, and use writeStream to save the predictions.



---

## Problems and fixes

Anything that went wrong and what resolved it. Paste the actual error message. If nothing
went wrong, say so.

Docker was initially not running:

`failed to connect to the docker API at npipe:////./pipe/dockerDesktopLinuxEngine`

I started Docker Desktop. The existing fixed-name containers were from another hands-on
and mounted the wrong directory, producing:

`[PATH_NOT_FOUND] Path does not exist: file:/opt/spark/work-dir/shared/data/sf-airbnb-clean.parquet`

I replaced those three containers with this repository's Compose services. The official
Spark image also lacked NumPy:

`ModuleNotFoundError: No module named 'numpy'`

I installed NumPy, then reran both tasks successfully.
