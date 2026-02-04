Step 1: Prepare Jenkins Server and Configure it
        * Install Java and Jenkins
        * Install Ansible as Jenkins Server will also act as an Ansible Machine
        * Create SSH Public key and Copy that to Deployment Server's Authorized Keys
        * Check if remote SSH succesful by: ssh ubuntu@<deploymentServer>
Step 2: Prepare Deployment Server and Configure it
Step 3: Create Git Repository and Push the Codebase to Remote Repository
Step 4: Write hosts, ansible.cfg and ansible playbook and Push to remote repository
Step 5: Write Jenkinsfile
Step 6: Go to Jenkins WebUI and install necessary Plug-Ins such as Ansible and Create a Pipeline
Step 7: Create a GitHub Webhook to complete the CICD Pipeline. (https://<jenkinsServer>:8080/github-webhook/)

************************************** In Detail **********************************************

## Overview

This repository demonstrates a simple **CI/CD** pipeline using Jenkins, Ansible, and NGINX to deploy a static website to a web server.

The flow:

1. Code is pushed to GitHub.
2. Jenkins pulls the latest code and runs an Ansible playbook.
3. Ansible installs/configures NGINX and deploys the website to `/var/www/html`.

***

## Architecture

- Jenkins server
    - Has Ansible installed
    - Runs the pipeline defined in `Jenkinsfile`
- Web (deployment) server
    - Accessible via SSH from Jenkins
    - Target of the Ansible playbook (`web-server` host group)
- GitHub repository
    - Holds:
        - Application files in `web/`
        - Ansible playbook (`web-playbook.yml`)
        - Inventory (`hosts`)
        - `Jenkinsfile`

***

## Prerequisites

- Jenkins installed and reachable via browser.
- Ansible installed on the Jenkins server.
- SSH key-based access from Jenkins server to the web server (no password prompts).
- Web server is running a Debian/Ubuntu-based OS with `apt`.

***

## Repository Structure

Example layout:

```text
.
├── Jenkinsfile
├── hosts
├── web-playbook.yml
└── web/
    ├── index.html
    ├── styles.css
    └── ...
```

- `web-playbook.yml`: Ansible playbook that installs NGINX and deploys content.
- `hosts`: Ansible inventory defining the `web-server` group.
- `web/`: Static website files copied to `/var/www/html/`.

***

## Ansible Inventory (`hosts`)

Make sure your `hosts` file contains the correct group name and target:

```ini
[web-server]
<your_web_server_ip_or_hostname> ansible_user=ubuntu
```

Adjust `ansible_user` and host according to your environment.

***

## Ansible Playbook (`web-playbook.yml`)

This playbook:

1. Updates apt cache.
2. Installs NGINX.
3. Ensures NGINX is running and enabled.
4. Replaces the default NGINX web root with your custom site.
```yaml
- hosts: web-server
  become: true

  tasks:
    - name: update apt
      apt:
        update_cache: yes

    - name: install nginx with apt
      apt:
        name: nginx
        state: present

    - name: ensure nginx is running
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Deleting default landing page
      file:
        path: /var/www/html
        state: absent

    - name: Re-creating the entire directory
      file:
        path: /var/www/html
        state: directory

    - name: Copying web content from the host machine
      copy:
        src: "{{ item }}"
        dest: /var/www/html/
      with_fileglob:
        - "web/*"
```

> Note: This assumes the `web/` directory is present in the workspace where Ansible runs (Jenkins checkout).

***

## Jenkins Pipeline (`Jenkinsfile`)

The Jenkins pipeline has two main stages:

1. Clone the GitHub repository.
2. Run the Ansible playbook against the inventory.
```groovy
pipeline {
    agent any

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/farhaz1449/nginx-website-ec2-ansible-jenkins-CICD.git'
            }
        }
        stage('Run Ansible Playbook') {
            steps {
                ansiblePlaybook playbook: 'web-playbook.yml', inventory: 'hosts'
            }
        }
    }
    post {
        success {
            echo 'Pipeline execution successful.'
        }
        failure {
            echo 'Pipeline execution failed.'
        }
    }
}
```

Make sure:

- The Jenkins Ansible plugin is installed.
- `web-playbook.yml` and `hosts` are at the repo root (or update paths accordingly).

***

## Jenkins Setup

1. Install required plugins:
    - Git plugin
    - GitHub plugin (if using webhook triggers)
    - Ansible plugin
2. Create a new Pipeline job:
    - In Jenkins, click **New Item** → enter a name → select **Pipeline**.
    - Under **Pipeline**:
        - Definition: **Pipeline script from SCM**
        - SCM: **Git**
        - Repository URL: `https://github.com/farhaz1449/nginx-website-ec2-ansible-jenkins-CICD.git`
        - Branch: `main`
        - Script Path: `Jenkinsfile` (if at repo root)
3. Save and run **Build Now** once to verify:
    - Jenkins can pull code.
    - Ansible can SSH to the web server.
    - NGINX gets installed and serves your site.

***

## GitHub Webhook (Optional but Recommended)

To trigger the pipeline automatically on each push:

1. In Jenkins job configuration:
    - Under **Build Triggers**, enable:
**GitHub hook trigger for GITScm polling**.
2. In GitHub:
    - Go to **Settings → Webhooks → Add webhook**.
    - Payload URL:
`http://<jenkins-server-domain-or-ip>:8080/github-webhook/`
(use `https` if TLS is configured).
    - Content type: `application/json`.
    - Choose **Just the push event**.
    - Click **Add webhook**.

Push a change to the `main` branch and confirm Jenkins triggers a new run.

***

## How to Test the Deployment

1. After a successful pipeline run, open your browser and go to:

```text
http://<your_web_server_ip>/
```

2. You should see the content from the `web/` directory (for example, `index.html`).
3. Modify a file in `web/`, commit and push, then:
    - Watch Jenkins run the pipeline.
    - Refresh your browser to see the updated site.

***

## Troubleshooting

- **SSH errors**:
Ensure the Jenkins server can SSH into the web server using the same user and key Ansible uses.
- **Host unreachable / not in group**:
Confirm the host in `hosts` is reachable and belongs to the `web-server` group (matching `hosts: web-server` in the playbook).
- **NGINX not serving content**:
Check:
    - NGINX status: `sudo systemctl status nginx`
    - Contents of `/var/www/html` on the web server.

This `INSTRUCTION.md` should give anyone cloning your repo a clear, end‑to‑end path from setup to a working CI/CD deployment.

