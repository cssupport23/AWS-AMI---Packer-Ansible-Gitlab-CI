

### AWS AMI Creation with Packer, Ansible, and GitLab CI/CD

This project leverages Packer, Ansible, and GitLab CI/CD to automate the creation of an AWS AMI for deploying a Next.js application. The workflow begins with Packer, which provisions an EC2 instance, installs required dependencies using Ansible, and creates a customized AMI for future use. The process is integrated with GitLab CI/CD, where the pipeline is defined in a GitLab YAML file.

The pipeline performs the following steps:
1. **Packer Build**: Packer initiates the build by launching an EC2 instance, installs necessary tools using an Ansible playbook, and takes a snapshot of the configured instance as an AMI. The AMI ID is stored in AWS SSM for further use.
2. **Ansible Provisioning**: Ansible playbooks are used for installing dependencies such as PM2, Node.js, and pulling the latest application code from a Git repository. The playbook is applied directly on the EC2 instance during the Packer build.
3. **CI/CD Pipeline**: GitLab CI/CD automates the build and deployment process. The pipeline fetches variables such as AWS credentials and application version from AWS SSM, clones the Next.js repository, and invokes the Packer build. The generated AMI ID is stored in SSM for subsequent deployment stages.

The GitLab CI pipeline also handles tasks like updating group variables for Ansible, storing artifacts like the AMI ID, and running shell scripts for environment setup.

This solution allows for seamless creation, provisioning, and deployment of AWS AMIs in an automated, repeatable fashion, integrating well with GitLab for continuous delivery.

---

### GitLab CI Pipeline (`.gitlab-ci.yml`)

```yaml
image: 
  name: hashicorp/packer:full
  entrypoint: [""]

variables:
  EC2_KEYS_PUB_SSM: 'ec2_keys_public'
  CICD_TOK_SSM: 'cicdtoken'
  VERSION_SSM: '/nextjs-app/version'
  CISD_USER_SSM: '/cicd/user'
  GIT_REPO: <domain>/aws/nextjsapp.git # Update with your GitLab domain

stages:
  - build

before_script:
  - |
    cat /etc/os-release
    apk update
    apk add openssh-client ansible aws-cli git
    mkdir ansible/files
    EC2_KEY_PUBLIC=$(aws ssm get-parameter --name "${EC2_KEYS_PUB_SSM}" --query 'Parameter.Value' --output text)
    echo $EC2_KEY_PUBLIC >> id_rsa.pub
    export CICD_TOKEN=$(aws ssm get-parameter --name "${CICD_TOK_SSM}" --query 'Parameter.Value' --output text)
    export VERSION=$(aws ssm get-parameter --name "${VERSION_SSM}" --query 'Parameter.Value' --output text)
    export CICD_USER=$(aws ssm get-parameter --name "${CISD_USER_SSM}" --query 'Parameter.Value' --output text)
    cp id_rsa.pub ansible/files/id_rsa.pub
    REPO_URL="http://$CICD_USER:$CICD_TOKEN@$GIT_REPO"
    git clone --branch $VERSION $REPO_URL nj_repo
    cd nj_repo
    export COMMIT_HASH=$(git rev-parse HEAD)
    cd ..
    echo "Updating group_vars/all.yml with fetched values..."
    sed -i "s|personal_access_token:.*|personal_access_token: $CICD_TOKEN|" ./ansible/group_vars/all.yml
    sed -i "s|version:.*|version: $COMMIT_HASH|" ./ansible/group_vars/all.yml  
    sed -i "s|username:.*|username: $CICD_USER|" ./ansible/group_vars/all.yml
    cat ./ansible/group_vars/all.yml

build:
  stage: build
  script:
    - |
      echo "Running Packer"
      packer build ./nextjs_ami.json | tee packer_output.log
      AMI_ID=$(grep -o 'ami-[a-zA-Z0-9]\+' packer_output.log | tail -n 1)
      aws ssm put-parameter --name "/nextjs/ami-id" --value "$AMI_ID" --type "String" --overwrite
      echo $AMI_ID > ami_output.txt
  artifacts:
    paths:
      - ami_output.txt
  tags:
    - aws
```

### Explanation:
1. **Variables**: Values such as SSM parameters, tokens, and the Git repository URL are fetched and stored in variables. T
2. **Before Script**: It installs required packages (`ansible`, `aws-cli`, `git`), retrieves public keys, tokens, and application version from AWS SSM, and clones the Next.js repository. This code can be put into custom docker image, which then can be used to run the gitlab job.
3. **Build Stage**: Runs Packer to build the AMI, logs the AMI ID, and stores it in AWS SSM for later use.

---

### Packer Template (`nextjs_ami.json`)

```json
{
  "variables": {
    "aws_access_key": "",
    "aws_secret_key": "",
    "ami_name" : "nextjs-ami-{{env `VERSION`}}-{{env `CI_COMMIT_SHORT_SHA`}}-{{env `COMMIT_HASH`}}-{{timestamp}}"
  },
  "builders": [{
    "type": "amazon-ebs",
    "access_key": "{{user `aws_access_key`}}",
    "secret_key": "{{user `aws_secret_key`}}",
    "region": "ap-south-1",
    "source_ami_filter": {
      "filters": {
        "virtualization-type": "hvm",
        "image-id": "ami-08718895af4dfa033",
        "root-device-type": "ebs"
      },
      "owners": ["137112412989"],
      "most_recent": true
    },
    "instance_type": "t2.micro",
    "ssh_username": "ec2-user",
    "ami_name": "{{user `ami_name`}}"
  }],
  "provisioners": [
    {
      "type": "file",
      "source": "scripts",
      "destination": "/home/ec2-user/"
    },
    {
      "type": "shell",
      "scripts": [
        "scripts/prepare.sh"
      ]
    },
    {
      "type": "file",
      "source": "ansible",
      "destination": "/home/ec2-user/"
    },
    {
      "type": "ansible",
      "playbook_file": "ansible/setup_playbook.yml",
      "user": "ec2-user",
      "extra_arguments": [
        "--become", "-e ANSIBLE_CONFIG=/home/ec2-user/ansible/ansible.cfg"
      ]
    },
    {
      "type": "shell",
      "scripts": [
        "scripts/cleanup.sh"
      ]
    }
  ]
}
```

### Explanation:
1. **Builders**: Defines the configuration for an `amazon-ebs` build, using a base AMI and specific region. The `ami_name` is dynamic and includes environment variables such as version and commit hash.
2. **Provisioners**: It includes file uploads (e.g., Ansible playbook), running shell scripts (prepare, cleanup), and applying the Ansible playbook on the instance.

---

### Ansible Playbook (`setup_playbook.yml`)

```yaml
---
# Prepare host
- name: Prepare host
  hosts: all
  become: true  
  gather_facts: false  
  roles:
    - ssh_setup
    - dependencies
    - app
```

### Explanation:
- This playbook defines roles like `ssh_setup`, `dependencies`, and `app` to configure the EC2 instance.
  
- **Roles**:
  - `ssh_setup`: Handles the SSH configuration.
  - `dependencies`: Installs necessary packages such as Node.js, PM2.
  - `app`: Clones the application and sets up the Next.js environment.

---
