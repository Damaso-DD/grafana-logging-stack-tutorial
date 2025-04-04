# Streamline Your Kubernetes Logging: Deploy Loki, Alloy, and Grafana with Helm

This repository has all the necessary configuration files for the tutorial: **Streamline Your Kubernetes Logging: Deploy Loki, Alloy, and Grafana with Helm**.

Here's a quick step-by-step procedure:

## Prerequisites

1. **Kubernetes Cluster:** A running Kubernetes cluster with a bare minimum of 4 GB of RAM and 2 vCPUs per node.
2. **`kubectl`:** Configured to interact with your K8s cluster.
3. **`helm`:** Helm v3 installed in your local machine.
4. **AWS S3 Bucket:** An S3 bucket created in your desired region (e.g., `us-east-1`).
5. **AWS Credentials:** An `AWS Access Key ID` and `Secret Access Key` with permissions to read/write to the S3 bucket.

## Preparing the environment

1. **Create a dedicated namespace for the logging stack:**

```bash
kubectl create namespace logging
```

2. **Create a K8s Secret with AWS Credentials:** Store your AWS credentials securely in a Kubernetes secret within the `logging` namespace. **Replace placeholders** with your actual credentials.

```bash
kubectl create secret generic aws-credentials \
  --from-literal=AWS_ACCESS_KEY_ID=<your-access-key-id> \
  --from-literal=AWS_SECRET_ACCESS_KEY=<your-secret-access-key> \
  -n logging
```

3. **Add Grafana Helm Repository:** Add the Grafana Helm repository which contains charts for Loki, Grafana, and Alloy.

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

## Setting Up Loki

1. **Install Loki using Helm:** The tutorial uses chart version `6.28.0`, feel free to experiment with newer stable versions if desired, but verify compatibility with the config.

>**IMPORTANT:** Edit `loki-helm-settings.yaml` and replace `us-east-1` and `loki-tutorial` with your actual S3 region and bucket name before continuing.

```bash
helm install loki grafana/loki \
  --version 6.28.0 \
  -f loki-helm-settings.yaml \
  -n logging
```

2. **Verify Loki Pods:**

```bash
kubectl get pods -n logging -l app.kubernetes.io/instance=loki
```

The output should be like this:

```bash
NAME                            READY   STATUS    RESTARTS   AGE
loki-backend-0                  2/2     Running   0          109s
loki-backend-1                  2/2     Running   0          109s
loki-backend-2                  2/2     Running   0          109s
loki-canary-52xcw               1/1     Running   0          110s
loki-canary-h8lgn               1/1     Running   0          110s
loki-canary-v69lg               1/1     Running   0          110s
loki-gateway-5ddb64f98d-vrb6n   1/1     Running   0          109s
loki-read-787ffc5594-7xxg7      1/1     Running   0          109s
loki-read-787ffc5594-97hsr      1/1     Running   0          109s
loki-read-787ffc5594-n9c8p      1/1     Running   0          109s
loki-results-cache-0            2/2     Running   0          109s
loki-write-0                    1/1     Running   0          109s
loki-write-1                    1/1     Running   0          109s
loki-write-2                    1/1     Running   0          109s
```

Wait for all Loki pods (write, read, backend, gateway) to be `Running` and `READY`.

>**IMPORTANT:**
>If Loki pods keep crashing or have errors, consider using nodes with more RAM and vCPUs.

## Setting Up Grafana

1. **Install Grafana using Helm:** The tutorial uses chart version `8.11.2`, feel free to experiment with newer stable versions if desired, but verify compatibility with the config. 

>**IMPORTANT:** Edit `grafana-helm-settings.yaml` and replace <YourSecurePassword!> with a strong password before continuing.

```bash
helm install grafana grafana/grafana \
  --version 8.11.2 \
  -f grafana-helm-settings.yaml \
  -n logging
```

2. **Verify Grafana and Get Access:**

```bash
kubectl get pods -n logging -l app.kubernetes.io/instance=grafana 
```

The output should be like this:

```bash
NAME                      READY   STATUS    RESTARTS   AGE
grafana-f6b969f99-lx97b   1/1     Running   0          4m54s
```

Wait for it to be `Running` and `READY 1/1`.

Get the Load Balancer External IP:

```bash
kubectl get service grafana -n logging 
```

Wait for the `EXTERNAL-IP` to be assigned (it can take a while). Copy the external IP address (e.g., 104.248.109.128)

```bash
NAME      TYPE           CLUSTER-IP     EXTERNAL-IP                                      PORT(S)        AGE
grafana   LoadBalancer   10.109.3.110   104.248.109.128,2604:a880:400:d1:0:1:655d:d001   80:32573/TCP   2m42s
```

* Access Grafana UI by opening `http://<EXTERNAL-IP>` in your browser.
* Login: Use username `admin` and the password you set in `grafana-helm-settings.yaml`.
* Check Datasource: Go to **Connections** -> **Data sources**. You should see the "Loki" data source already configured.

## Setting Up Alloy

1. **Review Alloy Configuration File:** Review and edit `alloy-settings.alloy` as necessary. This is the River configuration for Allow with K8s label definitions and relabeling logic. You can skip this step if you're following the tutorial.

2. **Create Kubernetes ConfigMap:** Create a `ConfigMap` named `alloy-settings` from the file `alloy-settings.alloy`.

```bash
kubectl create configmap alloy-settings \
  --from-file=config.alloy=./alloy-settings.alloy \
  -n logging
```

3. **Install Alloy using Helm:**

```bash
helm install alloy grafana/alloy \
  --version 0.12.6 \
  -f alloy-helm-settings.yaml \
  -n logging
```

4. **Verify Alloy Pods:** Alloy runs as a `DaemonSet` (one pod per node).

```bash
kubectl get pods -n logging -l app.kubernetes.io/name=alloy
```

Wait for all Alloy pods to be `Running` and `READY 1/1`. The output should be like this:

```bash
NAME          READY   STATUS    RESTARTS   AGE
alloy-hl4jl   1/2     Running   0          24s
alloy-ttbkm   2/2     Running   0          24s
alloy-xhfcj   1/2     Running   0          24s
```

## Deploy Sample Workload (Nginx)

Deploy a simple Nginx application to generate some logs.

1. **Create Namespace:**

```bash
kubectl create namespace demoapp
```

2. **Apply Deployment:**

```bash
kubectl apply -f nginx-deployment.yaml
```

3. **Verify Deployment:**

```bash
kubectl get pods -n demoapp
```

Ensure the Nginx pod is `Running`.

## Verify Log Flow in Grafana

1. Go back to your Grafana instance (`http://<EXTERNAL-IP>`).
2. Navigate to **Explore** or **Drilldown**.
3. Verify that **Loki** is set as datasource (**Connections > Data sources**).
4. Explore Kubernetes labels like `namespace`, `pod`, `container`, or `service` (e.g., **Drilldown > Logs > Add label**)

![Grafana Logs](https://i.imgur.com/XvFT1Df.png)
