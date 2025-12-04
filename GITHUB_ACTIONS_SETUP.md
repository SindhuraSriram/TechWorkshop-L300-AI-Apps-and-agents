# GitHub Actions Deployment Setup

## Overview
This document explains the GitHub Actions workflow that builds and deploys your application to Azure Container Registry (ACR).

## Workflow File
- **Location**: `.github/workflows/deploy-acr.yml`
- **Trigger**: Pushes to the `main` branch (when files in `src/` or the workflow itself change)
- **Actions**: Builds Docker image and pushes to ACR

## Required GitHub Secrets

You need to set up the following secrets in your GitHub repository settings (`Settings > Secrets and variables > Actions`):

### 1. `AZURE_CONTAINER_REGISTRY`
- **Value**: Your Azure Container Registry name (e.g., `myregistry` NOT the full login server)
- **How to find it**: 
  ```bash
  az acr show --resource-group <your-rg> --name <your-acr-name> --query name -o tsv
  ```

### 2. `AZURE_CONTAINER_REGISTRY_USERNAME`
- **Value**: Your ACR username
- **How to find it**: 
  ```bash
  az acr credential show --resource-group <your-rg> --name <your-acr-name> --query username -o tsv
  ```

### 3. `AZURE_CONTAINER_REGISTRY_PASSWORD`
- **Value**: Your ACR access key (password)
- **How to find it**: 
  ```bash
  az acr credential show --resource-group <your-rg> --name <your-acr-name> --query "passwords[0].value" -o tsv
  ```

### 4. `ENV`
- **Value**: The contents of your `.env` file (include all environment variables needed for your application)
- **Example content**:
  ```
  AZURE_OPENAI_API_KEY=your-key-here
  AZURE_OPENAI_ENDPOINT=https://your-instance.openai.azure.com/
  AZURE_SEARCH_SERVICE_ENDPOINT=https://your-search.search.windows.net/
  # ... other environment variables
  ```

## How It Works

1. **Trigger**: When code is pushed to the `main` branch (or manually via workflow_dispatch)
2. **Checkout**: Repository code is checked out
3. **Create .env**: The `ENV` secret is written to `src/.env` file
4. **ACR Login**: GitHub Actions logs into your Azure Container Registry using the secrets
5. **Build & Push**: Docker image is built from `src/Dockerfile` with:
   - Context: `src/` folder (cd into src before building)
   - Image name: `your-registry/chat-app:latest`
   - All dependencies from `requirements.txt` are installed
   - `.env` file is copied into the image
6. **Push**: Image is pushed to ACR

## Security Considerations

✅ **What's Protected**:
- `.env` file is **NOT** committed to git (checked in `.gitignore`)
- Secrets are **NEVER** logged or exposed in workflow output
- ACR credentials are stored securely in GitHub Secrets
- `.env` content is only used during Docker build, not stored in the image as a secret layer

⚠️ **Important Notes**:
- The `.env` content passed as a build argument will be visible in the image layers if someone inspects the image
- For production, consider using:
  - [Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/) for secret management
  - [App Configuration](https://learn.microsoft.com/en-us/azure/app-configuration/) for configuration
  - Runtime environment variables instead of .env files

## Manual Testing

To test locally before pushing:

1. Create a `.env` file in the `src/` folder (it's in `.gitignore` so won't be committed)
2. Build the image:
   ```bash
   cd src
   docker build -t chatbot:test .
   ```
3. Run the container:
   ```bash
   docker run -p 8000:8000 chatbot:test
   ```

## Modifying the Workflow

To customize the workflow:
- **Change Docker image name**: Modify the `tags` in the `build-and-push` step
- **Add different triggers**: Update the `on:` section
- **Add more build steps**: Add new steps before `build-and-push`
- **Customize context or Dockerfile path**: Update `context` and `file` fields

## Troubleshooting

**ACR Login fails**:
- Verify ACR username/password are correct
- Check that your ACR has admin access enabled
- Confirm the ACR_LOGIN_SERVER is in the correct format

**Secrets not found**:
- Go to `Settings > Secrets and variables > Actions` to verify secrets exist
- Ensure secret names match exactly (case-sensitive)

**Image build fails**:
- Check Dockerfile syntax
- Verify all COPY commands reference correct paths
- Review `requirements.txt` for dependency issues
