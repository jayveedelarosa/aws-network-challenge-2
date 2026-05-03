# AWS Network Challenge 2: Deploy a File Upload App on EC2, RDS, and DocumentDB

![AWS](https://img.shields.io/badge/AWS-Cloud-orange)
![Flask](https://img.shields.io/badge/Backend-Flask-blue)
![PostgreSQL](https://img.shields.io/badge/Database-PostgreSQL-336791)
![MongoDB](https://img.shields.io/badge/Database-MongoDB-green)
![Status](https://img.shields.io/badge/Status-In%20Progress-yellow)

---

## 👋 About This Project

My name is **Jayvee Dela Rosa**, a 1st year BSCS student.

On **April 21, 2026**, I attended **VibeAIPH #2: Kiro for Work Workshop** at
Arthaland Century Pacific Tower, Bonifacio Global City (BGC), Taguig City,
Metro Manila, an event typically attended by industry professionals. Despite
being a 1st year student, I chose to participate.

During the event, I met sir **Raphael Jambalos** — AWS Community Hero, Head
of Tech at eCloudValley Philippines, and the speaker of the workshop. I
expressed my desire to join his **Elevate Innovations Training Program**, but
the program prioritizes graduating students and fresh graduates.

Because I insisted on wanting to join, **Sir Raphael challenged me** to:

- ✅ Build a full AWS project
- ✅ Document it professionally on GitHub
- ✅ Be able to explain it clearly — by **May 9, 2026**

This repository is my response to that challenge.

---

## 📌 Project Overview

This project is based on **Sir Raphael's AWS Network Challenge 2**, which
involves deploying a Flask-based file upload eCommerce application across
multiple AWS services, with an architecture that evolves from a simple
single-server setup to a fully managed, auto-scaling, CI/CD-enabled cloud
deployment.

> 📖 Original Challenge Article by Sir Raphael:
> [AWS Network Challenge 2](https://dev.to/raphael_jambalos/aws-network-challenge-2-deploy-a-file-uploading-app-on-ec2-rds-documentdb-16eb)
>
> 💻 Original Application Repository:
> [jamby1100/file-upload-flask](https://github.com/jamby1100/file-upload-flask/tree/main)

---

## 📁 How the Application Works

The app (`main.py`) is a Flask web application with 4 routes:

| Route | What it does |
|-------|-------------|
| `/` | Returns "Hello, World Elisha!" — health check |
| `/upload-file` | Upload a product image + set stock count |
| `/images` | View all uploaded product images |
| `/uploads/<name>` | Serve the actual image file |

When a user uploads an image:

1. The image file is saved to disk (later to EFS)
2. The filename is saved to **MongoDB/DocumentDB**
3. The product details are saved to **PostgreSQL/RDS**

---

## 🛠️ Tech Stack

| Category | Technology |
|----------|-----------|
| Backend | Python, Flask |
| Database (NoSQL) | MongoDB → Amazon DocumentDB |
| Database (SQL) | PostgreSQL → Amazon RDS |
| File Storage | Local → Amazon EFS |
| Web Server | Nginx → AWS Application Load Balancer |
| Cloud Provider | Amazon Web Services (AWS) |
| Region | ap-southeast-1 (Singapore) |
| OS | Amazon Linux 2 (Lab 1), Amazon Linux 2023 (Lab 2+) |

---

## 🏗️ Architecture Evolution (Labs 1–6)

This project is structured as a series of labs, each building on the
previous one. The architecture evolves from a single fragile server to a
fully managed, auto-scaling cloud infrastructure.

| Lab | Status | Description |
|---|---|---|
| Lab 1 | ✅ Complete | Progressive LightSail deployment: single server, three separate servers, five servers with replication |
| Lab 2 | ✅ Complete | Multi-server deployment inside AWS VPC with public and private subnets |
| Lab 3 | ✅ Complete | Replace manual databases with Amazon RDS and Amazon DocumentDB, replace Nginx with Application Load Balancer |
| Lab 4 | ⏳ Coming Soon | Remove state from app servers using Amazon EFS for shared file storage |
| Lab 5 | ⏳ Coming Soon | Auto Scaling Groups for automatic EC2 scaling |
| Lab 6 | ⏳ Coming Soon | CI/CD pipeline with AWS CodePipeline and CodeDeploy |

---

## ⚙️ AWS Services Used (So Far)

- **Amazon LightSail** — Lab 1 simple deployment across single and
  multi-server configurations

- **Amazon VPC** — Virtual Private Cloud with public and private subnets

- **Amazon EC2** — Virtual servers for the app, proxy, MongoDB, and PostgreSQL

- **Nginx** — Reverse proxy on the public-facing server (Lab 2)

- **AWS Security Groups** — Per-server firewall rules

- **NAT Gateway** — Temporary internet access for private subnet servers
  during setup

- **AWS Service Quotas** — Instance limit management for multi-server
  LightSail deployments

- **Amazon RDS** — Fully managed PostgreSQL database replacing the manually
  configured PostgreSQL EC2 server

- **Amazon DocumentDB** — Fully managed MongoDB-compatible database replacing
  the manually configured MongoDB EC2 server

- **AWS Application Load Balancer** — Managed load balancer replacing the
  Nginx Proxy Server EC2, distributing traffic to the App Server

- **DB Subnet Groups** — Required configuration for placing RDS and
  DocumentDB inside the VPC private subnets

- **DocumentDB Parameter Groups** — Used to disable TLS on the DocumentDB
  cluster for pymongo compatibility

---

## 🙏 References and Credits

- **Raphael Jambalos** — Original challenge creator, AWS Community Hero,
  Head of Tech at eCloudValley Philippines
  - 📖 [Dev.to Article](https://dev.to/raphael_jambalos/aws-network-challenge-2-deploy-a-file-uploading-app-on-ec2-rds-documentdb-16eb)
  - 💻 [Original GitHub Repository](https://github.com/jamby1100/file-upload-flask/tree/main)

- **VibeAIPH Community** — For hosting the event where this challenge
  was given

---

## 👤 Author

**Jayvee Dela Rosa** — 1st Year BSCS Student

- 💼 LinkedIn: [Jayvee Dela Rosa](https://www.linkedin.com/in/jayveedelarosa/)
- 🐙 GitHub: [@jayveedelarosa](https://github.com/jayveedelarosa)
- 📘 Facebook: [Jayvee C. Dela Rosa](https://www.facebook.com/JvCDr/)
- 🌐 Portfolio: [jayveedelarosa.dev](https://jayveedelarosa.dev)

