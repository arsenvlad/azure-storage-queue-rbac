# Azure Storage Queue RBAC across Azure AD tenants

Document describing Azure Storage Authentication with Azure AD (Preview)
https://docs.microsoft.com/en-us/rest/api/storageservices/authenticate-with-azure-active-directory

Select ISV subscription
```
az account list
az account set --subscription "isv"
```

Create multi-tenanted Azure AD application in the ISV tenant using https://portal.azure.com/ with a password key and required resource access to Azure AD
```
az ad app create --display-name avmultitenant2 --available-to-other-tenants true --homepage http://localhost/avmultitenant2 --reply-urls http://localhost/avmultitenant2 --identifier-uris https://this-domain-must-match-your-tenant-url.com/avmultitenant2 --key-type Password --password PutSecureKeyValueHere123 --required-resource-accesses app-required-resource-accesses.json
```

Admin consent customer subscription to the multi-tenanted application created above by visting URL like below in a browser
```
https://login.microsoftonline.com/{customer-tenant}/adminconsent?client_id={isv-application-id}&state={any-state-to-pass}&redirect_uri={isv-application-return-url}
```

Switch to the cutomer subscription and tenant
```
az account set --subscription "customer"
```

List existing custom roles
```
az role definition list --custom-role-only true
```

Create custom role for Storage Queue Processor (read/delete) after setting proper subscription id in the .json file definition for the role
```
az role definition create --role-definition custom-role-queue-processor.json
```

Create storage account and queue in the customer subscription (if you don't have them yet)
```
az group create --name avegs1 --location eastus2
az storage account create --resource-group avegs1 --sku Standard_LRS --kind StorageV2 --name avegs1
az storage queue create --account-name avegs1 --name queue1
export storageAccountResourceId=$(az storage account show --resource-group avegs1 --name avegs1 --query id -o tsv)
export queueResourceId=$storageAccountResourceId/queueServices/default/queue1
export ak=$(az storage account keys list --resource-group avegs1 --account-name avegs1 --query "[0].value" -o tsv)
```

Grant newly created "Storage Queue Processor Custom Role" definition to the multi-tenanted ISV application Service Principal in customer's subscription to the queue
(Azure CLI example below creates the assignment at the scope of storage account because az cli does not yet have version 2018-03-01-preview for assigned roles to queueServices. For now, use https://portal.azure.com or an ARM template to create the role assignment at queue scope)
```
az ad list sp --display-name avmultitenant2
az role assignment create --role "Storage Queue Processor Custom Role" --assignee {service-principal-id} --scope "$storageAccountResourceId"
```

ISV can now obtain access_token using their application's client_id and client_secret and customer's tenant
```
curl -X POST 'https://login.microsoftonline.com/{customer-tenant}/oauth2/token' -d 'grant_type=client_credentials&resource=https://storage.azure.com/&client_id={isv-application-service-principal-id}&client_secret={isv-application-secret}'
```

Set access_token environment variable to make it easier to make curl calls shown below
```
export access_token="{value-returned-by-login-oauth2-token}"
```

Use the obtained access_token to peek at messages in the queue
```
curl -X GET -H 'x-ms-version: 2017-11-09' -H "Authorization: Bearer $access_token" https://avegs1.queue.core.windows.net/queue1/messages?peekonly=true
```

Trying adding a new message to queue. It should fail because the custom role  created allows only read and delete but not write
```
curl -X POST -H 'x-ms-version: 2017-11-09' -H "Authorization: Bearer $access_token" https://avegs1.queue.core.windows.net/queue1/messages -d '<QueueMessage><MessageText>SGVsbG8gV29ybGQh</MessageText></QueueMessage>'
```

Error returned will be similar to
```
<?xml version="1.0" encoding="utf-8"?>
<Error><Code>AuthorizationPermissionMismatch</Code>
<Message>This request is not authorized to perform this operation using this permission.
RequestId:eacd4d79-e003-0031-304e-30da8d000000
Time:2018-08-10T02:03:34.2897261Z</Message></Error>
```

Get messages from the queue including popreceipt value
```
curl -X GET -H 'x-ms-version: 2017-11-09' -H "Authorization: Bearer $access_token" https://avegs1.queue.core.windows.net/queue1/messages
```

If there are messages in the queue, try to delete one by passing the message id and corresponding popreceipt value obtained from get messages
```
curl -X DELETE -H 'x-ms-version: 2017-11-09' -H "Authorization: Bearer $access_token" https://avegs1.queue.core.windows.net/queue1/messages/{messageid}?popreceipt={popreceipt-string-from-get-messages}
```

CURL calls above show how to interact with the queue using raw REST API. Azure Storage Java SDK (and others) provide similar functionality and support OAuth tokens as of 2018.05.22 Version 7.1.0

* Example Azure Storage Java SDK code showing how to obtain OAuth token using client credential
https://github.com/Azure/azure-storage-java/blob/07fba08e00d3fa29161b8fd9ad0042c5f932cd78/microsoft-azure-storage-test/src/com/microsoft/azure/storage/TestHelper.java#L82

* Example Azure Storage Java SDK code showing how to create StorageCredentialsToken from the OAuth token:
https://github.com/Azure/azure-storage-java/blob/4195c3286c508f93d446b600d9ccd54e2eb9deed/microsoft-azure-storage-test/src/com/microsoft/azure/storage/OAuthTests.java#L61

* Example Azure Storage Java SDK code for interacting with Azure Queue
https://github.com/Azure/azure-storage-java/blob/master/microsoft-azure-storage-samples/src/com/microsoft/azure/storage/queue/gettingstarted/QueueBasics.java

