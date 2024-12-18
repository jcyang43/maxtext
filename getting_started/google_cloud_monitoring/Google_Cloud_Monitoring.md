# Getting started with Google Cloud monitoring 
This guide provides an overview on how to integrate Google's [workload observability agent](https://us-west2-docker.pkg.dev/gce-ai-infra/workload-observability/model-workload-observability) container into your MaxText training workload. 

## Overview
To address Google Cloud's lack of visibility into user workload performance, Google has added a customer workload performance monitoring feature for critical workloads sensitive to infrastructure changes. 
Once integrated, the workload observability agent reports metric to Google Cloud, enabling Google engineers to track workload performance metrics.
If performance falls below a certain threshold, the Google Cloud on-call team will be alerted. 

The workload observability currently supports heartbeat and performance (training step time) metrics. In the near future, support for the goodput metric will also be added.
Users should work with their Customer Engineer (CE) and the Google team to define appropriate thresholds for the performance metrics.

This guide provides an example of how to integrate with the workload observability agent to send metrics to Google Cloud for monitoring for MaxText workloads.

## Pre-requisites 
Please make sure you have the following before starting: 
1. A GCP account with billing enabled 
2. A GKE cluster ready in your project. For this example, we use a v6e TPU cluster with a 4x4 topology. If you choose a different cluster, please remember to modify your configurations accordingly.
3. A service account with the following permissions:
    - Access to your Google Cloud Storage (GCS) bucket
    - Access to your Artifact Repository 
    - Access to your GKE cluster

## Instructions
### 1. Authenticate with GCP
Verify you're authenticated with GCP on your environment: 
```
gcloud auth login
gcloud auth configure-docker
```

### 2. Set up GCS bucket, artifact repository & cluster nodepool 
Create a GCS bucket that will serve as the output directory for your MaxText training workload:
```
gsutil mb -l <your-zone> gs://<your-bucket-name>/
```

Export the GCS bucket path as environment variable `GCS_BUCKET_PATH`:
```
export GCS_BUCKET_PATH=gs://<your-bucket-name>
```

Create an artifact repository for Docker images: 
```
gcloud artifacts repositories create <repo-name> \
    --repository-format=docker \
    --location=<your-zone> \
    --description=<your-choice>
```

Create a nodepool on the GKE cluster:
```
gcloud container node-pools create <pool-name> \
    -- location=<yoru-zone> \
    --cluster=<your-gke-cluster-name>
    --node-locations=<your-node-locations> \
    --machine-type=ct6e-standard-4t \ 
    --tpu-topology=4x4 \ 
    --num-nodes=4
```

### 3. Create a Kubernetes secret 
Create a Kubernetes secret to provide access to your GCS bucket: 
```
kubectl create secret generic gcs-key \ 
    --from-file=/path/to/your/service-account-key.json
```

### 4. Build your MaxText Docker image 
In the project root directory, run 
```
bash docker_build_dependency_image.sh DEVICE=tpu

# tag and push your image 
docker tag maxtext_base_image:latest <your-docker-registry-path>/maxtext_base_tpu:latest
docker push <your-docker-registry-path>/maxtext_base_tpu:latest
```

On this [line](./tpu_v6e_with_gcp_monitoring.yaml#L39) of the config file, replace the placeholder image with the image you just built (`<your-docker-registry-path>/maxtext_base_tpu:latest`).

### 5. Set the training dataset
Export the path to your dataset on GCS bucket: 

```
EXPORT DATASET_PATH=<path-to-your-training-dataset>
```

If you don't have a training dataset or want to try with synthetic data, replace `dataset_path=${DATASET_PATH}` with `dataset_type=synthetic` on this [line](./tpu_v6e_with_gcp_monitoring.yaml#L52).

### 6. Launch the workload
Finally, launch the workload with the [config](./tpu_v6e_with_gcp_monitoring.yaml) in this directory
```
# in the current directory 

export TIMESTAMP=$(date +"%Y-%m-%dT%H-%M-%S") && envsubst < tpu_v6e_with_gcp_monitoring.yaml | kubectl apply -f - 
```

Once deployed, the workload observability container will parse the TFevents files written to your chosen GCS bucket and report metrics to Google Cloud for monitoring.