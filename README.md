# Practice Lab: Autoscaling and Secrets Management

This practice lab is designed to provide hands-on experience with Kubernetes, focusing on vertical and horizontal pod autoscaling and secrets management.

---

## Objectives

In this practice lab, you will:

- **Build and deploy an application to Kubernetes**
- **Implement Vertical Pod Autoscaler (VPA)** to adjust pod resource requests/limits
- **Implement Horizontal Pod Autoscaler (HPA)** to scale the number of pod replicas based on resource utilization
- **Create a Secret** and update the deployment to use it

**Note:** Kindly complete the lab in a single session without any break because the lab may go into offline mode and cause errors. If you face any issues or errors during the lab process, please log out from the lab environment. Then, clear your system cache and cookies and try to complete the lab.

---

## Table of Contents

- [Setup the Environment](#setup-the-environment)
- [Prerequisites](#prerequisites)
- [Exercise 1: Build and Deploy an Application to Kubernetes](#exercise-1-build-and-deploy-an-application-to-kubernetes)
  - [Step 1: Build the Docker Image](#step-1-build-the-docker-image)
  - [Step 2: Push and List the Image](#step-2-push-and-list-the-image)
  - [Step 3: Deploy Your Application](#step-3-deploy-your-application)
  - [Step 4: View the Application Output](#step-4-view-the-application-output)
- [Exercise 2: Implement Vertical Pod Autoscaler (VPA)](#exercise-2-implement-vertical-pod-autoscaler-vpa)
  - [Step 1: Create a VPA Configuration](#step-1-create-a-vpa-configuration)
  - [Step 2: Apply the VPA](#step-2-apply-the-vpa)
  - [Step 3: Retrieve the Details of the VPA](#step-3-retrieve-the-details-of-the-vpa)
- [Exercise 3: Implement Horizontal Pod Autoscaler (HPA)](#exercise-3-implement-horizontal-pod-autoscaler-hpa)
  - [Step 1: Create an HPA Configuration](#step-1-create-an-hpa-configuration)
  - [Step 2: Configure the HPA](#step-2-configure-the-hpa)
  - [Step 3: Verify the HPA](#step-3-verify-the-hpa)
  - [Step 4: Start the Kubernetes Proxy](#step-4-start-the-kubernetes-proxy)
  - [Step 5: Generate Load on the App](#step-5-generate-load-on-the-app)
  - [Step 6: Observe the Effect of Autoscaling](#step-6-observe-the-effect-of-autoscaling)
  - [Step 7: Observe the Details of the HPA](#step-7-observe-the-details-of-the-hpa)
- [Exercise 4: Create a Secret and Update the Deployment](#exercise-4-create-a-secret-and-update-the-deployment)
  - [Step 1: Create a Secret](#step-1-create-a-secret)
  - [Step 2: Update the Deployment to Utilize the Secret](#step-2-update-the-deployment-to-utilize-the-secret)
  - [Step 3: Apply the Secret and Deployment](#step-3-apply-the-secret-and-deployment)
  - [Step 4: Verify the Secret and Deployment](#step-4-verify-the-secret-and-deployment)
- [Conclusion](#conclusion)
- [Author(s)](#authors)

---

## Setup the Environment

1. **Open the Terminal:**

   On the menu bar, click **Terminal** and select the **New Terminal** option from the drop-down menu.

   **Note:** If the terminal is already open, please skip this step.

---

## Prerequisites

- **kubectl Installed and Configured:** Ensure that you have `kubectl` installed and properly configured to interact with your Kubernetes cluster.

- **IBM Cloud CLI and Docker:** If you are using IBM Cloud, ensure that you have the IBM Cloud CLI and Docker installed.

---

## Exercise 1: Build and Deploy an Application to Kubernetes

The Dockerfile in this repository already has the code for the application. You will build the Docker image and push it to the registry. You will name your Kubernetes deployed application **myapp**.

### Step 1: Build the Docker Image

1. **Navigate to the Project Directory:**

   ```bash
   cd k8-scaling-and-secrets-mgmt
   ```

2. **Export Your Namespace:**

   ```bash
   export MY_NAMESPACE=sn-labs-$USERNAME
   ```

3. **Build the Docker Image:**

   ```bash
   docker build . -t us.icr.io/$MY_NAMESPACE/myapp:v1
   ```

### Step 2: Push and List the Image

1. **Push the Tagged Image to the IBM Cloud Container Registry:**

   ```bash
   docker push us.icr.io/$MY_NAMESPACE/myapp:v1
   ```

2. **List All the Images Available:**

   ```bash
   ibmcloud cr images
   ```

   You will see the newly created `myapp` image.

### Step 3: Deploy Your Application

1. **Open the `deployment.yaml` File:**

   The file is located in the main project directory. Its content will be as follows:

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: myapp
     labels:
       app: myapp
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: myapp
     strategy:
       type: RollingUpdate
       rollingUpdate:
         maxSurge: 25%
         maxUnavailable: 25%
     template:
       metadata:
         labels:
           app: myapp
       spec:
         containers:
           - name: myapp
             image: us.icr.io/<your SN labs namespace>/myapp:v1
             imagePullPolicy: Always
             ports:
               - containerPort: 3000
                 name: http
             resources:
               limits:
                 cpu: 50m
               requests:
                 cpu: 20m
   ```

2. **Replace `<your SN labs namespace>` with Your Actual SN Lab's Namespace:**

   **Ways to Get Your Namespace:**

   - Run the command `oc project`, and use the namespace mapped to your project name.
   - Run the command `ibmcloud cr namespaces` and use the one which shows your `sn-labs-username`.

3. **Apply the Deployment:**

   ```bash
   kubectl apply -f deployment.yaml
   ```

4. **Verify That the Application Pods Are Running and Accessible:**

   ```bash
   kubectl get pods
   ```

### Step 4: View the Application Output

1. **Start the Application on Port-Forward:**

   ```bash
   kubectl port-forward deployment.apps/myapp 3000:3000
   ```

2. **Launch the App on Port 3000:**

   Open your browser and navigate to `http://localhost:3000` to view the application output.

3. **Verify the Application Output:**

   You should see the message:

   ```
   Hello from MyApp. Your app is up!
   ```

4. **Stop the Server Before Proceeding Further:**

   Press `CTRL + C` in the terminal to stop the port-forwarding.

5. **Create a ClusterIP Service for Exposing the Application:**

   ```bash
   kubectl expose deployment/myapp
   ```

---

## Exercise 2: Implement Vertical Pod Autoscaler (VPA)

Vertical Pod Autoscaler (VPA) helps you manage resource requests and limits for containers running in a pod. It ensures pods have the appropriate resources to operate efficiently by automatically adjusting the CPU and memory requests and limits based on the observed resource usage.

### Step 1: Create a VPA Configuration

You will create a Vertical Pod Autoscaler (VPA) configuration to automatically adjust the resource requests and limits for the `myapp` deployment.

**Explore the `vpa.yaml` File:**

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myvpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"  # VPA will automatically update the resource requests and limits
```

**Explanation:**

- **updateMode: "Auto"** means that VPA will automatically update the resource requests and limits for the pods in this deployment based on the observed usage.

### Step 2: Apply the VPA

Apply the VPA configuration using the following command:

```bash
kubectl apply -f vpa.yaml
```

### Step 3: Retrieve the Details of the VPA

1. **Retrieve the Created VPA:**

   ```bash
   kubectl get vpa
   ```

   This output shows that:

   - The VPA named `myvpa` is in **Auto** mode.
   - It is providing recommendations for CPU and memory resources.

2. **Retrieve the Details and Current Running Status of the VPA:**

   ```bash
   kubectl describe vpa myvpa
   ```

**Explanation:**

The output provides recommendations for CPU and memory:

- **Lower Bound:** Minimum resources the VPA recommends.
- **Target:** Optimal resources the VPA recommends.
- **Uncapped Target:** Target without any predefined limits.
- **Upper Bound:** Maximum resources the VPA recommends.

**Resource Recommendations Example:**

| Resource | CPU  | Memory       |
|----------|------|--------------|
| Lower    | 25m  | 256MiB       |
| Target   | 25m  | 256MiB       |
| Upper    | 671m | 1.34GiB      |

---

## Exercise 3: Implement Horizontal Pod Autoscaler (HPA)

Horizontal Pod Autoscaler (HPA) automatically scales the number of pod replicas based on observed CPU/memory utilization or other custom metrics. While VPA adjusts the resource requests and limits for individual pods, HPA changes the number of pod replicas to handle the load.

### Step 1: Create an HPA Configuration

You will configure a Horizontal Pod Autoscaler (HPA) to scale the number of replicas of the `myapp` deployment based on CPU utilization.

**Explore the `hpa.yaml` File:**

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: myhpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 1        # Minimum number of replicas
  maxReplicas: 10       # Maximum number of replicas
  targetCPUUtilizationPercentage: 5  # Target CPU utilization for scaling
```

**Explanation:**

- The HPA will ensure that the average CPU utilization across all pods remains close to **5%**.
- It will scale the replicas between **1** and **10** based on CPU usage.

### Step 2: Configure the HPA

Apply the HPA configuration:

```bash
kubectl apply -f hpa.yaml
```

### Step 3: Verify the HPA

Obtain the status of the created HPA resource:

```bash
kubectl get hpa myhpa
```

This command provides details about the current and target CPU utilization and the number of replicas.

### Step 4: Start the Kubernetes Proxy

Open another terminal and start the Kubernetes proxy:

```bash
kubectl proxy
```

### Step 5: Generate Load on the App

Open a new terminal and enter the following command to generate load:

```bash
for i in `seq 100000`; do curl -L localhost:8001/api/v1/namespaces/sn-labs-$USERNAME/services/myapp/proxy; done
```

### Step 6: Observe the Effect of Autoscaling

1. **Watch the HPA Scaling:**

   ```bash
   kubectl get hpa myhpa --watch
   ```

2. **Observe the Increase in Replicas:**

   You will see an increase in the number of replicas, which shows that your application has been autoscaled.

3. **Terminate the Watch Command:**

   Press `CTRL + C` to stop watching.

### Step 7: Observe the Details of the HPA

1. **Get the HPA Details:**

   ```bash
   kubectl get hpa myhpa
   ```

2. **Notice the Number of Replicas Has Increased:**

   The output shows the increased number of replicas due to autoscaling.

3. **Stop the Proxy and Load Generation Commands:**

   In the other two terminals, press `CTRL + C` to stop the commands.

---

## Exercise 4: Create a Secret and Update the Deployment

Kubernetes Secrets let you securely store and manage sensitive information, such as passwords, OAuth tokens, and SSH keys. Secrets are base64-encoded and can be used in your applications as environment variables or mounted as files.

### Step 1: Create a Secret

**Explore the Content of the `secret.yaml` File:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
type: Opaque
data:
  username: bXl1c2VybmFtZQ==
  password: bXlwYXNzd29yZA==
```

**Explanation:**

- This YAML file defines a secret named `myapp-secret` with two key-value pairs: `username` and `password`.
- The values are base64-encoded strings.

### Step 2: Update the Deployment to Utilize the Secret

Add the following lines to the `deployment.yaml` file under the `containers` section:

```yaml
env:
  - name: MYAPP_USERNAME
    valueFrom:
      secretKeyRef:
        name: myapp-secret
        key: username
  - name: MYAPP_PASSWORD
    valueFrom:
      secretKeyRef:
        name: myapp-secret
        key: password
```

**Explanation:**

- **env:** Defines environment variables for the container.
- **valueFrom.secretKeyRef:** Indicates that the value of the environment variable should come from a Kubernetes secret.
- **name:** The name of the secret (`myapp-secret`).
- **key:** The specific key within the secret to use (`username` and `password`).

### Step 3: Apply the Secret and Deployment

1. **Apply the Secret:**

   ```bash
   kubectl apply -f secret.yaml
   ```

2. **Apply the Updated Deployment:**

   ```bash
   kubectl apply -f deployment.yaml
   ```

### Step 4: Verify the Secret and Deployment

1. **Retrieve the Details of the Secret:**

   ```bash
   kubectl get secret
   ```

   This shows the secrets in your cluster, including `myapp-secret`.

2. **Show the Status of the Deployment:**

   ```bash
   kubectl get deployment
   ```

   This shows the status of the deployment, including information about replicas and available replicas.

---

## Conclusion

In this lab, you:

- **Built and deployed** an application called `myapp` on Kubernetes.
- **Configured a Vertical Pod Autoscaler (VPA)** to automatically adjust resource requests and limits for the `myapp` deployment.
- **Implemented a Horizontal Pod Autoscaler (HPA)** to scale the number of replicas for the `myapp` deployment based on CPU utilization.
- **Created a Secret** and updated the `myapp` deployment to utilize it securely.

---

## Author(s)

**Nikesh Kumar**

© IBM Corporation. All rights reserved.
