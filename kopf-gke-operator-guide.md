

# Create a Kubernetes Operator in Minutes with Kopf on GKE

Managing applications on Kubernetes is powerful, but it often involves repetitive manual tasks. The **Operator pattern** is the gold standard for automating this complexity, extending the Kubernetes API to manage your applications and their resources as if they were native components.

While many operators are written in Go, this can be a steep learning curve. Enter **Kopf (Kubernetes Operator Pythonic Framework)**, a framework that lets you write full-featured operators in Python with surprising simplicity.

In this guide, we'll build a practical Kubernetes operator on the **Google Kubernetes Engine (GKE)**. Our operator will watch for a new kind of resource, `GcsBucket`, and will automatically create or delete a corresponding **Google Cloud Storage (GCS)** bucket.

### What We'll Build

*   **A Custom Resource Definition (CRD)** for `GcsBucket` objects.
*   **A Python-based operator** using Kopf that reacts to the creation and deletion of these resources.
*   **Secure authentication** between our GKE pod and GCP APIs using Workload Identity.

---

### Prerequisites

Before you begin, ensure you have the following:
1.  A Google Cloud Platform (GCP) project.
2.  The `gcloud` CLI installed and authenticated (`gcloud auth login`).
3.  A GKE cluster with **Workload Identity** enabled.
4.  `kubectl` installed and configured to connect to your GKE cluster.
5.  Python 3.8+ and `pip` installed.

---

### Step 1: Configure GKE and GCP Permissions

For our operator's pod to securely interact with GCS, we'll use **Workload Identity**, GKE's recommended method for granting workloads access to GCP services.

First, let's define some environment variables to make the next steps easier.

```bash
# Your GCP Project ID
export PROJECT_ID=$(gcloud config get-value project)

# The name for our GCP and Kubernetes Service Accounts
export SERVICE_ACCOUNT_NAME=gcs-bucket-operator

# The namespace for our operator
export K8S_NAMESPACE=default
```

Now, let's create the necessary service accounts and bind them.

1.  **Create a GCP Service Account (GSA):**
    This account will have permissions to manage GCS buckets.

    ```bash
    gcloud iam service-accounts create ${SERVICE_ACCOUNT_NAME} \
      --display-name="GCS Bucket Operator Service Account"
    ```

2.  **Grant the GSA GCS Admin Permissions:**
    We grant the `storage.admin` role to allow the operator to create and delete buckets.

    ```bash
    gcloud projects add-iam-policy-binding ${PROJECT_ID} \
      --member="serviceAccount:${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
      --role="roles/storage.admin"
    ```

3.  **Create a Kubernetes Service Account (KSA):**
    This account will be used by our operator's pod inside the GKE cluster.

    ```bash
    kubectl create serviceaccount ${SERVICE_ACCOUNT_NAME} --namespace ${K8S_NAMESPACE}
    ```

4.  **Bind the GSA and KSA:**
    This is the core of Workload Identity. We create an IAM policy binding that allows the KSA to impersonate the GSA.

    ```bash
    gcloud iam service-accounts add-iam-policy-binding \
      ${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com \
      --role="roles/iam.workloadIdentityUser" \
      --member="serviceAccount:${PROJECT_ID}.svc.id.goog[${K8S_NAMESPACE}/${SERVICE_ACCOUNT_NAME}]"
    ```

5.  **Annotate the Kubernetes Service Account:**
    Finally, we annotate the KSA to link it to the GSA.

    ```bash
    kubectl annotate serviceaccount ${SERVICE_ACCOUNT_NAME} \
      --namespace ${K8S_NAMESPACE} \
      iam.gke.io/gcp-service-account=${SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com
    ```

Our environment is now securely configured!

---

### Step 2: Define the Custom Resource (CRD)

We need to tell Kubernetes about our new `GcsBucket` resource type. Save the following YAML as `gcsbucket-crd.yaml`.

```yaml
# gcsbucket-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: gcsbuckets.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                location:
                  type: string
                  description: "The GCP location for the bucket (e.g., US-CENTRAL1)."
              required:
                - location
  scope: Namespaced
  names:
    plural: gcsbuckets
    singular: gcsbucket
    kind: GcsBucket
    shortNames:
    - gcs
```

This CRD defines a new resource `GcsBucket` with a single required field in its `spec`: `location`. The bucket's name will be derived from the resource's metadata name.

Apply it to your cluster:
```bash
kubectl apply -f gcsbucket-crd.yaml
```

---

### Step 3: Write the Operator Code

Now for the fun part! Let's write the Python code for our operator.

First, install the necessary libraries:
```bash
pip install kopf google-cloud-storage
```

Next, save the following code as `operator.py`.

```python
# operator.py
import kopf
import logging
import os # Added for os.environ
from google.cloud import storage
from google.api_core.exceptions import Conflict, NotFound

# Get the GCP Project ID from an environment variable
PROJECT_ID = os.environ.get("GCP_PROJECT_ID")
if not PROJECT_ID:
    raise RuntimeError("GCP_PROJECT_ID environment variable must be set")

@kopf.on.create('example.com', 'v1', 'gcsbuckets')
def create_bucket(name, spec, logger, **kwargs):
    """
    This function is triggered when a GcsBucket resource is created.
    It creates a corresponding Google Cloud Storage bucket.
    """
    location = spec.get('location')
    if not location:
        raise kopf.PermanentError("Spec must include a 'location'.")

    bucket_name = f"{PROJECT_ID}-{name}"
    logger.info(f"Attempting to create GCS bucket: {bucket_name} in {location}")

    try:
        storage_client = storage.Client()
        bucket = storage_client.create_bucket(bucket_name, location=location)
        logger.info(f"Successfully created bucket: {bucket.name}")
        return {'bucketName': bucket.name, 'message': 'Bucket created successfully'}
    except Conflict:
        logger.warning(f"Bucket {bucket_name} already exists. Adopting.")
        return {'bucketName': bucket_name, 'message': 'Bucket already exists'}
    except Exception as e:
        raise kopf.PermanentError(f"Failed to create bucket: {e}")

@kopf.on.delete('example.com', 'v1', 'gcsbuckets')
def delete_bucket(name, logger, **kwargs):
    """
    This function is triggered when a GcsBucket resource is deleted.
    It deletes the corresponding Google Cloud Storage bucket.
    """
    bucket_name = f"{PROJECT_ID}-{name}"
    logger.info(f"Attempting to delete GCS bucket: {bucket_name}")

    try:
        storage_client = storage.Client()
        bucket = storage_client.get_bucket(bucket_name)
        bucket.delete()
        logger.info(f"Successfully deleted bucket: {bucket_name}")
        return {'message': f'Bucket {bucket_name} deleted successfully'}
    except NotFound:
        logger.warning(f"Bucket {bucket_name} not found. Nothing to delete.")
        return {'message': 'Bucket not found'}
    except Exception as e:
        raise kopf.PermanentError(f"Failed to delete bucket: {e}")

```

**Code Breakdown:**
*   **`@kopf.on.create(...)`**: This decorator tells Kopf to run the `create_bucket` function whenever a `GcsBucket` resource is created.
*   **`@kopf.on.delete(...)`**: This tells Kopf to run `delete_bucket` when a `GcsBucket` is deleted.
*   **Authentication**: We don't pass any credentials to `storage.Client()`. Because we configured Workload Identity, the Google Cloud client library automatically and securely authenticates using the pod's service account.
*   **Error Handling**: We catch `Conflict` errors if the bucket already exists and `NotFound` if we try to delete a bucket that isn't there.
*   **`kopf.PermanentError`**: This tells the operator to stop retrying for this resource if a critical error occurs.

---

### Step 4: Run the Operator

Kopf provides a fantastic CLI for running the operator locally, which is perfect for development. It connects to your cluster using your local `kubectl` configuration.

Open a terminal and run:
```bash
# Replace with your actual Project ID
export GCP_PROJECT_ID=${PROJECT_ID}

kopf run operator.py --namespace=${K8S_NAMESPACE}
```

You should see Kopf start up and begin watching for resources.

---

### Step 5: Test the Operator

With the operator running, let's create a `GcsBucket` resource.

1.  **Create a Resource:**
    Save the following as `my-first-bucket.yaml`.

    ```yaml
    # my-first-bucket.yaml
    apiVersion: example.com/v1
    kind: GcsBucket
    metadata:
      name: my-awesome-app-data
    spec:
      location: US-CENTRAL1
    ```

2.  **Apply it:**
    In a **new terminal**, apply the manifest.

    ```bash
    kubectl apply -f my-first-bucket.yaml
    ```

3.  **Check the Logs:**
    Look at the terminal where `kopf` is running. You should see logs indicating that it detected the new resource and created a GCS bucket named `<your-project-id>-my-awesome-app-data`.

4.  **Verify in GCP:**
    Check the GCS console in your GCP project. The new bucket will be there!

5.  **Delete the Resource:**
    Now, let's test the deletion handler.

    ```bash
    kubectl delete -f my-first-bucket.yaml
    ```

6.  **Verify Deletion:**
    The Kopf logs will show the deletion event, and if you check the GCS console again, the bucket will be gone.

---

### Step 6: Deploying to the Cluster (Production)

Running the operator locally is great for testing, but for production, you'll want to deploy it as a `Deployment` in your cluster.

1.  **Containerize the Operator:**
    Create a `Dockerfile` and a `requirements.txt` file.

    ```txt
    # requirements.txt
    kopf
    google-cloud-storage
    ```

    ```Dockerfile
    # Dockerfile
    FROM python:3.9-slim
    WORKDIR /app
    COPY requirements.txt .
    RUN pip install --no-cache-dir -r requirements.txt
    COPY operator.py .
    CMD ["kopf", "run", "/app/operator.py", "--all-namespaces"]
    ```
    Build and push this image to a container registry like GCR or Docker Hub.

2.  **Create the Deployment Manifest:**
    Save the following as `operator-deployment.yaml`. **Remember to replace the `image` and `GCP_PROJECT_ID` values.**

    ```yaml
    # operator-deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: gcs-bucket-operator
      namespace: default
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: gcs-bucket-operator
      template:
        metadata:
          labels:
            app: gcs-bucket-operator
        spec:
          # Use the KSA we created in Step 1
          serviceAccountName: gcs-bucket-operator
          containers:
          - name: operator
            # Replace with your image path
            image: gcr.io/your-project/gcs-bucket-operator:latest
            env:
            - name: GCP_PROJECT_ID
              # Replace with your GCP Project ID
              value: "your-gcp-project-id"
    ```

3.  **Deploy it:**
    ```bash
    kubectl apply -f operator-deployment.yaml
    ```

Your operator is now running natively within your GKE cluster!

### Conclusion

You've successfully built and deployed a fully functional Kubernetes operator using Python and Kopf. We extended the Kubernetes API with a custom resource and automated the lifecycle of an external resource (a GCS bucket) in a secure, cloud-native way.

Kopf dramatically lowers the barrier to entry for operator development, empowering you to automate complex operational knowledge into simple, maintainable Python code. From here, you can add more features: updating bucket labels, managing IAM policies, or reporting status back to the `GcsBucket` resource. Happy coding!
