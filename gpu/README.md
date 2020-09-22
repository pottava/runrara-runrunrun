# Cloud Run with [NVIDIA GPUs](https://cloud.google.com/compute/docs/gpus)

## 1. GKE cluster

### 1.1. Basic configurations

```bash
project_id=""
compute_region="asia-northeast1"
compute_zone="asia-northeast1-a"
gcloud config set project "${project_id}"
gcloud config set compute/region "${compute_region}"
gcloud config set compute/zone "${compute_zone}"
```

### 1.2. Create a GKE cluster

```bash
cluster_name="gpu-based-apps"
release_channel="regular"
# Create a GKE cluster
gcloud container clusters create ${cluster_name} \
    --zone ${compute_zone} --release-channel ${release_channel} --enable-ip-alias \
    --machine-type n1-standard-4 \
    --addons HttpLoadBalancing,CloudRun --preemptible \
    --num-nodes 1 --min-nodes 1 --max-nodes 2 --enable-autoscaling \
    --enable-autoupgrade --enable-autorepair \
    --max-surge-upgrade 1 --max-unavailable-upgrade 0 \
    --enable-stackdriver-kubernetes --no-enable-basic-auth \
    --scopes "https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/trace.append"
```

### 1.3. Add a T4 GPU nodepool

```bash
machine_type="n1-standard-4"
accelerator_type="nvidia-tesla-t4"
# Create a GPU nodepool
gcloud container node-pools create gpu-instances \
    --cluster ${cluster_name} --zone ${compute_zone} \
    --machine-type ${machine_type} \
    --accelerator type=${accelerator_type},count=1 \
    --disk-type "pd-standard" --disk-size "200" \
    --num-nodes 1 --min-nodes 0 --max-nodes 5 --enable-autoscaling \
    --enable-autoupgrade --enable-autorepair \
    --max-surge-upgrade 1 --max-unavailable-upgrade 0 \
    --preemptible
```

### 1.4. Update configurations

```bash
gcloud config set run/platform gke
gcloud config set run/cluster "${cluster_name}"
gcloud config set run/cluster_location "${compute_zone}"
gcloud container clusters get-credentials "${cluster_name}"
```

### 1.5. Install the NVIDIA GPU driver

```bash
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/container-engine-accelerators/master/nvidia-driver-installer/cos/daemonset-preloaded.yaml
# Wait for a field of `"nvidia.com/gpu": "1"`
kubectl get node -l cloud.google.com/gke-nodepool=gpu-instances -o json \
    | jq '.items[].status.allocatable'
```

## 2. Let's deploy :)

### 2.1. A hello service

```bash
svc_name="hello"
gcloud run deploy "${svc_name}" --image gcr.io/cloudrun/hello
# Curl the result
curl -I -H "Host: ${svc_name}.default.example.com" \
    "$( kubectl get svc istio-ingress -n gke-system \
        -o jsonpath='{.status.loadBalancer.ingress[0].ip}' )"
```

### 2.2. Check the GPU status

```bash
cat << EOF > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
  - name: gpu-container
    image: nvidia/cuda:10.2-base-ubuntu18.04
    command: ["/bin/bash", "-c", "nvidia-smi; sleep 10"]
    resources:
      limits:
        nvidia.com/gpu: 1
  restartPolicy: Never
EOF
kubectl apply -f pod.yaml
kubectl get pods -w
kubectl logs po/gpu-pod
kubectl delete pod gpu-pod
```

### 2.3. Run a TensorFlow Serving app

Build an app with 

```bash
cat << EOF > Dockerfile
FROM alpine:3.12 AS builder
RUN apk --no-cache add curl zip
RUN mkdir -p /tmp/resnet
RUN curl -sL "http://download.tensorflow.org/models/official/20181001_resnet/savedmodels/resnet_v2_fp32_savedmodel_NHWC_jpg.tar.gz" \
    | tar --strip-components=2 -C /tmp/resnet -xvz

FROM tensorflow/serving:2.3.0-gpu
ENV MODEL_NAME=resnet \
    PORT=8080
COPY --from=builder /tmp/resnet /models/resnet/
ADD entrypoint.sh /usr/bin/tf_serving_entrypoint.sh
RUN chmod +x /usr/bin/tf_serving_entrypoint.sh
EOF
cat << EOF > entrypoint.sh
#!/bin/bash
tensorflow_model_server --port=8500 --rest_api_port=\${PORT} \\
  --model_name=\${MODEL_NAME} --model_base_path=\${MODEL_BASE_PATH}/\${MODEL_NAME}
EOF
gcloud builds submit --tag "gcr.io/${project_id}/gpu-apps/resnet:rest"
```

Run it as a service

```bash
cat << EOF > service.yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: resnet
spec:
  template:
    spec:
      containers:
      - name: inference-server
        image: gcr.io/${project_id}/gpu-apps/resnet:rest
        resources:
          limits:
            nvidia.com/gpu: "1"
      containerConcurrency: 80
      timeoutSeconds: 240
EOF
gcloud container clusters resize ${cluster_name} \
    --node-pool gpu-instances --num-nodes 1
gcloud beta run services replace service.yaml
# Curl the result
curl -sX GET -H "Host: resnet.default.example.com" \
    "http://$( kubectl get svc istio-ingress -n gke-system \
        -o jsonpath='{.status.loadBalancer.ingress[0].ip}' \
    )/v1/models/resnet/metadata" | jq '.model_spec'
```

## 3. Stop the cluster

```bash
gcloud container clusters delete "${cluster_name}" --quiet --async
```
