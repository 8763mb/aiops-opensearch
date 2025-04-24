# OpenSearch 2.19.1 ML Model Installation Guide

This guide provides step-by-step instructions for installing OpenSearch 2.19.1 with Docker Desktop on macOS, configuring the environment, and uploading/deploying a Machine Learning (ML) model using the ML Commons plugin. It includes troubleshooting tips for common issues encountered during the process.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
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
   ```
   Expected output: Docker version 27.x.x and Docker Compose version v2.x.x.

### Step 2: Configure Docker Desktop Resources

1. Open Docker Desktop > Preferences > Resources
2. Set:
   - Memory: 8GB (minimum 4GB)
   - CPUs: 4 cores (minimum 2)
   - Swap: 1GB
3. Click Apply & Restart

### Step 3: Adjust Virtual Machine Settings

OpenSearch requires `vm.max_map_count` to be at least 262144:

1. Access Docker Desktop's virtual machine:
   ```bash
   docker run -it --privileged --pid=host justincormack/nsenter1
   ```

2. Set the parameter:
   ```bash
   sysctl -w vm.max_map_count=262144
   ```

3. Verify:
   ```bash
   sysctl vm.max_map_count
   ```
   Expected output: `vm.max_map_count = 262144`

4. Exit:
   ```bash
   exit
   ```

> **Note**: This setting may reset on Docker Desktop restart. Reapply if needed.

## Starting OpenSearch

1. Start the containers:
   ```bash
   docker compose up -d
   ```
> **Important**: Replace `YourStrong@Passw0rd` with a secure password (minimum 8 characters, including uppercase, lowercase, numbers, and special characters).

2. Check container status:
   ```bash
   docker compose ps
   ```
   Ensure `opensearch-node1` and `opensearch-dashboards` are in the running state.

3. Verify OpenSearch API:
   ```bash
   curl -XGET https://localhost:9200 -u 'admin:YourStrong@Passw0rd' --insecure
   ```
   Expected output:
   ```json
   {
     "name": "opensearch-node1",
     "cluster_name": "opensearch-cluster",
     "version": {
       "number": "2.19.1",
       ...
     }
   }
   ```

4. Verify ML Commons plugin:
   ```bash
   curl -XGET https://localhost:9200/_cat/plugins -u 'admin:YourStrong@Passw0rd' --insecure
   ```
   Look for `opensearch-ml 2.19.1.0` in the output.

## Uploading and Deploying an ML Model

### Step 1: Enable Model URL Registration

Allow models to be registered via external URLs:
```bash
curl -XPUT https://localhost:9200/_cluster/settings -u 'admin:YourStrong@Passw0rd' -H 'Content-Type: application/json' -d '{"persistent":{"plugins.ml_commons.allow_registering_model_via_url": true}}' --insecure
```

Expected output:
```json
{"acknowledged":true,"persistent":{"plugins":{"ml_commons":{"allow_registering_model_via_url":"true"}}},"transient":{}}
```

### Step 2: Create a Model Group

Register a model group to organize models:
```bash
curl -XPOST https://localhost:9200/_plugins/_ml/model_groups/_register -u 'admin:YourStrong@Passw0rd' -H 'Content-Type: application/json' -d '{"name": "test_group", "description": "A test model group"}' --insecure
```

Record the `model_group_id` from the response (e.g., `ehCiRpYBdxx48goUAwgh`).

### Step 3: Register a Pre-trained Model

Register the huggingface/sentence-transformers/all-MiniLM-L6-v2 model:
```bash
curl -XPOST https://localhost:9200/_plugins/_ml/models/_register -u 'admin:YourStrong@Passw0rd' -H 'Content-Type: application/json' -d '{
  "name": "huggingface/sentence-transformers/all-MiniLM-L6-v2",
  "version": "1.0.1",
  "model_group_id": "<your_model_group_id>",
  "model_format": "TORCH_SCRIPT"
}' --insecure
```

Replace `<your_model_group_id>` with the ID from Step 2.
Record the `task_id` from the response (e.g., `exCiRpYBdxx48goU8AgS`).

### Step 4: Monitor Registration Task

Check the task status:
```bash
curl -XGET https://localhost:9200/_plugins/_ml/tasks/<task_id> -u 'admin:YourStrong@Passw0rd' --insecure
```

Wait until state is `COMPLETED` (may take 1-5 minutes depending on network speed).
Record the `model_id` from the response (e.g., `xyz123`).

> **Note**: If the task remains in `CREATED` for over 10 minutes, see [Troubleshooting](#troubleshooting-common-issues).

### Step 5: Deploy the Model

Deploy the model using the model_id:
```bash
curl -XPOST https://localhost:9200/_plugins/_ml/models/<model_id>/_deploy -u 'admin:YourStrong@Passw0rd' --insecure
```

Verify deployment:
```bash
curl -XGET https://localhost:9200/_plugins/_ml/models/<model_id> -u 'admin:YourStrong@Passw0rd' --insecure
```

Ensure status is `DEPLOYED`.

### Step 6: Test the Model

Test the model with a text embedding prediction:
```bash
curl -XPOST https://localhost:9200/_plugins/_ml/_predict/text_embedding/<model_id> -u 'admin:YourStrong@Passw0rd' -H 'Content-Type: application/json' -d '{
  "text_docs": ["today is sunny"],
  "return_number": true
}' --insecure
```

Expected output: A 384-dimensional vector embedding.

## Verifying in OpenSearch Dashboards

1. Open a browser and navigate to http://localhost:5601
2. Log in with username `admin` and password `YourStrong@Passw0rd`
3. Go to Machine Learning > Deployed Models
4. Verify that the model appears with a green status

If no models are listed, see [Troubleshooting](#troubleshooting-common-issues).

## Troubleshooting Common Issues

### Issue 1: Model Registration Task Stalls in CREATED

**Symptoms**: Task status remains `CREATED` for over 10 minutes.

**Causes**: Network issues, insufficient resources, or plugin errors.

**Solutions**:
1. Check logs:
   ```bash
   docker logs opensearch-node1
   ```
   Look for errors related to ML or huggingface.

2. Verify network connectivity:
   ```bash
   curl -I https://huggingface.co
   ```

3. Increase Docker Desktop resources (8GB memory, 4 CPUs).

4. Cancel and retry the task:
   ```bash
   curl -XDELETE https://localhost:9200/_plugins/_ml/tasks/<task_id> -u 'admin:YourStrong@Passw0rd' --insecure
   ```
   Redo Step 3.

### Issue 2: Deployment Fails with Failed to find model (404)

**Symptoms**: curl -XPOST .../_deploy returns a 404 error.

**Causes**: Incorrect model_id (e.g., using model_group_id).

**Solutions**:
1. Ensure you use the model_id from the completed task, not the model_group_id.

2. Check available models:
   ```bash
   curl -XGET https://localhost:9200/_plugins/_ml/models -u 'admin:YourStrong@Passw0rd' --insecure
   ```

### Issue 3: Dashboards Shows "Deployed models will appear here"

**Symptoms**: No models listed in the Machine Learning section.

**Causes**: Model not registered/deployed, or Dashboards configuration issue.

**Solutions**:
1. Verify model deployment (see Step 5).

2. Check Dashboards logs:
   ```bash
   docker logs opensearch-dashboards
   ```
   Look for Connection refused or Timeout errors.

3. Ensure `ml_commons_dashboards.enabled=true` in docker-compose.yml.

4. Test ML API:
   ```bash
   curl -XGET https://localhost:9200/_plugins/_ml/models -u 'admin:YourStrong@Passw0rd' --insecure
   ```

### Issue 4: QueryGroup _id can't be null Warning

**Symptoms**: Logs show `QueryGroup _id can't be null, It should be set before accessing it. This is abnormal behaviour.`

**Causes**: Possible bug in query-insights or opensearch-ml plugin.

**Solutions**:
1. Check .plugins-ml-model index:
   ```bash
   curl -XGET https://localhost:9200/.plugins-ml-model/_search -u 'admin:YourStrong@Passw0rd' --insecure
   ```

2. Rebuild index (if empty or corrupted):
   ```bash
   curl -XDELETE https://localhost:9200/.plugins-ml-model -u 'admin:YourStrong@Passw0rd' --insecure
   ```
   Redo Step 3.

3. Disable query-insights (temporary):
   ```yaml
   environment:
     - plugins.query_insights.enabled=false
   ```
   Restart containers:
   ```bash
   docker compose down && docker compose up -d
   ```

4. Report to OpenSearch GitHub.

### Issue 5: Dashboards Fails to Load

**Symptoms**: http://localhost:5601 returns errors (e.g., 502, timeout).

**Causes**: Resource constraints, network issues, or misconfiguration.

**Solutions**:
1. Check container status:
   ```bash
   docker compose ps
   ```

2. Inspect logs:
   ```bash
   docker logs opensearch-dashboards
   ```

3. Verify OPENSEARCH_HOSTS in docker-compose.yml.

4. Test connectivity from Dashboards container:
   ```bash
   docker exec -it opensearch-dashboards bash
   curl -XGET https://opensearch-node1:9200 -u 'admin:YourStrong@Passw0rd' --insecure
   ```

## References

- [OpenSearch Documentation](https://opensearch.org/docs/)
- [ML Commons Plugin](https://opensearch.org/docs/latest/ml-commons-plugin/)
- [OpenSearch Forum](https://forum.opensearch.org/)
- [Docker Desktop Documentation](https://docs.docker.com/desktop/)

---
*Last Updated: April 17, 2025*
