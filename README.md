# Operationalizing Machine Learning with Azure ML

## Overview

This project is part of the Udacity **Machine Learning Engineer with Microsoft Azure** Nanodegree. It uses the **Bank Marketing** dataset, which describes the clients of a Portuguese bank and whether they subscribed to a term deposit after a marketing campaign (column `y`: `yes` / `no`).

The goal is to take a model all the way from training to production in Azure Machine Learning:

1. Train a classification model with **Automated ML (AutoML)**.
2. **Deploy** the best model as a web service on **Azure Container Instance (ACI)** with authentication enabled.
3. Enable **Application Insights** and retrieve logs with the Python SDK.
4. Document the API with **Swagger** and **consume** the endpoint with a Python script.
5. Build, publish and trigger a reusable **AutoML training pipeline** through a REST endpoint.


## Key Steps

### 1. Authentication

Creating a Service Principal is not permitted in the Udacity-provided lab environment, so this step was skipped, as allowed by the project instructions. It is not graded and does not affect the other steps.

### 2. Automated ML Experiment

The original dataset link (`automlsamplenotebookdata.blob.core.windows.net`) is no longer available, so the same `bankmarketing_train.csv` file was obtained from a public GitHub copy and registered in Azure ML Studio as the **bankmarketing** dataset.

An AutoML run was then created with these settings:

- Task type: **Classification**, target column `y`
- Compute cluster: **Standard_D3_v2**, minimum 1 node
- **Explain best model** enabled
- Exit criterion: **1 hour**; max concurrent iterations: **5**

The best model was a **VotingEnsemble** with an AUC weighted of about **0.948**.

### 3. Deploy the Best Model

The best model was deployed as a web service on **Azure Container Instance (ACI)** with **key-based authentication enabled**. Once deployment finished, the endpoint reported a **Healthy** state.

### 4. Enable Application Insights

In the Azure ML compute instance terminal, `az` and the Azure ML Python SDK were verified, and a new Python 3 virtual environment was created. A `config.json` with the workspace details was placed next to `logs.py`.

`logs.py` was updated to set the deployment name and to call `service.update(enable_app_insights=True)`. Running it enabled Application Insights and printed the service logs. The endpoint's Details tab then showed **Application Insights enabled: true**.

### 5. Swagger Documentation

The `swagger.json` file was downloaded from the endpoint's Swagger URI into the `swagger` folder, next to `serve.py`. The port in `swagger.sh` was changed from 80 to **9000**.

Swagger UI was run in Docker on the Azure ML compute instance, and `serve.py` served `swagger.json` on port 8000. Because the lab browser cannot reach the compute instance's `localhost` directly, Swagger was opened through the compute instance's port-forwarding address. To avoid a browser CORS error between the two ports, the Swagger UI container was started with `swagger.json` mounted directly:

```bash
docker run -p 9000:8080 -e SWAGGER_JSON=/foo/swagger.json -v $(pwd):/foo swaggerapi/swagger-ui
```

The page shows the model's HTTP API: `GET /` (health check), `POST /score` (prediction) and `GET /swagger.json`, including the example input payload and the possible responses.

### 6. Consume the Model Endpoint

`endpoint.py` was updated with the endpoint's **scoring URI** and **primary key**. It sends two sample clients as a JSON payload in an HTTP POST request and prints the model's predictions.

The newer AutoML scoring script expects the input wrapped in an `Inputs` field, so one line was added to `endpoint.py` before the data is converted to JSON:

```python
data = {"Inputs": data, "GlobalParameters": {"method": "predict"}}
```

The script returned a JSON response with one prediction per client:

```
{'Results': ['no', 'no']}
```

### 7. Create, Publish and Consume a Pipeline
The notebook `aml-pipelines-with-automated-machine-learning-step.ipynb` was run in Azure ML Studio with the **Python 3.8 – AzureML** kernel, after updating the experiment, compute cluster and dataset names to match this workspace. The missing `azureml-pipeline` package was installed in the notebook.

