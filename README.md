# Gitea CI/CD Pipeline with Helm, MySQL, and ngrok

## Features
- Helm deployment of Gitea with persistence
- Uses external MySQL database (Bitnami Helm)
- Exposed publicly via ngrok
- Modular YAML structure (prod vs ngrok)

## Project Structure

A3-Container-orchestration-Gitea/
│
├── prod/
│   ├── up.yaml                 # Ansible playbook to deploy MySQL and Gitea using Helm
│   └── values-gitea.yaml       # Helm override values for Gitea (external DB, persistence, etc.)
│
├── ngrok/
│   └── up.yaml                 # Ansible playbook to expose Gitea using ngrok tunnel
│
├── screenshots/                # (Optional) Folder containing screenshots for demonstration
│   ├── gitea-dashboard.png
│   └── pipeline-status.png
│
├── .gitignore                  # Ignore unnecessary files for Git
├── README.md                   # Main project documentation
├── COMMENTS.txt                # Contains ngrok link as per assignment requirements
└── structure.md                # (Optional) This file describing folder structure


## Setup

### Install Ansible

```bash
sudo apt update
sudo apt install ansible -y
```

### Using kind (Kubernetes IN Docker) — Recommended for Codespaces or VMs

```bash
# 1. Install kind if not already installed
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.22.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# 2. Create a Kubernetes cluster
kind create cluster --name devops-gitea

# 3. Verify connection
kubectl get nodes

```

### Install ngrok in your environment

```bash
# 1. Download ngrok
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update

# 2. Install ngrok
sudo apt install ngrok -y
```


### Step 1: Deploy MySQL and Gitea
```bash
ansible-playbook prod/up.yaml
```

### Step 2: Expose Gitea using ngrok
```bash
ansible-playbook ngrok/up.yaml
```

## Public Access

The ngrok URL will look like:  
`https://your-random-subdomain.ngrok-free.app`

Pasted this into `COMMENTS.txt'.

## Helm Charts Used
- [Gitea](https://dl.gitea.io/charts/)
- [Bitnami MySQL](https://bitnami.com/stack/mysql/helm)

## Persistent Storage
Gitea uses a 5Gi PVC via Helm values.

# Gitea Deployment Issues & Solutions

## What Went Wrong

I ran into two major problems while trying to deploy Gitea with an external MySQL database using Helm:

### Problem 1: Unwanted Redis Cluster
Even though I tried to disable Redis/Valkey in my configuration, the Gitea chart kept deploying a 3-node Valkey cluster that I didn't want or need.

**The Issue:** Turns out the Gitea Helm chart has `valkey-cluster` enabled by default, and just disabling the general Redis settings wasn't enough.

**The Fix:** I had to specifically disable `valkey-cluster` in my values.yaml file. This wasn't obvious from the documentation!

### Problem 2: Database Connection Failed
My Gitea pods kept crashing during initialization with an error saying "Database settings are missing from the configuration file."

**The Issue:** I was using the `externalDatabase` section in my values.yaml, but the Gitea init containers weren't actually reading this configuration. The database settings never made it into the app.ini file that Gitea needs.

**The Fix:** I had to put the database configuration in a completely different section called `gitea.config.database`. Only then did the init containers properly process the database settings.

## What I Learned

1. **Default settings can surprise you** - Always check what's enabled by default in Helm charts
2. **Configuration structure matters** - Where you put settings in values.yaml really matters
3. **Init container logs are your friend** - They show exactly which config sections are being processed
4. **External database setup isn't straightforward** - The `externalDatabase` section alone doesn't work

## Final Result

After fixing both issues:
- No more unwanted Valkey pods
- Database configuration actually worked
- Both Gitea and MySQL pods ran successfully
- The whole deployment finally worked as expected

This was definitely a learning experience about reading Helm chart documentation more carefully and understanding how configuration gets processed!
