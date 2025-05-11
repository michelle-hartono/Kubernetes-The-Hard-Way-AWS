# Kubernetes the Hard Way on AWS (Mac Setup)

A step-by-step guide to manually deploying a Kubernetes cluster on AWS using EC2 instances and basic networking—without EKS or managed services.

> Inspired by techiescamp [Kubernetes the Hard Way AWS](https://github.com/techiescamp/kubernetes-projects/tree/main/01-kubernetes-the-hard-way-aws). 

---

## 🧠 Overview

This project walks through how to build a Kubernetes cluster from scratch on AWS using EC2 instances and manual component setup. No managed services. No shortcuts. Just raw Kubernetes knowledge.

### Project Goals

- Understand Kubernetes internals by building everything manually
- Deploy a working multi-node cluster using AWS EC2
- Use Terraform and bash scripts minimally, focusing on core Kubernetes components
- Document each step clearly and reproducibly

---
## 🏨 Visualizing AWS VPC Networking (Hotel Analogy)

Before diving into the full setup, I highly recommend reviewing this **hotel analogy** I created to simplify AWS VPC networking concepts.  
It’s a standalone learning aid—not directly part of the Kubernetes the Hard Way setup—but it helps visualize what you're about to build.

🔹 **Why this analogy?**  
Because VPCs, subnets, route tables, and gateways can be confusing at first. This approach offers an intuitive way to "see" how it all fits together.

📌 **[Explore the full analogy with diagram →](00-Analogy.md)**

![VPC Hotel Analogy](vpc-hotel-analogy.jpg)

---

## ⚙️ Prerequisites

### On your Mac:
- [Homebrew](https://brew.sh/)
- AWS CLI (`brew install awscli`)
- Terraform (`brew install terraform`)
- kubectl (`brew install kubectl`)
- cfssl (`brew install cfssl`)
- jq, wget, tmux (optional but useful)

### AWS Setup:
- An AWS account with EC2 and VPC permissions
- An IAM user or role with programmatic access
- AWS credentials configured:
  ```
  aws configure
  ```
