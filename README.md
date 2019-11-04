# Deploy NodeJs App to Azure Kubernetes service cluster by using GitHub actions
Sample workflow which uses GitHub actions to build and deploy a NodeJS app to a **Azure Kubernetes Service (AKS)** cluster

## End to end workflow for building container images and deploying to an AKS cluster

```yaml
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@master
    
    # Connect to Azure Container registry (ACR)
    - uses: azure/docker-login@v1
      with:
        login-server: demo.azurecr.io
        username: ${{ secrets.REGISTRY_USERNAME }} 
        password: ${{ secrets.REGISTRY_PASSWORD }}
    
    # Docker build and push to a Azure Container registry (ACR)
    - run: |
        docker build . -t demo.azurecr.io/k8sdemo:${{ github.sha }}
        docker push demo.azurecr.io/k8sdemo:${{ github.sha }}
    
    # Set the target AKS cluster. 
    - uses: azure/aks-set-context@v1
      with:
        creds: '${{ secrets.AZURE_CREDENTIALS }}'
        cluster-name: desattir
        resource-group: desattir
    
    # Create imagepullsecret for Azure Container registry (ACR)
    - uses: azure/k8s-create-secret@v1
      with:
        container-registry-url: demo.azurecr.io
        container-registry-username: ${{ secrets.REGISTRY_USERNAME }}
        container-registry-password: ${{ secrets.REGISTRY_PASSWORD }}
        secret-name: demo-k8s-secret
    
    # Deploy app to AKS
    - uses: azure/k8s-deploy@v1
      with:
        manifests: |
          manifests/deployment.yml
          manifests/service.yml
        images: |
          demo.azurecr.io/k8sdemo:${{ github.sha }}
        imagepullsecrets: |
          demo-k8s-secret
        # namespace: <optional>
```

### To test drive this sample NodeJS app using GitHub actions for AKS 

* You can create a separate branch using the sample code in master 
* To deploy the sample web app to your AKS cluster:
    * Create an Azure SPN by running the following command.
    ```yaml
    $ az ad sp create-for-rbac --name build-demo-k8s --scopes /subscriptions/avb1291-****-46be-****-70349146ddrr/resourceGroups/build-demo --role contributor --sdk-auth
    ```
    In the sample workflow of this repo we are using Azure SPN but other options are also supported. You can also use Kubeconfig [az aks get-credentials](https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest#az-aks-get-credentials) or for any other cluster refer to [Kubernetes](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) documentation. The [`azure/k8s-actions/k8s-set-context`](https://github.com/Azure/k8s-actions/tree/master/k8s-set-context) action supports Kubernetes service account as well.
    
    * Define a new secret in the [repository](https://github.com/bbq-beets/k8s/settings/secrets) >> “Add a new secret”  
    * Paste the output of the 'az ad sp...' command into the secret value field and add your secret.
    ```json
    {
  "clientId": "7*********8",
  "clientSecret": "******",
  "subscriptionId": "qvb11291-****-****-****-70349**46dre8",
  "tenantId": "72****bf-****-****-****-*****011db4*",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
  }
  ```
    * Similarly get the username and password of your container registry and create secrets for them. For Azure Container registry refer to admin [account document](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-authentication#admin-account) for username and password.
    * Now in the workflow file in your branch: `.github/workflows/workflow.yml`
        * Replace the secret name with your secret (Refer below)
    * Review the **container registry url and manifest file paths**. 
    
    ```yaml      
      # Connect to a container registry
    - uses: azure/docker-login@v1
      with:
        login-server: demo.azurecr.io
        username: ${{ secrets.REGISTRY_USERNAME }} 
        password: ${{ secrets.REGISTRY_PASSWORD }}
    ```
    
    ```yaml      
      # Set the target AKS cluster. 
    - uses: azure/k8s-actions/k8s-set-context@master
      with:
        kubeconfig: ${{ secrets.KUBE_CONFIG }}
    ```
    
    ```yaml
    # Create imagepullsecret
    - uses: azure/k8s-create-secret@v1
      with:
        container-registry-url: demo.azurecr.io
        container-registry-username: ${{ secrets.REGISTRY_USERNAME }}
        container-registry-password: ${{ secrets.REGISTRY_PASSWORD }}
        secret-name: demo-k8s-secret
     ```

## Build and deploy
In the sample workflow above we have setup build and deploy in the same job. You can use multiple jobs as well. For example one job for build followed by a job for deploying to staging cluster or deploying to production.

```yaml
jobs:
  my_first_job:
    name: My first job
  my_second_job:
    name: My second job
```

# Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.microsoft.com.

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

# Legal Notices

Microsoft and any contributors grant you a license to the Microsoft documentation and other content
in this repository under the [Creative Commons Attribution 4.0 International Public License](https://creativecommons.org/licenses/by/4.0/legalcode),
see the [LICENSE](LICENSE) file, and grant you a license to any code in the repository under the [MIT License](https://opensource.org/licenses/MIT), see the
[LICENSE-CODE](LICENSE-CODE) file.

Microsoft, Windows, Microsoft Azure and/or other Microsoft products and services referenced in the documentation
may be either trademarks or registered trademarks of Microsoft in the United States and/or other countries.
The licenses for this project do not grant you rights to use any Microsoft names, logos, or trademarks.
Microsoft's general trademark guidelines can be found at http://go.microsoft.com/fwlink/?LinkID=254653.

Privacy information can be found at https://privacy.microsoft.com/en-us/

Microsoft and any contributors reserve all others rights, whether under their respective copyrights, patents,
or trademarks, whether by implication, estoppel or otherwise.

