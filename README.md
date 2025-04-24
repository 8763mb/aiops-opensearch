# OpenSearch 2.19.1 ML Model Installation Guide

This guide provides step-by-step instructions for installing OpenSearch 2.19.1 with Docker Desktop on macOS, configuring the environment, and uploading/deploying a Machine Learning (ML) model using the ML Commons plugin. It includes troubleshooting tips for common issues encountered during the process.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
- [Docker Compose Configuration](#docker-compose-configuration)
- [Starting OpenSearch](#starting-opensearch)
- [Uploading and Deploying an ML Model](#uploading-and-deploying-an-ml-model)
- [Verifying in OpenSearch Dashboards](#verifying-in-opensearch-dashboards)
- [Troubleshooting Common Issues](#troubleshooting-common-issues)
- [References](#references)

## Prerequisites

- **Operating System:** macOS Ventura (13.x) or later
- **Docker Desktop:** Version 4.30 or higher
- **Minimum resources:** 8GB memory, 4 CPU cores, 1GB swap
- **Disk space:** At least 10GB free for OpenSearch data and ML models
- **Network:** Stable internet connection for downloading Docker images and ML models
- **Tools:** `curl` for API testing, `docker` and `docker compose` commands
- **Permissions:** Admin access to adjust Docker Desktop settings and execute commands

## Environment Setup

### Step 1: Install Docker Desktop

1. Download and install Docker Desktop from Docker's official website.

2. Verify installation:
   ```bash
   docker --version
   docker compose version
