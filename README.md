# 🚀 Step-by-Step Guide: Deploying an ASP.NET API via ACR and Automating CI/CD with GitHub Actions

This guide covers two parts:
1️⃣ **Manually deploying the API via Azure Container Registry (ACR) and Web App Service** (to **eliminate common issues**)
2️⃣ **Automating deployments with GitHub Actions** (CI/CD pipeline)

---

## **📌 Part 1: Manual Deployment via ACR and Web App Service**
This ensures everything works **before automating CI/CD**.

### **✅ Step 1: Create and Test the API**
1. **Create the API project** (ASP.NET Core Web API or Spring boot)
   ```sh
   dotnet new webapi -n MyApi
   cd MyApi
   ```
2. **Test the API locally**
   ```sh
   dotnet run
   ```
   Open `https://localhost:5001/weatherforecast` in a browser.

3. **Create a Dockerfile** inside the API folder:
   ```Dockerfile
   # 1 - Base image to work from. SDK for .NET
    FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build-stage
    # 2 - Set working directory
    WORKDIR /app
    # 3 - Copy source files to the image
    COPY . ./
    # 4 - Run 'dotnet restore' to install missing dependencies
    RUN dotnet restore 
    # 5 - Publish our application to the image
    RUN dotnet publish -c Release -o out

    # 6 - Runtime stage
    FROM mcr.microsoft.com/dotnet/aspnet:8.0
    # 7 - re-establish the working directory
    WORKDIR /app
    # 8 - Copy over the published files (the rest will be discarded)
    COPY --from=build-stage /app/out .
    # 9 - Configure how the application is run
    ENTRYPOINT ["dotnet", "weatherstationCICD.dll"]
   ```
4. **Build and test the Docker image locally**
   ```sh
   docker build -t aspcicd .
   docker run -d -p 8080:80 aspcicd
   ```
   Open `http://localhost:8080/weatherforecast` to verify it works.

---

### **✅ Step 2: Set Up a GitHub Repository**
1. **Initialize Git**
   ```sh
   git init
   git add .
   git commit -m "Initial API setup"
   ```
2. **Create a `.gitignore` file**:
   ```
   bin/
   obj/
   .vscode/
   .idea/
   .DS_Store
   ```
3. **Push the code to GitHub**
   ```sh
   git remote add origin https://github.com/your-username/apicicd.git
   git branch -M main
   git push -u origin main
   ```

---

### **✅ Step 3: Create and Push Image to Azure Container Registry (ACR)**
#### **1️⃣ Install and Configure Azure CLI**
1. **Install Azure CLI**: [Download & Install](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
2. **Login to Azure**
   ```sh
   az login
   ```
3. **Set your subscription** (if needed)
   ```sh
   az account set --subscription "your-subscription-id"
   ```

---

#### **2️⃣ Create Azure Container Registry**
```sh
az acr create --resource-group MyResourceGroup --name loopacademy --sku Basic
```

---

#### **3️⃣ Enable ACR Admin Access**
```sh
az acr update --name loopacademy --admin-enabled true
```

---

#### **4️⃣ Log in to ACR**
```sh
az acr login --name loopacademy
```

---

#### **5️⃣ Tag and Push the Image**
```sh
docker tag aspcicd:latest loopacademy.azurecr.io/aspcicd:v1
docker push loopacademy.azurecr.io/aspcicd:v1
```

---

#### **6️⃣ Verify Image in ACR**
```sh
az acr repository list --name loopacademy --output table
```

---

### **✅ Step 4: Deploy the API to Azure Web App**
1. **Create an Azure App Service Plan**
   ```sh
   az appservice plan create --name myAppServicePlan --resource-group MyResourceGroup --sku B1 --is-linux
   ```
2. **Create the Web App and Connect to ACR**
   ```sh
   az webapp create --resource-group MyResourceGroup --plan myAppServicePlan --name aspcicd --deployment-container-image-name loopacademy.azurecr.io/aspcicd:v1
   ```
3. **Set App Service to Pull From ACR**
   ```sh
   az webapp config container set --name aspcicd --resource-group MyResourceGroup --docker-custom-image-name loopacademy.azurecr.io/aspcicd:v1
   ```
4. **Restart Web App**
   ```sh
   az webapp restart --name aspcicd --resource-group MyResourceGroup
   ```
5. **Test Deployment**: `https://aspcicd.azurewebsites.net/weatherforecast`

---

## **📌 Part 2: Automating CI/CD with GitHub Actions**

### **✅ Step 5: Set Up GitHub Secrets**
Go to **GitHub → Your Repo → Settings → Secrets and Variables → Actions** and add these secrets:

| Secret Name | Value |
|---------------|-----------|
| `AZURE_CLIENT_ID` | **Azure App Registration → Client ID** |
| `AZURE_TENANT_ID` | **Azure AD → Tenant ID** |
| `AZURE_SUBSCRIPTION_ID` | **Azure Subscription ID** |
| `ACR_LOGIN_SERVER` | `loopacademy.azurecr.io` |
| `ACR_USERNAME` | **Azure Portal → ACR → Access Keys** |
| `ACR_PASSWORD` | **Azure Portal → ACR → Access Keys** |
| `WEBAPP_NAME` | `aspcicd` |
| `RESOURCE_GROUP` | `MyResourceGroup` |

---

### **✅ Step 6: Configure GitHub Actions (CI/CD Pipeline)**
Create `.github/workflows/deploy.yml`:
```yaml
name: Build and Deploy to Azure

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
      
      - name: Build and Push Docker Image
        run: |
          IMAGE_TAG=v1
          docker build -t ${{ secrets.ACR_LOGIN_SERVER }}/aspcicd:$IMAGE_TAG -f MyProject/Dockerfile MyProject/
          docker push ${{ secrets.ACR_LOGIN_SERVER }}/aspcicd:$IMAGE_TAG

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    
    steps:
      - name: Deploy to Azure Web App
        run: |
          az webapp config container set --name ${{ secrets.WEBAPP_NAME }} --resource-group ${{ secrets.RESOURCE_GROUP }} --docker-custom-image-name ${{ secrets.ACR_LOGIN_SERVER }}/aspcicd:v1
          az webapp restart --name ${{ secrets.WEBAPP_NAME }} --resource-group ${{ secrets.RESOURCE_GROUP }}
```

---

🚀 **Now, every push automatically updates the deployment to Azure!**

