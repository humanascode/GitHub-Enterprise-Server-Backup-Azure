# Restore

## Introduction

In this guide, we will walk you through the process of restoring a GitHub Enterprise Server instance using the backup files created by the backup process. The restore process is a manual process that requires you to create a container instance and provide the backup file to restore from. Here are the steps and details:

1. Creating another container instance from the exact same container image that was used for the backup process. The only difference here is that we will override the CMD to run the restore script. all the rest is the same as the backup process.
2. To restore, you need to create a folder in the file share called "restore" and place the backup file you want to restore from there. Next you will need to put the Github instance into maintanance mode and then run the restore container. The restore process will restore to the host that is specified in the `GHE_HOSTNAME` environment variable. Make this this is passed correctly whethere you are restoring to the same host or a different one. If you are unsure of the restore process always test restore to a different host to avoid any data loss.

Instructions:

1. Creating an Azure Container Instance:
    - We will be using the same container image that we used for the backup process.
    - We will override the CMD to run the restore script.
    - The container instance needs to be deployed into a subnet that has a line of sight to the GitHub Enterprise Server instance. This could be in the same VNet, a peered VNet, or a VNet connected Hub.
    - The container must be deployed in the same subscriptionas the vnet.
    - The subnet should be empty and configured with delegation to "Microsoft.ContainerInstance/containerGroups".
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
          --command-line "sh /backup-utils/bin/init-restore"
          --environment-variables \
              KEY_VAULT_NAME=<key-vault-name> \
              SECRET_NAME=ghes-backup-ssh-private-key \
              GH_HOST_KEY_SECRET_NAME=ghes-host-key \ 
              GHE_HOSTNAME=<ghes hostname> \
              SUBSCRIPTION_ID=<key-vault-subscription-id> \
              DAYS_RETENTION=30 \
              MIN_BACKUP_NUMBER=10
        ```
        > **Note:** The --azure-file-volume-mount-path must be set to /mnt/data. This is where the script expects the backup to be stored.