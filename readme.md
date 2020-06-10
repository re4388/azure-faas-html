# Tutorial I followed
https://mikepfeiffer.io/blog/azure-serverless-101

## 1. Setup your storage a/c for static hosting
check az a/c
`az account list`

set az a/c if you got many account
`az account set --subscription <YOUR SUBSCRIPTION ID>`

see all region available
`az account list-locations`

### 1.1 Create a resource group
u need to this to store the resources for the application
`az group create --location <YOUR REGION> --name <YOUR GROUP NAME>`
ex
`az group create --location westus2 --name g3`

### 1.2 Create the Storage Account
- use a globally unique DNS-compliant name(see PS1 below)
- only GPv2 storage accounts allow you to serve static content like HTML, CSS, JavaScript, and images.
```
az storage account create \
--location <YOUR REGION> \
--name <YOUR STORAGE ACCOUNT NAME> \
--resource-group <YOUR GROUP NAME> \
--sku Standard_LRS \
--kind StorageV2
```

### 1.3 Add cli extension to stactic website hosting
As I am writing this article, static website hosting for GPv2 storage accounts is generally available, but we still need to install the storage preview extension to enable it via the CLI.
`az extension add --name storage-preview`

### 1.4 We need to enable static website hosting
```
az storage blob service-properties update \
--account-name <YOUR STORAGE ACCOUNT NAME> \
--static-website \
--index-document index.html
```

- You can do above thing via GUI, see this: https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blob-static-website
- You can then log ing your Azure portal to check if static website is enabled and have index doc name is set to index.html
- go to the storage a/c and grap your primary endpoint in static Website left bar item, in this case -> https://g3storage.z5.web.core.windows.net/


## 2. Setup your Azure function

### 2.1 create function app
```
az functionapp create \
--resource-group <YOUR GROUP NAME> \
--consumption-plan-location <YOUR REGION> \
--name <YOUR FUNCTION APP NAME> \
--storage-account <YOUR STORAGE ACCOUNT NAME> \
--functions-version 2 \
--runtime node
```
In this case
- YOUR GROUP NAME: g3
- YOUR REGION: westus2
- YOUR FUNCTION APP NAME: mike-serverless-func
- YOUR STORAGE ACCOUNT NAME: g3storage

### 2.2 update the CORS
then, we need to update the Cross-Origin Resource Sharing (CORS) configuration (what is CORS? see PS2)
We'll need to target the previous resource group and function app resources we created, and also define the allowed origin
```
az functionapp cors add \
--resource-group <YOUR GROUP NAME> \
--name <YOUR FUNCTION APP NAME> \
--allowed-origins <YOUR STATIC WEBSITE URL>
```
> don't put `/` at the end of your URL
You can also see this CORS setting in your Azure website Function App


### 2.3 create HTTP trigger Azure function
Go to Azure portal, classic function view page to create function
- choose HTTP trigger
- choose name and choose anynoumos
- you shall see code gen
- If you check the generated code:
  - a context object which, among other things, is used for communicating with the Azure Functions runtime.
  - The second argument (req) is the request object that contains the details about what was sent to the function. We'll be able to use the req object to access the data sent to this function from our web form.
  - If a name was provided with the request, we use the context.res object to send a response. In this case, we send back a "hello" message to the caller.
  - If a name wasn't provided, we'll send back an HTTP 400 error with a message explaining what the problem is.

- get your function URL, in this case: https://mike-serverless-func.azurewebsites.net/api/SayHello

## 3. Prep the frontend
- your index.html should will have below functionalities:
  - html part:
    - a form to input your name
  - script part:
    - a listenner to listen "submit event" and fetch Azure function
    - a function to handle response type and show sucess and error situation on the html

 ## 4. upload stactic files to Azure storage
 To upload the file/blob to the $web container, we simply need to upload the file/blob to the $web container
 ```
az storage blob upload-batch \
--source <YOUR SOURCE FOLDER> \
--destination \$web \
--account-name <YOUR STORAGE ACCOUNT NAME>
 ```

In this case:
```
az storage blob upload-batch \
--source ./src \
--destination \$web \
--account-name g3storage
```

## Done

See the page -> https://g3storage.z5.web.core.windows.net/


## PS1
### What is globally unique DNS-compliant name?
- names should not contain upper-case letters
- names should not contain underscores (_)
- names should not end with a dash
- names should be between 3 and 63 characters long
- names cannot contain dashes next to periods (e.g., my-.bucket.com and my.-bucket are invalid)
- names cannot contain periods
- Valid Examples:
  - my-eu-bucket-3
  - my-project-x
  - 4my-group

## PS2
### What is CORS
- CORS is a mechanism that tells a web browser, connected to one origin (domain), that it's OK to access resources on another source.
- What does different source mean? e.g. https vs http, different domain name, different port.
- You can think that if your server want to use other server API, it's not the in the same origin
- More info? -> https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
- Here, we host a static HTML page in the Azure Storage account. There will be JavaScript in that page that executes when users fill out and submit a form, and that client-side web brower code will be connecting to a different endpoint (the function URL) to communicate with the back-end of the application. CORS mechnism is to define where that traffic is allowed, in this case, the static website in Azure Storage, and it will tell web browsers used by our clients that its not a security risk to do this.
- 因為使用者提交時，你的函數會呼叫Azure Storage, 你需要設定讓使用者的瀏覽器知道這個溝通沒有安全風險。
