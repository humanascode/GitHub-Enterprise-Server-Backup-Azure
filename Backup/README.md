# Backup

## Introduction

In this guide, we will walk through the steps to configure the backup process for GitHub Enterprise Server using Azure Container Instances and Azure Files. Here are the steps that will be done and a short explanation:

1. Keys generation and configuration:
    - SSH key pair generation for authentication between the backup container and the GitHub Enterprise Server instance.
    - Host key retrieval from the GitHub Enterprise Server instance - for the backup container to trust the GitHub Enterprise Server instance.
    - Store the keys in a key vault - The backup container will retrieve the keys dyanmically every time it runs using a managed identity.
    - Configure the public key in the GitHub Enterprise Server instance - to allow the backup container to connect to the GitHub Enterprise Server instance.
2. Docker image preparation:
    - Download the backup tools from GitHub. The backup tools come with a Dockerfile that is ready to build. We will be adding the Azure CLI to the Docker image to interact with the key vault. Also, we will be adding a wrapper script to prepare the environment (pull keys from the key vault) and call the backup script. After the backup is done, The wrapper script will archive the backup and upload it to Azure Files.
3. Azure Storage Account and file share:
    - Azure Container Instance currently only supports mounting Azure Files as volumes as a persistent storage option. We will create a storage account and a file share to store the backups.
4. Azure Container Instance deployment:
    - We will create an Azure Container Instance with the Docker image we built earlier. The container will be deployed in a subnet that has line of sight to the GitHub Enterprise Server instance. We will also create a user-assigned managed identity and assign it to the container instance to access the Azure Container Registry and the key vault.
5. Automation Account and Runbook:
    - We will create an Automation Account and a Runbook to trigger the backup on a schedule.


## Step-By-Step Instructions:
1. Create a key vault (or use an existing one).
2. Create an ssh key pair and store the private key in the key vault (Make sure you have RBAC permissions to write secrets to the key vault):
    ```bash
    # Generate an SSH key pair
    ssh-keygen -t rsa -b 4096 -N "" -C "backup@yourdomain.com" -f ~/.ssh/ghes_backup_key

    # Store the private key in the key vault
    az keyvault secret set --vault-name <key-vault-name> --name ghes-backup-ssh-private-key --file ~/.ssh/ghes_backup_key

    # Store the public key in the key vault
    az keyvault secret set --vault-name <key-vault-name> --name ghes-backup-ssh-public-key --file ~/.ssh/ghes_backup_key.pub
    ```
    > **Note:** This linux bash command can be run on a linux machine or on Windows using WSL.
3. Add the public key to the GitHub Enterprise Server instance:
    - Go to the GitHub Enterprise Server Admin Console (usualy on port 8443)
    - Go to `Settings` -> `SSH access`.
    - Paste the public key in the `Add SSH key` box.
    - Click `Add key`.
4. From a remote console with line of sight of the GHES, Retrieve the SSH host key from the GitHub Enterprise Server instance:
    ```bash
    # Retrieve the SSH host key and store it directly in the key vault
    ssh-keyscan -H private.github.com | az keyvault secret set --vault-name <key-vault-name> --name ghes-host-key --value @-
    ```
5. Download the backup tools from here (use the appropriate version for you environment):
https://github.com/github/backup-utils/releases
6. Extract the tools into a new folder.
7. Open the Dockerfile and add the following lines just above the 'ENTRYPOINT' / 'CMD' line:
    ```Dockerfile
    RUN apt-get update && apt-get install --no-install-recommends -y curl
    RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash
    ```
    These lines will install the Azure CLI in the container. which we will use to interact with the key vault.
8. Remove any 'CMD' or 'ENTRYPOINT' lines from the Dockerfile and replace them with the following:
    ```Dockerfile
    CMD ["sh", "/backup-utils/bin/init-backup"]
    ```
    This is our wrapper script that prepares the environment and calls the backup script.
9. Copy the [init-backup](./init-backup) and [init-restore](./init-restore) scripts from this repository to the `bin` folder. These are our wrappers.
10. Create an Azure Container Registry (if you don't have one already):
    ```bash
    az acr create --resource-group <resource-group-name> --name <acr-name> --sku <Basic | Standard | Premium>
    ```
    > **Note:** For private links support you need to use the Premium SKU.
11. Navigate to the folder containing the Dockerfile.
12. Build and push the Docker image to the Azure Container Registry:
    ```bash
    az acr login --name <acr-name>
    docker build --push -t <acr-name>.azurecr.io/ghes-backup:1.0 .
    ```
13. Create a storage account and a file share for storing the backups:
    ```bash
    az storage account create --resource-group <resource-group-name> --name <storage-account-name> --sku Standard_LRS
    az storage share create --name  ghes-backup --account-name <storage-account-name>
    ```
    > **Note:** The storage account and the container instance must be in the same region.
14. Creating an Azure Container Instance:
    - The container instance needs to be deployed into a subnet that has a line of sight to the GitHub Enterprise Server instance. This could be in the same VNet, a peered VNet, or a VNet connected Hub.
    - The container must be deployed in the same subscriptionas the vnet.
    - The subnet should be empty and configured with delegation to "Microsoft.ContainerInstance/containerGroups". This can be created with the following command:
        ```bash
            # Create a subnet with delegation to "Microsoft.ContainerInstance/containerGroups"
            az network vnet subnet create --resource-group <resource-group-name> --vnet-name <vnet-name> --name <subnet-name> --address-prefix <subnet-address-prefix> --delegations "Microsoft.ContainerInstance/containerGroups"
        ```
    - Create a user-assigned managed identity and assign to it the `Acr Pull` role on the Azure Container Registry:
        ```bash
            # Create a user-assigned managed identity
            az identity create --resource-group <resource-group-name> --name <managed-identity-name>

            # Assign the Acr Pull role to the managed identity
            az role assignment create --role "AcrPull" --assignee <managed-identity-client-id> --scope /subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.ContainerRegistry/registries/<acr-name>
        ```
    - Grant the managed identity access to the key vault:
        ```bash
            # Grant the managed identity access to the key vault
            az role assignment create --role "Key Vault Secrets User" --assignee <managed-identity-client-id> --scope /subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.KeyVault/vaults/<key-vault-name>
        ```
    - Create the container instance with managed identity access to ACR:

        ```bash
          az container create --resource-group <resource-group-name> \
          --name <container-name> --image <acr-name>.azurecr.io/ghes-backup:1.0 \
          --cpu 4 --memory 8 --os-type Linux --restart-policy Never \
          --vnet <vnet-resource-id> \
          --subnet <subnet-id> --assign-identity <user-assigned-managed-identity-resource-id> \
          --acr-identity <user-assigned-managed-identity-resource-id> \
          --azure-file-volume-share-name ghes-backup \
          --azure-file-volume-account-name <storage-account-name> \
          --azure-file-volume-account-key <storage-account-key> \
          --azure-file-volume-mount-path /mnt/data \
          --environment-variables \
              KEY_VAULT_NAME=<key-vault-name> \
              SECRET_NAME=ghes-backup-ssh-private-key \
              GH_HOST_KEY_SECRET_NAME=ghes-host-key \ 
              GHE_HOSTNAME=<ghes hostname> \
              SUBSCRIPTION_ID=<key-vault-subscription-id> \
              DAYS_RETENTION=30 \
              MIN_BACKUP_NUMBER=10
        ```
        > **Note:**
        >* The --azure-file-volume-mount-path must be set to /mnt/data. This is where the script expects the backup to be stored. 
        >* The cpu and memory are set to the recommended values from GitHub  
        >* In the `vnet` and `subnet` parameters, use the resource ID of the VNet and the subnet where the container will be deployed.  
        >* Use the managed identity you created earlier for the `--assign-identity` and `--acr-identity` parameters.
        >* The `--environment-variables` are the environment variables that the backup script expects.
        >* The `DAYS_RETENTION` and `MIN_BACKUP_NUMBER` are the number of days to keep the backups and the minimum number of backups to keep respectively.
        >* The `GHE_HOSTNAME` is the hostname of the GitHub Enterprise Server instance. - You need to make sure the container can resolve this name. Checkout the next section for DNS configuration.

15. DNS Configuration:
    - The ACI automatically uses Azure DNS, If you link an Azure Private DNS Zone to the VNET where the ACI is deployed, It will resolve it.
    - If you are using a custom DNS server, you will need to configure the DNS settings in the container group. Follow the instructions here: https://learn.microsoft.com/en-us/azure/container-instances/container-instances-custom-dns

16. Create an Automation Account to trigger the backup on a schedule:
    - Create an Automation Account in the Azure Portal:
    ```bash
    az automation account create --name <automation-account-name> --resource-group <resource-group-name> --location <location>
    ```
    - Enable the system assigned managed identity for the Automation Account from the Azure Portal:  
      * Go to the Automation Account -> Identity -> Status -> On
    - Grant the managed identity access to the ACI:
    ```bash
    az role assignment create --role "Contributor" --assignee <automation-account-client-id> --scope /subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.ContainerInstance/containerGroups/<container-name>
    ```
    - Create a Runbook in the Automation Account:
    ```bash
    az automation runbook create --resource-group <resource-group-name> 
    --automation-account-name <automation-account-name> --name <runbook-name> --type PowerShell
    ```
    - Edit the runbook in the portal and paste the bellow script.
    Go to the Automation Account -> Runbooks -> <runbook-name> -> Edit
    ```powershell
    Connect-AzAccount -Identity
    Start-AzContainerGroup -Name <aci name> -ResourceGroupName <resource group name>
    ```
    Click save and publish.
    - Create a schedule for the runbook:
    ```bash
    az automation schedule create --resource-group <resource-group-name> --automation-account-name <automation-account-name> --schedule-name <schedule-name> --frequency Day interval 1 --start-time <start-date-and-time> --timezone <timezone>
    ```
    - Link the schedule to the runbook from the Azure Portal:
    Go to the Automation Account -> Runbooks -> `<runbook-name>` -> Schedules -> Add a schedule -> Select the schedule you created.





After finishing the above steps, you should have a working solution. You can test it manually by running the container instance from the portal and checking the logs. It is recommended to link the ACI to a log analytics workspace so you can view the logs after the run is finished. Just go to the 'logs' blade in the ACI and configure the log analytics workspace.
