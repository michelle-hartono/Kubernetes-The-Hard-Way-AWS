# Kubernetes the Hard Way on AWS (Mac Setup)

A step-by-step guide to manually deploying a Kubernetes cluster on AWS using EC2 instances and basic networking—without EKS or managed services.

> Inspired by techiescamp [Kubernetes the Hard Way AWS](https://github.com/techiescamp/kubernetes-projects/tree/main/01-kubernetes-the-hard-way-aws). 

---

## 📋 Table of Contents

1. [Overview](overview.md)  
2. [Local Environment Setup (macOS)](local-environment-setup.md)  
3. [AWS Infrastructure Setup](aws-infrastructure-setup.md)  
4. [Kubernetes Cluster Setup](kubernetes-cluster-setup.md)  
5. [Cleanup](cleanup.md)  
6. [Lessons Learned & References](lessons-learned.md)

---

## 🧠 Overview

This project walks through how to build a Kubernetes cluster from scratch on AWS using EC2 instances and manual component setup. No managed services. No shortcuts. Just raw Kubernetes knowledge.

### Project Goals

- Understand Kubernetes internals by building everything manually
- Deploy a working multi-node cluster using AWS EC2
- Use Terraform and bash scripts minimally, focusing on core Kubernetes components
- Document each step clearly and reproducibly

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
