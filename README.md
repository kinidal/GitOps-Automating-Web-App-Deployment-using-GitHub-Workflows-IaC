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
