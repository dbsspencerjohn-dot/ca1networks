# Networks CA1 Project
**Automated Container deployment and Administration in the cloud**  
Network Systems and Administration - B9IS121

**Group members:**  
Nithyanantham Sanjeevi - 20097281  
Alexandre Peres Oliveira da Silva - 20096284    
Okeke Munachukwudinamma Praise - 20102055  
Spencer John - 20099070 
** Link to live Demo application **

[DEMO APPLICATION LINK](http://3.235.138.180:8501)


**Repositories**:  
[Main project repository - the only repository needed to run the project](https://github.com/dbsspencerjohn-dot/ca1networks.git)  

Auxiliary/intermediary repositories:  
[Docker setup repository - fork of the app repository](https://github.com/alex-silverr/nithyanands-irish-visa-tracker.git)  
[Web app used for the demo - Nithyanantham's Irish visa tracker](https://github.com/nithyanands/irish-visa-tracker.git)  
[Former setup repository - old main repository](https://github.com/dbsspencerjohn-dot/networking_ca1.git)

----

**Technologies Used:**
| Part      | Technology | Group Member Responsible |
| ----------- | ----------- | ----------- |
| Infrastructure Setup      | Terraform | Spencer John |
| Configuration Management   | Ansible | Okeke M. Praise |
| Docker Container Deployment | Docker | Alexandre P. Silva |
|CI/CD Pipeline Integration | Git Actions | Nithyanantham Sanjeevi |


## Part 1 - Terraform

## Purpose

Terraform is used to automate the provisioning of the cloud infrastructure. Rather than manually creating the application server through the AWS Management Console, Terraform defines the infrastructure as code, allowing the EC2 instance and its associated networking resources to be created consistently and repeatedly.

In this project, Terraform is installed on a management machine (either a manually created EC2 instance or a local computer). This management machine is responsible for provisioning the application server that will later be configured by Ansible and used to host the Docker container.

---

## Prerequisites

Before running Terraform, complete the following setup on the management machine:

- Install Terraform
- Install AWS CLI
- Install Ansible
- Create an AWS IAM user with programmatic access
- Attach the required permissions (or `AdministratorAccess` for demonstration purposes)
- Configure AWS CLI:

```bash
aws configure
```

Provide:

- AWS Access Key ID
- AWS Secret Access Key
- Region: `eu-west-1`
- Output format: `json`

Verify the configuration:

```bash
aws sts get-caller-identity
```

---

## Terraform Files

| File | Purpose |
|------|---------|
| `main.tf` | Defines the AWS infrastructure resources. |
| `variables.tf` | Stores configurable variables such as the AWS region, AMI ID, and instance type. |
| `outputs.tf` | Displays important information after deployment, such as the EC2 public IP address. |

---

## Deploying the Infrastructure

Initialize Terraform:

```bash
terraform init
```

Preview the execution plan (optional):

```bash
terraform plan
```

Provision the infrastructure:

```bash
terraform apply
```

Type:

```text
yes
```

when prompted.

Terraform provisions the application EC2 instance together with the required networking resources, including the security group.

After deployment completes, Terraform displays the public IP address of the newly created EC2 instance. This IP address is later used by Ansible to configure the server.

---

## Destroying the Infrastructure

To remove all resources created by Terraform:

```bash
terraform destroy
```

Confirm by typing:

```text
yes
```

Terraform will safely remove every resource it previously created.

---

## Verification

After deployment, verify that:

- The EC2 instance is running in the AWS Management Console.
- Terraform completed successfully without errors.
- The EC2 public IP address is displayed in the Terraform outputs.
- SSH access to the instance is successful.

Example:

```bash
ssh -i Networkca.pem ubuntu@<EC2_Public_IP>
```

Once SSH connectivity is confirmed, the server is ready for configuration using Ansible.

## Part 2 - Ansible

### Setup

- `sudo apt install ansible` - Install ansible  
- Ensure `Networkca.pem` (or your key) has correct permissions: `chmod 400 Networkca.pem`
- Confirm `host_file.ini` points at the current EC2 IP
- `ansible -i host_file.ini web -m ping` - Test connectivity before running anything

### Executing ansible

Configure the server (install & enable Docker):

- `ansible-playbook -i host_file.ini docker.yaml`

Deploy the application (clone repo + docker compose up):

- `ansible-playbook -i host_file.ini remote-clone.yaml`

Re-sync inventory after a fresh `terraform apply` (new IP):

- `./sync_deploy.sh`

### Verify

- `ansible -i host_file.ini web -m shell -a "systemctl is-enabled docker"` → should return `enabled`
- `ansible -i host_file.ini web -m shell -a "systemctl is-active docker"` → should return `active`
- `ansible -i host_file.ini web -m shell -a "docker ps"` → should show the running app container

---

## Part 3 - Docker

### Structure
**Docker Compose:**   
``docker-compose.yaml``  
This file sets up the service to run the container. It automates the arguments and options on a ``docker run`` command, making permanent settings instead of having to declare them every time the container is (re-)created or started.
- ``build: .`` instructs the composer to build the Dockerfile located on the same folder.
- ``container_name: visa-tracker`` sets the container name as *"visa-tracker*.
- ``tty: true`` makes it possible to execute the bash/shell inside the container.
- ``ports:`` specifies container ports connections to be accessed from outside.
  - ``8501:8501`` connects the internal port 8501 to external port 8501.
- ``restart:always`` later edit: makes sure the container restarts after being interrupted.



**Dockerfile:**  
``Dockerfile``  
This is the file that informs the container build instructions, and the main command to execute/deploy the app within it.
- ``FROM python:3.13`` declares the use of the Docker Python 3.13 image.
- ``WORKDIR /usr/local/app`` sets the internal working directory on the container to ``usr/local/app``.
- ``RUN apt-get update``*[...]* updates the ``apt-get`` command and allows for git integration.
- ``COPY requirements.txt ./`` copies the ``requirements.txt`` file from the external folder to the inside of the container.
  - ``requirements.txt`` is a file typical in Python projects listing the library dependencies necessary to run the project.
  - If these requirements change, such as new libraries being added, it's necessary to remake the ``requirements.txt`` file.
  - To do so, run the command ``pip freeze > requirements.txt`` on an environment that has all the libraries installed. 
- ``RUN pip install --no-cache-dir -r requirements.txt`` installs the library dependencies on the container.
- ``COPY . .`` copies files in the same folder as the ``Dockerfile`` to the container's working firectory.
- ``EXPOSE 8501`` exposes port 8501 for deployment of the Streamlit app.
- ``HEALTHCHECK CMD curl --fail http://localhost:8501/_stcore/health`` verifies if the exposed port 8501 is working correctly.
- ``CMD ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0" ]`` executes the Streamlit app ``app.py`` on port 8501.

### How to use
UP:
- Go to the folder where ``docker-compose.yaml`` is
- If there was a change in the app: ``docker compose up --build -d``
- If there was no change in the app: ``docker compose up -d``

DOWN:
- Go to the folder where ``docker-compose.yaml`` is
- ``docker compose down``

MAINTENANCE:
- See logs: ``docker container logs visa-tracker``
- Access container shell: ``docker exec -it visa-tracker sh``

## Part 4 - Git Actions

### Approach
GitHub Actions automatically deploys the application whenever code is pushed to `main`. The workflow (`.github/workflows/Highactions.yml`) runs on a **self-hosted runner** and executes an Ansible playbook (`remote-clone.yaml`) that clones/updates the repository on the EC2 instance and rebuilds the container with Docker Compose. This keeps the entire deploy step — code retrieval and container rebuild — inside a single, consistent Ansible run each time, rather than splitting it across separate tools.

### Workflow file
`.github/workflows/Highactions.yml`:
```yaml
name: Application Deployment

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: self-hosted

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Secure SSH Private Key
        run: chmod 400 Networkca.pem

      - name: Run Ansible Playbook
        run: ansible-playbook -i host_file.ini remote-clone.yaml
```

### Ansible playbook used by the workflow
`remote-clone.yaml`:
```yaml
---
- hosts: web
  become: yes

  tasks:
    - name: Clone or update repository
      git:
        repo: https://github.com/dbsspencerjohn-dot/ca1networks.git
        dest: /home/ubuntu/app
        version: main
        update: yes

    - name: Run Docker Compose
      become_user: ubuntu
      command: docker compose up -d --build
      args:
        chdir: /home/ubuntu/app
```

### How it works:
1. A commit is pushed to `main` repo
2. The self-hosted runner (registered on a persistent machine 'VM' ) picks up the job immediately
3. The workflow checks out the repository (including `Networkca.pem`, which is committed for this project) and sets its permissions to `400` so SSH will accept it.
4. Ansible runs `remote-clone.yaml` against the EC2 instance listed in `host_file.ini`: it clones the repo on first run, or pulls the latest changes on subsequent runs, into `/home/ubuntu/app`.
5. Also ansible playbook runs, Docker Compose rebuilds and restarts the container as the `ubuntu` user.

### Setup dependencies
- A self-hosted GitHub Actions runner registered against this repository, kept online
- `host_file.ini` present, pointing at the current EC2 instance's Public IP
- `Networkca.pem` present in the repository (used directly by the runner )
- The EC2 instance's `ubuntu` user already added to the `docker` group (set up once, outside this workflow, during initial server configuration)
- `docker-compose.yaml` includes `restart: always`, so the container also survives a full instance reboot, not just a redeploy

### Known limitations (accepted for this project's scope)
- **Self-hosted runner dependency**: this workflow only runs if the registered runner machine is online and reachable. If that machine is off, every push will fail to trigger a deployment.
- **Private key committed to the repository**: `Networkca.pem` is committed and used directly by the workflow.
