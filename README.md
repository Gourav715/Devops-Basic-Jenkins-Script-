# Vainterior - DevOps CI/CD Demo

[![Jenkins Pipeline](https://img.shields.io/badge/Jenkins-CI/CD-blue)](https://jenkins.io)
[![Docker](https://img.shields.io/badge/Docker-Containerized-blue)](https://docker.com)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

## 🚀 Overview

A simple web application with complete CI/CD pipeline using **Jenkins** and **Docker**.

### Architecture
GitHub → Jenkins → Docker Build → Registry → Deploy

## 📋 Prerequisites

- Docker (20.10+)
- Docker Compose (2.0+)
- Jenkins (2.3+)
- Node.js (18+)

## 🛠️ Local Development

```bash
# Clone repository
git clone https://github.com/Gourav715/vainterior.git
cd vainterior

# Install dependencies
npm install

# Run locally
npm start

# Run with Docker
docker-compose up -d
