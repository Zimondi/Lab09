# **Lab 09 Practice Report – GitLab CI/CD Pipeline Crash Course**

## **1. Introduction**
Continuous Integration and Continuous Deployment, or CI/CD, automates application development, testing, and release.  This experiment uses Python (Flask), Docker, and GitLab CI/CD to show a safe, repeatable pipeline.

## **2. Steps Performed – 16 Required Steps**

### **Step 1: Clone the GitLab CI/CD Tutorial Project**
#### **What was done:**
Cloned the official GitLab crash course repository into my local development environment.
#### **Commands used:**
```bash
git clone https://gitlab.com/nanuchi/gitlab-cicd-crash-course.git

cd gitlab-cicd-crash-course
```
#### **Key decision:**
Used GitLab instead of GitHub to leverage GitLab's built-in CI/CD.

### **Step 2: Explore the Project Structure**
#### **What was done:**
Inspected project files and folders to understand the structure and prepare for pipeline creation.
#### **Command used:**
`ls`

**Some of the files discovered:**
- app.py
- Dockerfile
- requirements.txt
- gitlab-ci.yml
#### **Key decision:**
Used the existing structure without modification to ensure compatibility.

### **Step 3: Run the Tests Locally**
#### **What was done:**
Installed dependencies and ran unit tests locally to validate app behavior before pushing to CI/CD. To test and run the app locally, I used the following commands and the app was launched on http://127.0.0.1:5000:
#### **Commands used:**
- make test
- make run
#### **Key decision:**
Confirmed the code works locally to isolate issues before GitLab pipeline testing.

### **Step 4: Identify All Project Dependencies**
#### **What was done:**
Checked `requirements.txt` for all Python libraries used in the app.
#### **Command used:**
`cat requirements.txt`

**Dependencies:**
- Flask==2.1.0
- py-cpuinfo==7.0.0
- psutil==5.8.0
- gunicorn==20.1.0
- black==20.8b1
- flake8==3.9.0
- pytest==6.2.2
#### **Key decision:**
I used the `requirements.txt` file instead of manually listing packages from the code, since it’s the official and standard way to define Python dependencies. This also aligns with how they’re installed in Docker and CI/CD environments.

### **Step 5: Set and Document the Application Port**
#### **What was done:**
Verified the app listens on port 5000.
#### **Commands used:**
`cat Dockerfile`

#### **Key decision:**
Kept port as-is for consistency with Docker exposure later.

### **Step 6: Show the App Running Locally**
#### **What was done:**
I launched the app locally using Docker and verified it was accessible on http://localhost:5000.
#### **Commands used:**
`docker run -d -p 5000:5000 nanuchi/demo-app:python-app-1.0`

#### **Key decision:**
Tested locally before containerizing to ensure functionality.

### **Step 7: Create .gitlab-ci.yml File to Run the Tests**
#### **What was done:**
Created a CI file that uses a Python image, installs dependencies, and runs tests.
```yaml
variables:
  IMAGE_NAME: ahmedsiddig1/demo-app
  IMAGE_TAG: python-app-1.0

stages:
  - test
  - build
  - deploy

run_tests:
  stage: test
  image: python:3.9-slim
  before_script:
    - apt-get update && apt-get install make
  script:
    - make test

build_image:
  stage: build
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $REGISTRY_USER -p $REGISTRY_PASS
  script:
    - docker build -t $IMAGE_NAME:$IMAGE_TAG .
    - docker push $IMAGE_NAME:$IMAGE_TAG

deploy:
  stage: deploy
  before_script:
    - chmod 400 $SSH_KEY
  script:
    - ssh -o StrictHostKeyChecking=no -i $SSH_KEY ahmedsiddig@128.203.114.140 "
        docker login -u $REGISTRY_USER -p $REGISTRY_PASS &&
        docker ps -aq | xargs docker stop | xargs docker rm &&
        docker run -d -p 5000:5000 $IMAGE_NAME:$IMAGE_TAG"
```
#### **Key decision:
Used make test to run tests and for consistency.**

### **Step 8: Build and Push a Docker Image to DockerHub Securely**
#### **What was done:**
Configured the GitLab CI pipeline to build the Docker image and push it to DockerHub using secure credentials stored as GitLab CI variables.
#### **Key decision:**
Used Docker-in-Docker (docker:dind) service for building and pushing inside CI; ensured secrets were never exposed in logs.

### **Step 9: Execute the Full GitLab Pipeline**
#### **What was done:**
Triggered the GitLab CI/CD pipeline by pushing code changes to the repository. This started the defined pipeline stages including testing, building, and pushing the Docker image automatically.
#### **Commands used:**
```bash
git add .
git commit -m
git push origin main
```
#### **Key decision:**
Used GitLab’s web UI to closely monitor each pipeline stage. Broke pipeline into multiple stages (test, build) for easier troubleshooting. Repeated pipeline runs to confirm fixes and successful execution.

### **Step 10: Show the Image Was Built and Pushed**
#### **What was done:**
Verified that the Docker image was successfully built in the CI pipeline and pushed to DockerHub.

**Verification steps:**
- Checked the GitLab pipeline job logs for successful docker build and docker push commands without errors.
- Logged into DockerHub to confirm the new image appeared in the repository with the latest timestamp.
- Used docker pull ahmedsiddig1/demo-app:latest locally to confirm the image can be downloaded

### **Step 11: Configure Pipeline Stages and Explain Them**
#### **What was done:**
Configured multiple stages in .gitlab-ci.yml to organize the CI/CD process into test, build, and deploy steps for a complete workflow.

**Pipeline stages configured:**
- test: Run automated tests to verify code quality.
- build: Build the Docker image and push it to DockerHub.
- deploy: Deploy the Docker container to the cloud server using SSH.

**Explanation:**
- Test: Validates code correctness before any build or deployment.
- Build: Packages the app into a Docker image only if tests pass.
- Deploy: Automatically deploys the verified image to the cloud server, enabling continuous delivery without manual steps.
#### **Key decision:**
- Adding a deploy stage completes the pipeline, automating the full path from code commit to live app.
- Ensures only tested and built code is deployed, improving security and reliability.
- Using SSH keys in deploy stage keeps credentials secure during remote deployment.

### **Step 12: Set Up a Cloud Deployment Server Using SSH Keys**
#### **What was done:**
Provisioned an Azure Virtual Machine (VM) and configured SSH key-based authentication to enable secure automated deployment from GitLab.

**Steps performed:**
- Created an Azure Linux VM via Azure Portal or Azure CLI, specifying SSH public key authentication during setup
- Generated an SSH key pair locally.
- Uploaded the public key to the Azure VM’s ~/.ssh/authorized_keys automatically during creation or manually afterward.
- Verified SSH access to the VM without a password.
- Added the private SSH key to GitLab CI/CD variables **(SSH_KEY)** as a protected secret for secure pipeline use.
#### **Commands used:**
```bash
ssh-keygen
ssh-copy-id ahmedsiddig@128.203.114.140
```
**Error encountered:** SSH access failed due to file permissions → **fixed with:** `chmod 600 ~/.ssh/id_rsa`
#### **Key decision:**
- Using Azure VM with SSH keys allows secure, password-less authentication ideal for automated deployments.
- Keeping the private key in GitLab protected variables ensures credentials remain secure and not exposed in the pipeline logs.
- This setup enables the CI/CD pipeline to connect and deploy containers to the Azure VM seamlessly.

### **Step 13: Deploy the Container Securely from GitLab**
#### **What was done:**
Configured the GitLab CI pipeline to securely deploy the Docker container to the Azure VM using SSH keys and automated scripts, eliminating any manual deployment steps.
#### **Commands and configuration:**
- Used the private SSH key stored as a protected variable **(SSH_PRIVATE)** in GitLab CI.
- Added a deploy job in .gitlab-ci.yml that:
	- Sets up SSH agent and adds the private key.
	- Connects to the Azure VM via SSH.
	- Pulls the latest Docker image from DockerHub.
	- Stops and removes any existing container.
	- Runs the new container.
#### **Key decision:**
Used `StrictHostKeyChecking=no` to bypass interactive confirmation in CI.

### **Step 14: Ensure Repeatable Deployment**
#### **What was done:**
Added command to remove existing containers before running new ones.
```bash
variables:
  IMAGE_NAME: ahmedsiddig1/demo-app
  IMAGE_TAG: python-app-1.0

stages:
  - test
  - build
  - deploy

run_tests:
  stage: test
  image: python:3.9-slim
  before_script:
    - apt-get update && apt-get install make
  script:
    - make test

build_image:
  stage: build
  image: docker:20.10.16
  services:
    - docker:20.10.16-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $REGISTRY_USER -p $REGISTRY_PASS
  script:
    - docker build -t $IMAGE_NAME:$IMAGE_TAG .
    - docker push $IMAGE_NAME:$IMAGE_TAG

deploy:
  stage: deploy
  before_script:
    - chmod 400 $SSH_KEY
  script:
    - ssh -o StrictHostKeyChecking=no -i $SSH_KEY ahmedsiddig@128.203.114.140 "
        docker login -u $REGISTRY_USER -p $REGISTRY_PASS &&
        docker ps -aq | xargs docker stop | xargs docker rm &&
        docker run -d -p 5000:5000 $IMAGE_NAME:$IMAGE_TAG"
```
#### **Key decision:**
- Using SSH keys and ssh-agent ensures secure, password-less remote execution.
- Automating container stop/remove before deploy ensures repeatable deployments without conflicts.
- Running deployment inside CI removes manual intervention, speeding up delivery and reducing human error.

### **Step 15: Prove the App is Accessible via a Browser**
#### **What was done:**
Verified that the deployed app is live and accessible through a web browser after the GitLab pipeline completed successfully
#### **Key decision:**
Ensured port 5000 was open in Azure’s Network Security Group (NSG) to allow inbound HTTP traffic. 

### **Step 16: Destroy the Cloud Instance When Finished**
#### **What was done:**
Deleted the Azure Virtual Machine (VM) after completing deployment and testing to avoid ongoing cloud charges.
#### **Key decision:**
Manually destroyed the instance instead of stopping the Docker container only.

## **3. Screenshots**

### **1. CI/CD Pipeline Output**
![CI-CD-pipeline-output](https://github.com/Zimondi/Lab09/blob/main/CI-CD-pipeline-output.png)
#### **What it shows:**
A screenshot of the completed GitLab pipeline showing all stages (test, build, deploy) marked as successful.
#### **Why it’s relevant to CI/CD security:**
It confirms the automation flow worked as expected with no manual interference, which reduces human error and enforces secure, repeatable deployments.

### **2. Running Containers**
![Running-containers](https://github.com/Zimondi/Lab09/blob/main/Running-containers.png)
#### **What it shows:**
The output of docker ps from the deployment server, showing the demo-app container running and exposing port 5000.
#### **Why it’s relevant to CI/CD security:**
Validates that the app was deployed securely and is running in an isolated container environment, reducing risks tied to direct server-level changes.

### **3. GitLab Interface Views**
![Gitlab-interface-views](https://github.com/Zimondi/Lab09/blob/main/Gitlab-interface-views.png)
#### **What it shows:**
Screenshots of the GitLab repository’s CI/CD > Pipelines tab, including .gitlab-ci.yml and project files.
#### **Why it’s relevant to CI/CD security:**
Shows the GitLab pipeline is properly configured and managed in version control, which ensures auditability and traceability of changes.

### **4. Browser Showing App**
![Browser-showing-app](https://github.com/Zimondi/Lab09/blob/main/Browser-showing-app.png)
#### **What it shows:**
A web browser displaying the live app at http://128.203.114.140:5000
#### **Why it’s relevant to CI/CD security:**
Proves that the deployed app is accessible and the deployment was successful, completing the CI/CD loop from code to production securely.

### **5. Deployment Server Details**
![Deployment-server-details](https://github.com/Zimondi/Lab09/blob/main/Deployment-server-details.png)
#### **What it shows:**
Azure Portal the VM’s status, IP address, and general information before deletion.
#### **Why it’s relevant to CI/CD security:**
Confirms that the deployment target was correctly provisioned and accessed, helping verify that only trusted environments were used.

### **6. Name and Date Screenshot**
![Gitlab-interface-views](https://github.com/Zimondi/Lab09/blob/main/Gitlab-interface-views.png)
#### **What it shows:**
Browser shows my name in Gitlab and the task bar shows the current date and time.
#### **Why it’s relevant to CI/CD security:**
Helps validate the authenticity and timing of the lab work — a common security practice for audit trails and verification.


## **4. Findings and Reflection**
### **Challenges:**
- YAML indenting broke early pipelines
- DockerHub login timed out
- SSH keys required secure permission management
### **Security Steps:**
- Stored secrets in CI/CD variables
- Used SSH key-based authentication
- Removed old containers for clean deployment
### **Comparison:**
GitLab CI/CD is easier to set up small to medium apps. Built-in logs and YAML pipelines are beginner friendly.

