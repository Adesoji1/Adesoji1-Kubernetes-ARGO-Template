# Project Title: Argo Workflows for SQL Stored Procedures on AWS EKS

This project sets up an Argo Workflow to call several SQL stored procedures both in parallel and sequentially on an AWS EKS cluster. The workflow involves checking if it's the end of the day, stopping services, taking backups, performing indexing, and starting services.

## Prerequisites

- AWS CLI
- kubectl
- eksctl
- Python (for encoding credentials)

## Files

- `encode_credentials.py`: Python script to encode the database credentials.
- `workflow.yaml`: YAML file containing the Argo Workflow definition.
- `create_eks_cluster.sh`: Shell script to create the EKS cluster and set up the environment.

## Steps

### Step 1: Encode Database Credentials

Create a Python script named `encode_credentials.py` to encode your database username and password:

```python
import base64

# Replace these with your actual username and password
username = "your-username"
password = "your-password"

encoded_username = base64.b64encode(username.encode()).decode()
encoded_password = base64.b64encode(password.encode()).decode()

print(f"Encoded username: {encoded_username}")
print(f"Encoded password: {encoded_password}")
```

Run the script to get the base64 encoded values:

```sh
python encode_credentials.py
```

### Step 2: Create the EKS Cluster

Create a shell script named `create_eks_cluster.sh` to set up the EKS cluster:

```sh
#!/bin/bash

# Create EKS cluster
eksctl create cluster --name my-cluster --region us-west-2 --nodegroup-name linux-nodes --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 4 --managed

# Update kubeconfig
aws eks --region us-west-2 update-kubeconfig --name my-cluster

# Verify the configuration
kubectl get nodes
```

Run the script to create the EKS cluster:

```sh
bash create_eks_cluster.sh
```

### Step 3: Install Argo Workflows

Install Argo Workflows in your Kubernetes cluster:

```sh
kubectl create namespace argo
kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo-workflows/stable/manifests/install.yaml
```

### Step 4: Create the Secret for DB Credentials

Create a Kubernetes secret with the encoded database credentials:

```sh
kubectl create secret generic db-credentials --from-literal=username=YWRtaW4= --from-literal=password=c2VjcmV0
```

Replace `YWRtaW4=` and `c2VjcmV0` with your actual base64 encoded username and password.

### Step 5: Submit the Argo Workflow

Save the following YAML content to a file named `workflow.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: sql-procedures-
spec:
  entrypoint: main
  templates:
  - name: main
    steps:
    - - name: check-end-of-day
        template: check-end-of-day
    - - name: stop-services
        template: stop-services
        when: "{{steps.check-end-of-day.outputs.result}} == true"
    - - name: backup-servers
        template: backup-servers
        when: "{{steps.stop-services.outputs.result}} == true"
    - - name: index-servers
        template: index-servers
        when: "{{steps.backup-servers.outputs.result}} == true"
    - - name: start-services
        template: start-services
        when: "{{steps.index-servers.outputs.result}} == true"

  - name: check-end-of-day
    script:
      image: python:3.8
      command: [python]
      source: |
        import datetime
        import pytz

        tz = pytz.timezone("{{workflow.parameters.timezone}}")
        now = datetime.datetime.now(tz)
        if now.weekday() < 5 and now.hour >= 17:  # Assuming end of the day is 5 PM on weekdays
            print("true")
        else:
            print("false")

  - name: stop-services
    script:
      image: postgres:13
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      command: [psql]
      args: ["-h", "{{workflow.parameters.db-host}}", "-U", "$(DB_USER)", "-c", "CALL {{workflow.parameters.stop-services-sp}}();"]
      resources:
        limits:
          memory: 128Mi
          cpu: 500m

  - name: backup-servers
    parallelism: 2
    steps:
    - - name: backup-server-1
        template: backup
        arguments:
          parameters:
            - name: server
              value: "{{workflow.parameters.db-host-1}}"
    - - name: backup-server-2
        template: backup
        arguments:
          parameters:
            - name: server
              value: "{{workflow.parameters.db-host-2}}"

  - name: backup
    inputs:
      parameters:
        - name: server
    script:
      image: postgres:13
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      command: [psql]
      args: ["-h", "{{inputs.parameters.server}}", "-U", "$(DB_USER)", "-c", "CALL {{workflow.parameters.backup-sp}}();"]
      resources:
        limits:
          memory: 128Mi
          cpu: 500m

  - name: index-servers
    parallelism: 2
    steps:
    - - name: index-server-1
        template: index
        arguments:
          parameters:
            - name: server
              value: "{{workflow.parameters.db-host-1}}"
    - - name: index-server-2
        template: index
        arguments:
          parameters:
            - name: server
              value: "{{workflow.parameters.db-host-2}}"

  - name: index
    inputs:
      parameters:
        - name: server
    script:
      image: postgres:13
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      command: [psql]
      args: ["-h", "{{inputs.parameters.server}}", "-U", "$(DB_USER)", "-c", "CALL {{workflow.parameters.index-sp}}();"]
      resources:
        limits:
          memory: 128Mi
          cpu: 500m

  - name: start-services
    script:
      image: postgres:13
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      command: [psql]
      args: ["-h", "{{workflow.parameters.db-host}}", "-U", "$(DB_USER)", "-c", "CALL {{workflow.parameters.start-services-sp}}();"]
      resources:
        limits:
          memory: 128Mi
          cpu: 500m

  arguments:
    parameters:
      - name: db-host
        value: "your-db-host"
      - name: db-host-1
        value: "your-db-host-1"
      - name: db-host-2
        value: "your-db-host-2"
      - name: timezone
        value: "UTC"
      - name: stop-services-sp
        value: "stop_services_procedure"
      - name: backup-sp
        value: "backup_procedure"
      - name: index-sp
        value: "index_procedure"
      - name: start-services-sp
        value: "start_services_procedure"
```

To encode your username and password in base64, you can use various tools or programming languages. Here's how you can do it using Python:

```python
import base64

# Replace these with your actual username and password
username = "your-username"
password = "your-password"

encoded_username = base64.b64encode(username.encode()).decode()
encoded_password = base64.b64encode(password.encode()).decode()

print(f"Encoded username: {encoded_username}")
print(f"Encoded password: {encoded_password}")
```

Running this script will give you the base64 encoded values of your username and password. Replace the placeholders `<base64_encoded_username>` and `<base64_encoded_password>` in the YAML file with the encoded values. For example, if your username is `admin` and password is `secret`, the encoded values would look something like this:

```yaml
username: YWRtaW4=
password: c2VjcmV0
```

So your secret section in the YAML would be:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=
  password: c2VjcmV0
```

### Final YAML Template

Here's the complete YAML template with the encoded username and password:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: sql-procedures-
spec:
  entrypoint: main
  templates:
  - name: main
    steps:
    - - name: check-end-of-day
        template: check-end-of-day
    - - name: stop-services
        template: stop-services
        when: "{{steps.check-end-of-day.outputs.result}} == true"
    - - name: backup-servers
        template: backup-servers
        when: "{{steps.stop-services.outputs.result}} == true"
    - - name: index-servers
        template: index-servers
        when: "{{steps.backup-servers.outputs.result}} == true"
    - - name: start-services
        template: start-services
        when: "{{steps.index-servers.outputs.result}} == true"

  - name: check-end-of-day
    script:
      image: python:3.8
      command: [python]
      source: |
        import datetime
        import pytz

        tz = pytz.timezone("{{workflow.parameters.timezone}}")
        now = datetime.datetime.now(tz)
        if now.weekday() < 5 and now.hour >= 17:  # Assuming end of the day is 5 PM on weekdays
            print("true")
        else:
            print("false")

  - name: stop-services
    script:
      image: postgres:13
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      command: [psql]
      args: ["-h", "{{workflow.parameters.db-host}}", "-U", "$(DB_USER)", "-c", "CALL {{workflow.parameters.stop-services-sp}}();"]
      resources:
        limits:
          memory: 128Mi
          cpu: 500m

  - name: backup-servers
    parallelism: 2
    steps:
    - - name: backup-server-1
        template: backup
        arguments:
          parameters:
            - name: server
              value: "{{workflow.parameters.db-host-1}}"
    - - name: backup-server-2
        template: backup
        arguments:
          parameters:
            - name: server
              value: "{{workflow.parameters.db-host-2}}"

  - name: backup
    inputs:
      parameters:
        - name: server
    script:
      image: postgres:13
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      command: [psql]
      args: ["-h", "{{inputs.parameters.server}}", "-U", "$(DB_USER)", "-c", "CALL {{workflow.parameters.backup-sp}}();"]
      resources:
        limits:
          memory: 128Mi
          cpu: 500m

  - name: index-servers
    parallelism: 2
    steps:
    - - name: index-server-1
        template: index
        arguments:
          parameters:
            - name: server
              value: "{{workflow.parameters.db-host-1}}"
    - - name: index-server-2
        template: index
        arguments:
          parameters:
            - name: server
              value: "{{workflow.parameters.db-host-2}}"

  - name: index
    inputs:
      parameters:
        - name: server
    script:
      image: postgres:13
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      command: [psql]
      args: ["-h", "{{inputs.parameters.server}}", "-U", "$(DB_USER)", "-c", "CALL {{workflow.parameters.index-sp}}();"]
      resources:
        limits:
          memory: 128Mi
          cpu: 500m

  - name: start-services
    script:
      image: postgres:13
      env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      command: [psql]
      args: ["-h", "{{workflow.parameters.db-host}}", "-U", "$(DB_USER)", "-c", "CALL {{workflow.parameters.start-services-sp}}();"]
      resources:
        limits:
          memory: 128Mi
          cpu: 500m

  arguments:
    parameters:
      - name: db-host
        value: "your-db-host"
      - name: db-host-1
        value: "your-db-host-1"
      - name: db-host-2
        value: "your-db-host-2"
      - name: timezone
        value: "UTC"
      - name: stop-services-sp
        value: "stop_services_procedure"
      - name: backup-sp
        value: "backup_procedure"
      - name: index-sp
        value: "index_procedure"
      - name: start-services-sp
        value: "start_services_procedure"
```

To run the Kubernetes workflow, you'll need to follow these steps:

1. **Set up a Kubernetes Cluster**: You can use a managed Kubernetes service like AWS EKS, Google Kubernetes Engine (GKE), Azure Kubernetes Service (AKS), or set up a local cluster using Minikube or kind (Kubernetes in Docker).

2. **Install Argo Workflows**: Ensure you have Argo Workflows installed in your Kubernetes cluster. You can install Argo Workflows by running:

   ```sh
   kubectl create namespace argo
   kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo-workflows/stable/manifests/install.yaml
   ```

3. **Create the Secret**: Create the Kubernetes secret containing your database credentials:

   ```sh
   kubectl create secret generic db-credentials \
     --from-literal=username=YWRtaW4= \
     --from-literal=password=c2VjcmV0
   ```

   Replace `YWRtaW4=` and `c2VjcmV0` with the actual base64 encoded username and password.

4. **Submit the Workflow**: Save the workflow YAML to a file (e.g., `workflow.yaml`) and submit it to the Argo Workflows controller:

   ```sh
   kubectl apply -f workflow.yaml
   ```

5. **Monitor the Workflow**: You can monitor the workflow execution using the Argo Workflows UI or by checking the workflow status via `kubectl`:

   ```sh
   kubectl get workflows
   ```

   To view detailed information about a specific workflow:

   ```sh
   kubectl get workflow <workflow-name> -o yaml
   ```

   Alternatively, you can access the Argo Workflows UI by port-forwarding the Argo UI service to your local machine:

   ```sh
   kubectl -n argo port-forward deployment/argo-server 2746:2746
   ```

   Then, open your browser and go to `http://localhost:2746`.

### Example Commands

Here's a summary of the commands you might run:

1. **Set up Argo Workflows**:
   ```sh
   kubectl create namespace argo
   kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo-workflows/stable/manifests/install.yaml
   ```

2. **Create the Secret**:
   ```sh
   kubectl create secret generic db-credentials \
     --from-literal=username=YWRtaW4= \
     --from-literal=password=c2VjcmV0
   ```

3. **Submit the Workflow**:
   ```sh
   kubectl apply -f workflow.yaml
   ```

4. **Monitor the Workflow**:
   ```sh
   kubectl get workflows
   kubectl get workflow <workflow-name> -o yaml
   kubectl -n argo port-forward deployment/argo-server 2746:2746
   ```

Now, the project setup is complete. Here's what you should expect to see in the output for each step and in the Argo Workflow UI:

### Step-by-Step Outputs

1. **Creating the EKS Cluster**:
   - The `eksctl create cluster` command will output the progress of cluster creation. It includes creating VPC, subnets, security groups, EKS control plane, and node group. The final output will confirm the successful creation of the cluster.

2. **Configuring kubectl**:
   - The `aws eks --region us-west-2 update-kubeconfig --name my-cluster` command should complete without errors.
   - Running `kubectl get nodes` should list the nodes in your EKS cluster, indicating that kubectl is properly configured to communicate with the cluster.

   Example output:
   ```sh
   NAME                                            STATUS   ROLES    AGE   VERSION
   ip-192-168-xx-xxx.us-west-2.compute.internal   Ready    <none>   10m   v1.21.5-eks-bc4871b
   ip-192-168-xx-yyy.us-west-2.compute.internal   Ready    <none>   10m   v1.21.5-eks-bc4871b
   ```

3. **Installing Argo Workflows**:
   - The `kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo-workflows/stable/manifests/install.yaml` command should show the creation of various Kubernetes resources (Deployments, Services, etc.).
   - Running `kubectl get pods -n argo` should show several Argo Workflows pods in the `Running` state.

   Example output:
   ```sh
   NAME                                       READY   STATUS    RESTARTS   AGE
   argo-server-xxxxxxxxxx-xxxxx               1/1     Running   0          2m
   workflow-controller-xxxxxxxxxx-xxxxx       1/1     Running   0          2m
   ```

4. **Creating the Secret for DB Credentials**:
   - The `kubectl create secret generic db-credentials --from-literal=username=YWRtaW4= --from-literal=password=c2VjcmV0` command should confirm the creation of the secret.

5. **Submitting the Argo Workflow**:
   - The `kubectl apply -f workflow.yaml` command should confirm the creation of the workflow.

   Example output:
   ```sh
   workflow.argoproj.io/sql-procedures-xxxxx created
   ```

### Monitoring the Workflow

1. **List Workflows**:
   - Running `kubectl get workflows -n argo` should show your submitted workflow with its current status.

   Example output:
   ```sh
   NAME                       STATUS      AGE
   sql-procedures-xxxxx       Running     30s
   ```

2. **View Workflow Details**:
   - Running `kubectl get workflow <workflow-name> -n argo -o yaml` will show detailed information about the workflow execution, including the status of each step.

3. **Argo UI** (Optional):
   - Accessing the Argo UI by port-forwarding and going to `http://localhost:2746` will provide a graphical interface to monitor your workflow. You'll see a DAG (Directed Acyclic Graph) representation of your workflow steps, where you can observe the progress and status of each step in real-time.

### Expected Outputs in the Workflow

- **Check End of Day**: The script will output `true` if the current time matches the criteria, otherwise `false`.
- **Stop Services**: Should indicate successful execution of the stored procedure or failure if it encounters any issues.
- **Backup Servers**: Parallel execution of the backup stored procedure on two servers. Each step should show success or failure.
- **Index Servers**: Parallel execution of the indexing stored procedure on two servers. Each step should show success or failure.
- **Start Services**: Should indicate successful execution of the stored procedure or failure if it encounters any issues.

### Example Final Status in Argo UI

- A successful workflow will show all steps in green, indicating that each step completed successfully.
- Any failure will be indicated in red, with details on the specific step that failed.

By monitoring the output and status of each step, you'll be able to verify that the workflow is executing correctly and identify any issues that need to be addressed.































Submit the workflow:

```sh
kubectl apply -f workflow.yaml
```

### Step 6: Monitor the Workflow

1. List workflows:

   ```sh
   kubectl get workflows -n argo
   ```

2. View workflow details:

   ```sh
   kubectl get workflow <workflow-name> -n argo -o yaml
   ```

3. Access the Argo UI (optional):

   ```sh
   kubectl -n argo port-forward deployment/argo-server 2746:2746
   ```

   Open your browser and go to `http://localhost:2746`.










Setting up a Kubernetes cluster using AWS Elastic Kubernetes Service (EKS) involves several steps. Below is a detailed guide to help you get started:

### Prerequisites

1. **AWS CLI**: Make sure you have the AWS CLI installed and configured. You can install it by following the instructions [here](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html).
   
2. **kubectl**: Install `kubectl`, the Kubernetes command-line tool. You can find the installation instructions [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
   
3. **eksctl**: This is a simple CLI tool for creating and managing EKS clusters. Install `eksctl` by following the instructions [here](https://eksctl.io/).

### Step 1: Create an EKS Cluster

1. **Create the EKS Cluster**: Use `eksctl` to create your EKS cluster. This command creates a cluster with one node group.

   ```sh
   eksctl create cluster --name my-cluster --region us-west-2 --nodegroup-name linux-nodes --node-type t3.medium --nodes 3 --nodes-min 1 --nodes-max 4 --managed
   ```

   This command does the following:
   - Creates an EKS cluster named `my-cluster`.
   - Sets the region to `us-west-2`.
   - Creates a managed node group named `linux-nodes` with `t3.medium` instances.
   - Sets the initial number of nodes to 3, with a minimum of 1 and a maximum of 4.

2. **Wait for the Cluster to be Ready**: This process takes several minutes. `eksctl` will create the necessary AWS resources, such as VPC, subnets, and security groups.

### Step 2: Configure kubectl to Connect to Your Cluster

After the cluster is created, configure `kubectl` to connect to your EKS cluster.

1. **Update kubeconfig**:

   ```sh
   aws eks --region us-west-2 update-kubeconfig --name my-cluster
   ```

2. **Verify the Configuration**: Check if your `kubectl` configuration is correct by listing the nodes in your cluster.

   ```sh
   kubectl get nodes
   ```

   You should see a list of nodes that are part of your EKS cluster.

### Step 3: Install Argo Workflows

1. **Create a Namespace for Argo**:

   ```sh
   kubectl create namespace argo
   ```

2. **Install Argo Workflows**:

   ```sh
   kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo-workflows/stable/manifests/install.yaml
   ```

3. **Verify the Installation**:

   ```sh
   kubectl get pods -n argo
   ```

   You should see several pods running.

### Step 4: Create the Secret for DB Credentials

1. **Create the Secret**:

   ```sh
   kubectl create secret generic db-credentials --from-literal=username=YWRtaW4= --from-literal=password=c2VjcmV0
   ```

   Replace `YWRtaW4=` and `c2VjcmV0` with your actual base64 encoded username and password.

### Step 5: Submit the Argo Workflow

1. **Save Your Workflow YAML**: Save the workflow YAML file provided earlier to a file named `workflow.yaml`.

2. **Apply the Workflow**:

   ```sh
   kubectl apply -f workflow.yaml
   ```

### Step 6: Monitor the Workflow

1. **List Workflows**:

   ```sh
   kubectl get workflows -n argo
   ```

2. **View Workflow Details**:

   ```sh
   kubectl get workflow <workflow-name> -n argo -o yaml
   ```

3. **Access the Argo UI** (optional):

   ```sh
   kubectl -n argo port-forward deployment/argo-server 2746:2746
   ```

   Open your browser and go to `http://localhost:2746`.

