

Here's an outline for the **DockerOperator Adoption Documentation** to guide your team in implementing containerized tasks in Airflow.  

---

# **Airflow DockerOperator Integration Guide**

## **Purpose**
This guide explains how to containerize Airflow tasks using the `DockerOperator` and run them in isolated environments. The goal is to enable modular development, resolve dependency conflicts, and improve scalability and reproducibility.

---

## **1. Prerequisites**

### **For Local Development**  
1. **Docker Installed**  
   - Install Docker on your machine. Follow the [official Docker installation guide](https://docs.docker.com/get-docker/).
2. **Airflow Setup**  
   - Ensure Airflow is installed locally and accessible via the webserver.
3. **AWS ECR Setup (if using AWS)**  
   - Install and configure the AWS CLI.
   - Authenticate Docker with ECR using:
     ```bash
     aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com
     ```

### **For Airflow Server**
- Ensure the Airflow instance has access to Docker (use the EC2 public/private IP or other configured Docker daemons).
- Docker Daemon must run with proper network configurations (`bridge`, `host`, etc.).

---

## **2. Defining a Dockerized Task**

### **Dockerfile Example**  
Create a `Dockerfile` for your task. Below is an example for a Python-based data cleaning task.

```dockerfile
# Base image
FROM python:3.10-slim

# Install dependencies
COPY requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r /app/requirements.txt

# Copy your script
COPY script.py /app/script.py

# Set working directory
WORKDIR /app

# Default command to execute
CMD ["python", "/app/script.py"]
```

### **Build and Push the Image**  
1. Build the Docker image locally:
   ```bash
   docker build -t job-recommender-task:v1.0 .
   ```
2. Test the container locally:
   ```bash
   docker run -it --rm job-recommender-task:v1.0
   ```
3. Tag and push to ECR (replace `<account-id>` and `<repo-name>`):
   ```bash
   docker tag job-recommender-task:v1.0 <account-id>.dkr.ecr.us-east-1.amazonaws.com/<repo-name>:v1.0
   docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/<repo-name>:v1.0
   ```

---

## **3. Configuring DockerOperator in Airflow**

### **Task Configuration**
Here’s how you can configure a `DockerOperator` in an Airflow DAG:

```python
from airflow import DAG
from airflow.providers.docker.operators.docker import DockerOperator
from airflow.utils.dates import days_ago
from datetime import timedelta

# Default args
default_args = {
    'owner': 'wei',
    'depends_on_past': False,
    'retries': 3,
    'retry_delay': timedelta(seconds=30),
}

# Define the DAG
dag = DAG(
    'data_cleaning_task_script',
    default_args=default_args,
    description='Cleans, transforms, and loads job data into PostgreSQL',
    schedule_interval=None,
    start_date=days_ago(1),
    tags=['scraper_job_recommender_project'],
)

# DockerOperator Task
docker_task = DockerOperator(
    task_id='run_data_cleaning_task',
    image='739949438651.dkr.ecr.us-east-1.amazonaws.com/job-recommender-task:v1.0',  # Replace with your ECR image URL
    api_version='auto',
    auto_remove=True,
    docker_url='tcp://3.147.199.196:2375',  # Update with Docker Daemon host (EC2/localhost)
    network_mode='bridge',
    command="python /app/script.py",  # Command to run inside the container
    environment={"MY_ENV_VAR": "value"},  # Any required environment variables
    dag=dag,
)
```

---

## **4. Managing Dependencies**

### **Best Practices**  
- Use **Docker Compose** locally to test tasks with interdependencies.  
- Each container should include only the dependencies necessary for its task.  
- Include a `requirements.txt` or `pip freeze` file to lock dependency versions.

### **Handling OpenAI and Other API Keys**  
Store API keys and secrets securely using:
- **Airflow Connections** for environment variables.
- Secrets Manager (e.g., AWS Secrets Manager) for sensitive data.

---

## **5. CI/CD Pipeline for Docker Images**

Automate image building and deployment with GitHub Actions:

```yaml
name: Build and Push Docker Image

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Log in to Amazon ECR
      run: aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com

    - name: Build Docker image
      run: docker build -t <repo-name>:latest .

    - name: Tag Docker image
      run: docker tag <repo-name>:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/<repo-name>:latest

    - name: Push Docker image
      run: docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/<repo-name>:latest
```

---

## **6. Debugging & Logs**

### **Debugging Docker Containers**
- Use `docker logs <container_id>` to inspect task-specific issues.
- Add verbose logging in your scripts (`logging` library in Python) to capture debug information.

### **Airflow Logs**
- Navigate to Airflow's task log view for each DAG execution.
- Enable debug mode in Airflow for verbose error messages (`airflow.cfg` > `logging_level=DEBUG`).

---

## **7. Roadmap Adoption Steps**
1. Roll out a pilot DAG using `DockerOperator`.
2. Test locally and on the Airflow server.
3. Define templates for Dockerfiles, CI/CD pipelines, and task DAGs.
4. Migrate existing tasks to Docker containers incrementally.
5. Document and train the team on containerized workflows.

---

Does this structure and documentation align with your team's needs? Let me know if you'd like to expand on any section!
