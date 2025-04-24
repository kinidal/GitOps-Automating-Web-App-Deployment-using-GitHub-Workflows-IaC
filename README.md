# GitOps-Automating-Web-App-Deployment-using-GitHub-Workflows-IaC

## 📘 Overview

In today’s cloud-native world, we needed a **robust, repeatable, and auditable way** to manage both infrastructure and applications. This project implements a **GitOps approach** using GitHub Actions to automate:

- **Infrastructure provisioning** with **Terraform**
- **Application deployment** using **Docker**, **Helm**, and **AWS EKS**

The goal is to remove manual intervention, reduce human errors, and embrace full version control across both infrastructure and applications using GitOps principles.

---

## 📂 Table of Contents

1. [⚙️ Technology Stack](#️-technology-stack)
2. [📐 Architecture](#-architecture)
3. [🏗️ Infrastructure Workflow](#️-infrastructure-workflow)
4. [📦 Application Workflow](#-application-workflow)
5. [✅ Conclusion](#-conclusion)

---

## ⚙️ Technology Stack

| Layer                       | Tools Used                   | Reasoning                                                                 |
|-----------------------------|------------------------------|---------------------------------------------------------------------------|
| Version Control & CI/CD      | Git, GitHub, GitHub Actions  | GitOps core tools for declarative, automated pipelines                     |
| Infrastructure as Code       | Terraform                    | Modularity, remote state, and strong AWS support                          |
| Cloud Platform               | AWS (IAM, S3, EKS, ECR)      | Scalable, production-grade cloud services                                 |
| Containerization             | Docker                       | Ensures consistency across environments                                   |
| Code Quality                 | Maven, SonarCloud            | Build/test Java apps and enforce code quality via quality gates           |
| Deployment                   | Helm                         | Declarative and rollback-safe Kubernetes deployments                      |

---

## 📐 Architecture

![image](https://github.com/user-attachments/assets/c4832c68-2fa8-4470-86bb-5b7c03fbda98)

There are two GitHub repositories: one for Infrastructure as Code (IaC) and the other for application development.

For the Infrastructure as Code repository, whenever a change is made to the IaC code in the staging branch, a GitHub Actions workflow is triggered. This workflow executes a Terraform plan and validates the proposed changes against the current infrastructure architecture. At this stage, only the Terraform plan and validation are performed.
If the changes are deemed appropriate, a pull request is created to merge the updates into the main branch. Upon approval, the IaC code is applied to the infrastructure using Terraform apply, completing the infrastructure-related workflow.

For the application development repository, any code changes trigger a separate GitHub Actions workflow. The first step in this workflow is fetching the staging branch, followed by building, testing, and deploying the application using Maven, SonarCloud, and Docker. Once the Docker image is built, it is pushed to Amazon ECR.
Finally, the Helm charts, which contain variables for the image name and tag, specify the image location and version. The Helm charts are executed within the Kubernetes (K8s) cluster to deploy the application.



## 🏗️Setup and Configuration of - Infrastructure Workflow

## 🔧 Setup

### 1. Clone the GitHub Repositories
Clone both the Infrastructure as Code (IaC) and Application Development code repositories to your local system.

### 2. Create AWS Resources for IaC
After cloning the repositories, create the required AWS resources for the IaC setup, including:
- IAM User
- Storage Account
- Amazon ECR

### 3. Create IAM User in AWS
- In AWS, create an IAM user with administrator privileges (which can be adjusted to adhere to the principle of least privilege). Console access is not necessary.
- Generate an Access Key for CLI access (Note: The Access Key should not be downloaded to avoid security risks, as it will be stored securely as a secret in GitHub).
- Copy the Access Key and Access Key ID.

### 4. Store AWS Secrets in GitHub
- Navigate to the GitHub repository for the IaC code.
- Go to **Settings → Secrets and variables → Actions → New repository secret**.
- Create a new secret named `AWS_SECRET_ACCESS_KEY` and paste the Access Key value.
- Create another secret named `AWS_ACCESS_KEY_ID` and paste the Access Key ID.

Repeat the same steps for the Application Development code repository.

### 5. Create S3 Bucket for Terraform State
- In AWS, create an S3 bucket to store Terraform state files. Ensure you note the region selected for the bucket
- This ensures state consistency across teams and prevents race conditions during Terraform runs, which is crucial when multiple users are working on infrastructure.

### 6. Store S3 Bucket Secrets in GitHub
- In the IaC repository on GitHub, create a new secret variable to store the S3 bucket information. This secret will be used by Terraform to store state and run commands like `apply`, `test`, and `validate`.

### 7. Create a Repository in Amazon ECR
- In AWS, create a repository in Amazon ECR (ensure the region matches the one used for other resources).
- Once the repository is created, copy the repository URI.
- In the IaC GitHub repository, create a new secret with the repository URI value, ensuring to remove the repository name from the URI (i.e., exclude the part ending with `amazonaws.com`).

### 8. Update Terraform Code in GitHub Repository
- The IaC repository contains Terraform files for creating infrastructure in AWS, located in the `Terraform` folder. Key resources to create include the EKS Cluster and VPC.
- The Terraform code also references the S3 bucket for storing state files.
- Update the Terraform code in the GitHub repository to reflect necessary values (e.g., names, regions) that may vary depending on the resources being created. Ensure the updates are made in the **staging** branch.

### 9. Configure GitHub Actions Workflows
- In the locally cloned IaC repository (staging branch), navigate to the parent directory (where the `Terraform` folder is located).
- Create a new folder named `.github/workflows` in this directory.
- Workflow files in this folder, with a `.yml` extension, will define the GitHub Actions workflows.

Each `.yml` file created in this folder will trigger a GitHub Actions workflow upon specific events in the repository.

## 🔧 Configuration: Terraform GitHub Actions Workflow

### Create `terraform.yml` in the `.github/workflows` directory and Define Trigger, Environment, Jobs, etc.
The workflow will be triggered under the following conditions:
- Any changes made in the `Terraform` folder (which contains files like `vpc.yml`, `eks-cluster.yml`, etc.) will initiate the workflow.
- The workflow will also trigger when there is a pull request to the `main` branch.

### Environment Variables
Environment variables, which are already stored as secrets in GitHub and referenced in Terraform files, will be used in this workflow.

### Jobs
There is only one job in this workflow: applying the changes from the Terraform code. To run this job, we need a runner. GitHub provides runners (e.g., `ubuntu-latest`) with pre-installed software.

### Working Directory
We define the working directory as `./terraform`, where all the resource creation `.yml` files are located.

### Steps in the Job
The following tasks are defined as steps in the job:

![image](https://github.com/user-attachments/assets/127cae35-a973-4f66-a715-bf0aaf0b1dd9)

1. **Checkout the Code**
   - Even though the code exists in GitHub, it must be checked out into the runner.

2. **Install Terraform**
   - Use the predefined GitHub action `uses` to install Terraform on the runner.

3. **Terraform Init**
   - Initialize Terraform and define the state file, which will be stored in the S3 bucket. 
   - The `id` is used to reference the step in subsequent steps. For example, we can check whether the outcome of a step was successful.

4. **Terraform Format Check**
   - This step will format and check the Terraform code. If an error is found, it will return a non-zero exit code and fail the workflow.

5. **Terraform Validate**
   - This step checks the syntax of the Terraform code

6. **Terraform Plan**
   - This step checks which resources need to be created or updated. The output of this command is stored in a `planfile`. This file is used in the `terraform apply` command to apply the changes.
   - The `continue-on-error: true` ensures the workflow reaches the end of the steps, allowing for a clean exit, even if errors occur.
   - The `Terraform plan status` step checks if there are any errors, and if so, it forces the runner to exit.

![image](https://github.com/user-attachments/assets/d09430ba-e611-4e8b-ab06-64209c2a8611)


### Testing the Workflow
- You can test the workflow by making a dummy change in the `ecs.yml` (or any other file) in the `Terraform` folder and committing it. The workflow will not create any resources but will only run `terraform plan` and `terraform validate` to show the resources that will be created during `terraform apply`.

### Apply Changes to AWS Infrastructure
Once the `terraform.yml` workflow file is in place, we can add the `terraform apply` commands to apply changes to the AWS infrastructure.

1. **Terraform Apply Job**
   - Add a job for applying changes when a push happens to the `main` branch of the repository.
   - Run the `terraform apply` command with the `auto-approve` option, and set `parallelism` to 1. The output is referenced from the previously created `planfile`.

2. **Install Nginx Ingress Controller on Kubernetes (K8s) Cluster**
   - The next workflow step is to install an Nginx Ingress Controller on the K8s cluster, which will be used for application deployment.
   - To install the ingress controller, we need to run the `kubectl apply` command. To get the necessary `kubeconfig` file, we need to run an AWS CLI command. The `aws cli` command requires the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as inputs.

3. **Run AWS CLI Command**
   - The `kubeconfig` file is generated by running the `aws cli` command, which requires passing the secret access key and access key ID.

![image](https://github.com/user-attachments/assets/86a0dad7-a54f-4685-b5c4-27e73ff4d030)

### Merging Changes and Triggering the Workflow
The condition for applying the changes is if there is any change in the `main` branch of the IaC code repository. To trigger this workflow, merge the changes from the `staging` branch to the `main` branch.

### Set Branch Protection in GitHub
- Go to the repository in GitHub → **Settings → Branches → Add branch protection rule**.
- Select **Require pull request reviews before merging** to ensure no unchecked merges occur from staging to main.

After merging the changes, the GitHub Actions workflow will be triggered automatically and will create resources in AWS.

### Validating Resources Created in AWS Cloud
After the workflow completes, verify that the following resources have been created in AWS:
- **AWS EKS** with two node groups (Check EC2 instances as nodes).
- **VPC** with EKS public and private subnets, along with a single NAT gateway.
- **Elastic IP**.
- **Nginx Ingress Controller (Load Balancer)**.

---

This markdown format is ready to be pasted into your `README.md` file for GitHub. It provides a structured overview of your workflow setup and deployment process.


---

## 🛠️ Setup and Configuration of - App deployment Workflow

## Step 1: Create a SonarCloud Project

1. Log in to [SonarCloud.io](https://sonarcloud.io) using your GitHub account.
2. Create a new project and assign it a meaningful name.
3. After creation, **note the project key** — this will be needed for later configuration.

---

## Step 2: Generate a SonarCloud Authentication Token

1. Navigate to:  
   `My Account` → `Security` → **Generate a Token**  
   Name the token appropriately and copy it immediately.
2. In your **GitHub repository**, go to:  
   `Settings` → `Secrets and variables` → `Actions` → **New repository secret**
3. Create a new secret to store the SonarCloud token.  
   > Example Name: `SONAR_TOKEN`

---

## Step 3: Set Up a Quality Gate in SonarCloud

1. Go to your **SonarCloud Organization**.
2. Navigate to:  
   `Quality Gates` → **Create** → Provide a name.
3. Add a new condition:
   - **Condition Type**: Overall Code
   - **Metric**: Bugs
   - **Threshold**: Greater than 50
4. Associate this quality gate with your project.

---

## Step 4: Configure Additional Secrets in GitHub

Create the following secrets in your GitHub repository:

| Secret Name        | Description                          |
|--------------------|--------------------------------------|
| `SONAR_ORG`        | Your SonarCloud organization name    |
| `SONAR_PROJECT_KEY`| The project key from SonarCloud      |
| `SONAR_URL`        | The base URL (`https://sonarcloud.io`) |

---

## Step 5: Repository Structure

Ensure your application development repository contains the following components:

- `Dockerfile`: Builds a Java application using Maven and runs it on Tomcat.
- `k8s/`: Contains Kubernetes deployment manifests and ingress configuration for the application domain (host).

---

## Step 6: GitHub Actions Workflow Setup

1. Inside the root of your repository, create the following directory:
   ```bash
   mkdir -p .github/workflows
2. Create a main.yml file, which is the workflow file for the application development.

## `main.yml` Workflow Overview

![image](https://github.com/user-attachments/assets/dd488b19-ba89-4424-b6a7-bebb5904a4e0)
![image](https://github.com/user-attachments/assets/0c4b1863-ffc1-4bed-96bd-dad7466cc554)
![image](https://github.com/user-attachments/assets/feea53c0-6a01-446e-b25a-a9f56266e7e6)



The workflow includes two main stages:

### 1. Code Analysis  
This stage focuses on testing and analyzing the codebase using Maven and SonarCloud.

**Key Steps:**
- **Trigger:** Manually triggered using `workflow_dispatch`.
- **Environment Variables:** Loaded securely via GitHub Secrets.
- **Runner:** Utilizes `ubuntu-latest` for job execution.
- **Steps include:**
  - Code checkout.
  - Running Maven tests and Checkstyle analysis.
  - Installing Java 11.
  - Configuring and authenticating with SonarCloud CLI.
  - Uploading analysis results and enforcing quality gate conditions.

---

### 2. Docker Build & Push  
After successful code analysis, this stage builds the Docker image and pushes it to AWS ECR.

**Key Steps:**
- This job is **dependent** on the previous testing job using the `needs` keyword.
- Specifies the `ubuntu-latest` runner and checks out the code.
- Uses the **predefined GitHub Action** `Docker ECR` for building and deploying the image.
- Sets environment variables for:
  - AWS CLI authentication.
  - ECR registry and region configuration.
- Tags the Docker image with:  
  `latest.<GitHub build ID>`
- Defines the Dockerfile path and build context.

---

After the workflow completes successfully, login to aws to verify that the Docker image has been pushed to AWS ECR.

## Step 7: Deploy to EKS

Now we will deploy the application to an Elastic Kubernetes Cluster in Amazon Web Services (AWS). The deployment process begins with configuring Helm charts to manage Kubernetes resources.

### Configure Helm Charts
To make the container image dynamic, we replace the hardcoded image name and tag in the Kubernetes definition file with a variable. This allows us to update the image each time the Kubernetes cluster is deployed. We use Helm charts to pass this variable value, which is dynamically set through the GitHub workflow.

### Local Setup
1. Install Helm on your local system where the GitHub code is checked out.
2. Create a `helm` directory and initialize it with default files.
3. Replace the default Kubernetes definition files with your project-specific files.
4. In the `helm/{project-name}/templates` folder, modify the Kubernetes definition files. Specifically, replace the hardcoded image and tag with variables, which will be populated dynamically through the GitHub workflow.
   
   _Example:_
   ```yaml
   image:
     repository: {{ .Values.image.repository }}
     tag: {{ .Values.image.tag }}

### Configure the GitHub Workflow
Next, update the `main.yml` file located in `.github/workflows` to configure the deployment process:

**Setup Deployment Job**  
   Define a new job for deploying to the Kubernetes cluster. Specify a new runner and ensure that the job executes the deployment steps.

   ![image](https://github.com/user-attachments/assets/f8396784-bee7-4f22-aea3-a8c863854c1c)

**Checkout Code & Configure AWS Credentials**  
   The workflow will first check out the code and configure AWS credentials. This is essential for using AWS CLI commands in the subsequent steps to interact with AWS resources.

**Kubeconfig Setup**  
   Retrieve the `kubeconfig` file, which is necessary for configuring `kubectl` to interact with the Kubernetes cluster. This file contains all the configuration needed to communicate with your cluster securely.

**Login to ECR**  
   Use the `kubectl` command to create a secret for logging into AWS Elastic Container Registry (ECR). This secret will use credentials stored in GitHub Secrets, such as the ECR registry URI, default ECR username, and a generated password, which will be securely retrieved during the workflow run and these values will be stored in the k8s secrets as well.

**Run Helm Chart**  
   After configuring the Kubernetes secrets, execute the Helm chart to deploy the application to your Kubernetes cluster. The values for the image (image name and tag) will be dynamically set via variables in the `values.yaml` file, ensuring that the latest image is used for deployment every time the workflow runs.

![image](https://github.com/user-attachments/assets/9c1c4a25-b35c-461a-80e7-ea88cf9677cb)


## Step 8: Validate the Deployment
Once the GitHub workflow completes successfully, verify the deployment by following these steps:

1. **Check the Ingress Load Balancer**  
   Ensure that the load balancer is still running by accessing the ingress. This will help confirm that the service is exposed and the traffic is correctly routed to your application.

2. **Test Application with Custom Domain**  
   Update the DNS to point to a custom domain using a CNAME record. Verify that the application is accessible and functioning as expected when accessed through the custom domain.

3. **Confirm EKS Deployment**  
   If everything is working correctly, it means that the EKS cluster is pulling the latest image from ECR, and the application is running as a service through Kubernetes pods. This confirms the successful deployment and execution of your application in the cluster.

