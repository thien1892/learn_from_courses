# 1. Using BigQuery ML to Predict Penguin Weight
## 1.1. Overview
- In this lab, you use the penguin table to create a model that predicts the weight of a penguin based on the penguin's species, island of residence, culmen length and depth, flipper length, and sex.

- This lab introduces data analysts to BigQuery ML. BigQuery ML enables users to create and execute machine learning models in BigQuery using SQL queries. The goal is to democratize machine learning by enabling SQL practitioners to build models using their existing tools and to increase development speed by eliminating the need for data movement.

## 1.2. Learning objectives
- Create a linear regression model using the CREATE MODEL statement with BigQuery ML.

- Evaluate the ML model with the ML.EVALUATE function.

- Make predictions using the ML model with the ML.PREDICT function.

## 1.3. Step by step

### 1.3.1. Create your dataset
- In the Cloud Console, on the Navigation menu, click BigQuery.

- In the Explorer panel, click the View actions icon (three vertical dots) next to your project ID, and then click Create dataset.

- On the Create dataset page: For Dataset ID, type **bqml_tutorial**
- Leave the remaining settings as their defaults, and click Create Dataset

### 1.3.2. Create your model
```
#standardSQL
CREATE OR REPLACE MODEL `bqml_tutorial.penguins_model`
OPTIONS
  (model_type='linear_reg',
    input_label_cols=['body_mass_g']) AS
SELECT
  *
FROM
  `bigquery-public-data.ml_datasets.penguins`
WHERE
  body_mass_g IS NOT NULL
```
### 1.3.3. Evaluate your model
```
#standardSQL
SELECT
  *
FROM
  ML.EVALUATE(MODEL `bqml_tutorial.penguins_model`,
    (
    SELECT
      *
    FROM
      `bigquery-public-data.ml_datasets.penguins`
    WHERE
      body_mass_g IS NOT NULL))
```

### 1.3.4. Use your model to predict outcomes
```
#standardSQL
  SELECT
    *
  FROM
    ML.PREDICT(MODEL `bqml_tutorial.penguins_model`,
      (
      SELECT
        *
      FROM
        `bigquery-public-data.ml_datasets.penguins`
      WHERE
        body_mass_g IS NOT NULL
        AND island = "Biscoe"))
```

### 1.3.5. Explain prediction results with explainable AI methods
```
#standardSQL
SELECT
*
FROM
ML.EXPLAIN_PREDICT(MODEL `bqml_tutorial.penguins_model`,
  (
  SELECT
    *
  FROM
    `bigquery-public-data.ml_datasets.penguins`
  WHERE
    body_mass_g IS NOT NULL
    AND island = "Biscoe"),
  STRUCT(3 as top_k_features))
```
### 1.3.6. Globally explain your model (Optional)
- To know which features are the most important to determine the weights of the penguins in general, you can use the ML.GLOBAL_EXPLAIN function
```
#standardSQL
CREATE OR REPLACE MODEL bqml_tutorial.penguins_model
OPTIONS
  (model_type='linear_reg',
  input_label_cols=['body_mass_g'],
  enable_global_explain=TRUE) AS
SELECT
  *
FROM
  `bigquery-public-data.ml_datasets.penguins`
WHERE
  body_mass_g IS NOT NULL
```
```
#standardSQL
SELECT
*
FROM
ML.GLOBAL_EXPLAIN(MODEL `bqml_tutorial.penguins_model`)
```

# 2. Using the BigQuery ML Hyperparameter Tuning to Improve Model Performance

## 2.1. Overview
- This lab introduces data analysts to BigQuery ML. BigQuery ML enables users to create and execute machine learning models in BigQuery using SQL queries. This lab introduces a method of hyperparameter tuning that specifies the num_trials training option.

- In this lab, you use the tlc_yellow_trips_2018 sample table to create a model that predicts the tip for a taxi ride. You will see a ~40% performance (r2_score) improvement with hyperparameter tuning.

## 2.2. Objectives
>In this lab, you use BigQuery ML to:

- Create a linear regression model using the CREATE MODEL statement with the num_trials set to 20.

- Check the overview of all 20 trials using the ML.TRIAL_INFO function.

- Evaluate the ML model using the ML.EVALUATE function.

- Make predictions using the ML model and ML.PREDICT function.

>Enable the BigQuery API
- In the Google Cloud Console, on the Navigation menu (Navigation Menu icon), click APIs & services > Library.
- Search for BigQuery API, and then click Enable if it isn't already enabled.

## 2.3. Rezolve step by step
### 2.3.1. Create your training dataset
- On the BigQuery page, in the Explorer panel, click View actions (Views actions icon) next to your project ID, and select Create dataset.

- For Dataset ID, type **bqml_tutorial**, and for Data location, select United States (US).Currently, the public datasets are stored in the US multiregional location. For simplicity, place your dataset in the same location.

- Leave the remaining settings as their defaults, and click Create dataset.

### 2.3.2. Create your training input table
```
CREATE TABLE `bqml_tutorial.taxi_tip_input` AS
SELECT
  * EXCEPT(tip_amount), tip_amount AS label
FROM
  `bigquery-public-data.new_york_taxi_trips.tlc_yellow_trips_2018`
WHERE
  tip_amount IS NOT NULL
LIMIT 100000
```
### 2.3.3. Create your model
```
CREATE MODEL `bqml_tutorial.hp_taxi_tip_model`
OPTIONS
  (model_type='linear_reg',
   num_trials=20,
   max_parallel_trials=2) AS
SELECT
  *
FROM
  `bqml_tutorial.taxi_tip_input`
```

- The LINEAR_REG model has two tunable hyperparameters: l1_reg and l2_reg. The previous query uses the default search space. You can also specify the search space explicitly:
```
OPTIONS
  (...
    l1_reg=hparam_range(0, 20),
    l2_reg=hparam_candidates([0, 0.1, 1, 10]))
```
- In addition, these other hyperparameter tuning training options also use their default values:

    - hparam_tuning_algorithm: "VIZIER_DEFAULT"
    - hparam_tuning_objectives: ["r2_score"]
- max_parallel_trials is set to 2 to accelerate the tuning process. With two trials running at any time, the whole tuning should take approximately as long as 10 serial training jobs instead of 20. Note, however, that the two concurrent trials cannot benefit from each other's training results.

### 2.3.4. Get trials information
```
SELECT *
FROM
  ML.TRIAL_INFO(MODEL `bqml_tutorial.hp_taxi_tip_model`)
```

### 2.3.5. Evaluate your model
```
SELECT *
FROM
  ML.EVALUATE(MODEL `bqml_tutorial.hp_taxi_tip_model`)
```

### 2.3.6. Use your model to predict taxi tips
```
SELECT
  *
FROM
  ML.PREDICT(MODEL `bqml_tutorial.hp_taxi_tip_model`,
    (
    SELECT
      *
    FROM
      `bqml_tutorial.taxi_tip_input`
    LIMIT 10))
```
- The prediction is made against the optimal trial by default.
- You can select another trial by specifying the trial_id parameter. For instance, code below is use model trail id 3 (You can see the result in evaluation for details)
```
SELECT
  *
FROM
  ML.PREDICT(MODEL `bqml_tutorial.hp_taxi_tip_model`,
    (
    SELECT
      *
    FROM
      `bqml_tutorial.taxi_tip_input`
    LIMIT
      10),
    STRUCT(3 AS trial_id))
```

### 2.3.7.  Clean up

Deleting your project removes all datasets and all tables in the project. If you prefer to reuse the project, you can delete the dataset you created in this tutorial:

- If necessary, open the BigQuery page in the Cloud Console.

- In the Explorer panel, click View actions (Views actions icon) next to your dataset, and then click Delete.

- In the Delete dataset dialog, to confirm the delete command, type delete, and then click Delete.

## 4. Reference
- [ML.TRIAL_INFO](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-trial-info)
- [ML.EVALUATE](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-hyperparameter-tuning#existing_functions)
- [Data Split](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-hyperparameter-tuning#data_split): to see the difference between ML.TRIAL_INFO objectives and ML.EVALUATE evaluation metrics.
- [ML.PREDICT](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-hyperparameter-tuning#existing_functions)
- [BigQuery ML](https://cloud.google.com/bigquery-ml/docs)
- [Creating and Training Models](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-create)
- [BigQuery ML Hyperparameter Tuning](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-hp-tuning-overview)
- [BigQuery ML Model Evaluation Overview](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-evaluate-overview)