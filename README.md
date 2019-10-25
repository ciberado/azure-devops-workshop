# Azure DevOps Pipelines full demo

## Env preparation

* Those are the variables that may contain sensitive information

```bash
WEBAPPNAME=<name of the application>
ORGANIZATION=<the name of the Azure DevOps organization>
GITHUB_USERNAME=<your github account name>
AZURE_DEVOPS_EXT_GITHUB_PAT=<something like 00005c201447166b8679f7e1a24f3af100000>
DEVOPS_TOKEN="<something like 00000ybzb4zh57xf5usmz3juvkukaqr55wqlimjcfq00000>
DOCKER_HUB_USERNAME=<your docker hub username>
DOCKER_HUB_PASSWORD=<your docker hub password>
```

## Create the application

* Install *dotnet 3* sdk following the instructions provided by [the official page](https://dotnet.microsoft.com/download/linux-package-manager/ubuntu18-04/sdk-3.0.100)

* For ubuntu:

```bash
wget \
  -q https://packages.microsoft.com/config/ubuntu/18.04/packages-microsoft-prod.deb \
  -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo add-apt-repository universe
sudo apt-get update
sudo apt-get install apt-transport-https -y
sudo apt-get update
sudo apt-get install dotnet-sdk-3.0 -y
```

* Set a nice name for your app

```bash
WEBAPPNAME=<name of the app>
```

* Lets create and test our wonderful application:

```bash
dotnet new webapp -o $WEBAPPNAME
cd $WEBAPPNAME
sed -i '9i<img src="https://i.imgur.com/zFBchOV.jpg" style="width:60%;margin:0 auto">' Pages/Index.cshtml 
dotnet run
```

* Now we can add a proper `Dockerfile`

```dockerfile
cat > Dockerfile <<EOF

FROM mcr.microsoft.com/dotnet/core/aspnet:3.0-alpine
WORKDIR /webapp
COPY /webapp /webapp
EXPOSE 5000/tcp
ENV ASPNETCORE_URLS http://*:5000
ENTRYPOINT dotnet ${WEBAPPNAME}.dll

EOF
```

## Github

* Install `git` and `hub`

```bash
sudo apt install git -y
sudo add-apt-repository ppa:cpick/hub
sudo apt-get update
sudo apt-get install hub -y
hub version
```

* Save your Github username in a variable

```bash
GITHUB_USERNAME=<your github username>
```

* Create a repo and publish it on *github*

```bash
wget https://raw.githubusercontent.com/dotnet/core/master/.gitignore
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa 
git init
git add -A
git commit -m "Yai"
hub create -d "Hello world here we come again" $GITHUB_USERNAME/$WEBAPPNAME 
git push origin master
```

* Check the project has been correctly created

* Generate a new [personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line) (click on your profile -> settings -> developer settings -> Personal access token) with *repo*, *admin:public_key* and * admin:repo_hook*

```bash
AZURE_DEVOPS_EXT_GITHUB_PAT=<your github personal access token>
```

## Azure DevOps 

* Visit [Azure devops](https://azure.microsoft.com/en-us/services/devops/) and create a new account or login using [Sign in to Azure DevOps](https://go.microsoft.com/fwlink/?LinkId=2014676&githubsi=true&clcid=0x409&WebUserId=22209D4DA9E4678D2E7C90B2A897661B)
* Follow the *create organization* wizard
* If you want, you can follow next wizard to create the project. But instead, we will do it using the CLI

* Go to your [personal profile](https://dev.azure.com/capsideworkshop/_usersSettings/about) (up right corner, shown after clicking on your user avatar)
* Select [Personal Access Token](https://dev.azure.com/capsideworkshop/_usersSettings/tokens) from the left
* Click on the *new token* button and generate a new one with full access
* **ENSURE YOU COPY THE GENERATED TOKEN**

```bash
DEVOPS_TOKEN=<your Azure devops token>
```

* Install the [Azure CLI tool](https://docs.microsoft.com/es-es/cli/azure/install-azure-cli-apt?view=azure-cli-latest)

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

* Login into Azure to ensure credentials have been persisted on disk

```bash
az login
```

* Add the *devops* extension

```bash
az extension add --name azure-devops
```

* Save your organization name in a variable

```bash
ORGANIZATION=<your organization name>
```

* Login into your *Azure DevOps* account

```bash
echo "$DEVOPS_TOKEN" | az devops login --org https://dev.azure.com/$ORGANIZATION
```

* Create the project!

```bash
az devops project create \
  --name $WEBAPPNAME \
  --description "My big demo" \
  --org https://dev.azure.com/$ORGANIZATION
```

* Generate the *connection* to the *github* repository

```bash
export AZURE_DEVOPS_EXT_GITHUB_PAT
az devops service-endpoint github create \
  --name ${WEBAPPNAME}github \
  --github-url https://github.com/$GITHUB_USERNAME/$WEBAPPNAME \
  --project $WEBAPPNAME \
  --org https://dev.azure.com/$ORGANIZATION
```

* Create variables with your *docker hub* credentials

```bash
DOCKER_HUB_USERNAME=<your docker hub username>
DOCKER_HUB_PASSWORD=<your docker hub password>
```

* Create the configuration for generating the connection to the *docker hub*

```json
cat > docker-hub-connection.json <<EOF
{
 "description": "Docker hub connection",
 "administratorsGroup": null,
 "authorization": {
   "parameters": {
     "username": "${DOCKER_HUB_USERNAME}",
     "password": "${DOCKER_HUB_PASSWORD}",
     "email": "Docker_ID_Email",
     "registry": "https://index.docker.io/v1/"
   },
   "scheme": "UsernamePassword"
 },
 "createdBy": null,
 "data": {
   "registrytype": "Others"
 },
 "name": "dockerhub",
 "type": "dockerregistry",
 "url": "https://index.docker.io/v1/",
 "readersGroup": null,
 "groupScopeId": null,
 "serviceEndpointProjectReferences": null,
 "operationStatus": null
}
EOF
```

* Now we can actually create the connection:

```bash
az devops service-endpoint create \
  --service-endpoint-configuration docker-hub-connection.json  \
  --org https://dev.azure.com/$ORGANIZATION  \
  --project $WEBAPPNAME
```

* The created connections can be seen on the [project settings page](https://dev.azure.com/capsideworkshop/helloworld/_settings/adminservices)


## Build pipeline

* Add the desired build stages in your project configuration

```yaml
cat <<EOF > azure-pipelines.yml
trigger:
- master

stages:
- stage: UnitTesting
  jobs:
  - job: DotNetUnitTesting
    pool:
      vmImage: 'ubuntu-latest'
    steps:
      - script: echo Run your very cool test suite here
  - job: JavascriptUnitTesting
    pool:
      vmImage: 'ubuntu-latest'
    steps:
      - script: echo The fun is in the js tests

- stage: Build
  jobs:
  - job: Build
    pool:
      vmImage: 'ubuntu-latest'
    variables:
      buildConfiguration: 'Release'
      imageName: '${DOCKER_HUB_USERNAME}/${WEBAPPNAME}'

    steps:
    - task: DotNetCoreInstaller@0
      displayName: 'Install .net core 3.0 (preview)'
      inputs:
        version: '3.0.100-preview9-014004'
    - script: dotnet publish -c release -o webapp
      displayName: 'dotnet build $(buildConfiguration)'
    - task: Docker@2
      inputs:
        containerRegistry: 'dockerhub'
        repository: '${DOCKER_HUB_USERNAME}/${WEBAPPNAME}'
        command: 'buildAndPush'
        Dockerfile: '**/Dockerfile'

- stage: Compliance
  jobs:
  - job: ExecuteSecurityCompliance
    pool:
      vmImage: 'ubuntu-latest'
    steps:
      - script: echo Now trigger that fancy security tool execution
EOF
```

* You know the song: *add*, *commit*, *push*

```bash
git add -A && git commit -m "Added pipeline"
git push origin master
```

* Get the github connection id

```bash
GITHUB_CONN_ID=$(az devops service-endpoint list --project $WEBAPPNAME --query "[?contains(name,'github')].id" --output tsv)
```

* Create the build pipeline!!!

```bash
az pipelines create \
  --project $WEBAPPNAME \
  --name ${WEBAPPNAME}-build \
  --description "Build pipeline for $WEBAPPNAME"  \
  --branch master \
  --repository-type github \
  --repository https://github.com/$GITHUB_USERNAME/$WEBAPPNAME \
  --service-connection $GITHUB_CONN_ID \
  --org https://dev.azure.com/$ORGANIZATION \
  --skip-first-run \
  --yaml-path azure-pipelines.yml
```

* Go to [Azure DevOps](https://dev.azure.com/capsideworkshop/helloworld/_build) and run your pipeline

* Check the image has been created in the [Docker hub](https://cloud.docker.com/repository/docker/ciberado/helloworld)

## Staging environment

* Set the name of the Azure *resource group*

```bash
RGNAME=workshop
```

* Create a new resource group

```bash
az group create \
  --location westeurope \
  --name $RGNAME
```

* There is no such a thing as free pizza: create the service plan

```bash
az appservice plan create \
  --name ${WEBAPPNAME}-staging-plan \
  --resource-group $RGNAME \
  --sku S1 \
  --is-linux
```

* Create the staging app service

```bash
az webapp create \
  --name ${WEBAPPNAME}-staging \
  --resource-group $RGNAME \
  --plan ${WEBAPPNAME}-staging-plan \
  --deployment-container-image-name nginx
```

* Get the webapp url and check it is running correctly

```bash
az webapp list \
  --query "[?contains(name, '$WEBAPP-staging')].defaultHostName" \
  --output tsv
```

* If the *app service* is not able to detect the port number of your app it is possible to specify it with the next command:

```bash
az webapp config appsettings set \
  --resource-group $RGNAME \
  --name ${WEBAPPNAME}-staging \
  --settings WEBSITES_PORT=5000
```

* Optionally, it is possible to enable access to the container's log with the following sentences:

```bash
az webapp log config \
  --name ${WEBAPPNAME}-staging \
  --resource-group $RGNAME \
  --docker-container-logging filesystem
  
az webapp log tail \
  --name ${WEBAPPNAME}-staging \
  --resource-group $RGNAME
```

## Staging release pipeline


* Go to *Releases* and create a new one by pressing the *create pipeline* button
* Select *Azure App Service deployment* as the pipeline template and apply it
* Set *publish to staging* as the name of the stage and click on the cross to close the dialog (yes, I know, I know)
* Click on *Add an artifact* and select the *build pipeline* from the dropdown menu
* On the *Default version* dropdown choose *Specify at the time of release creation* and leave *Source alias* as suggested. Then click on the *Add* button
* Press the *bolt* icon on the upper right corner of the *artifact* box and enable *Continous deployment trigger*. Then click on the cross to close this dialog
* Now configure the *publish to staging* box by clicking on it
* Choose your subscription and authorize the use of it from *Azure DevOps*
* Select *Web App for Containers (Linux)* as *App type*
* Click on your app in the *App service name* dropdown list
* In the *Registry or Namespace* type your Docker registry address or your Docker Hub username
* And type the name of your image in the *Repository* textfield
* Leave *Startup command* empty and press the *Save* button (the icon is on the top)
* Take a look at the the default task configuration (*Deploy Azure App Service*). It has the correct values so no need to update it
* Click on the *Pipeline* tab to see the whole pipeline again if you want
* Go to your build pipeline and trigger a manual run
* Watch it all


## Production environment

* It is not the best idea to share resources between *staging* and *production*, so lets create new *service plan*

```bash
az appservice plan create \
  --name ${WEBAPPNAME}-prod-plan \
  --resource-group $RGNAME \
  --sku S1 \
  --is-linux
```

* Create the prod app service

```bash
az webapp create \
  --name ${WEBAPPNAME}-prod \
  --resource-group $RGNAME \
  --plan ${WEBAPPNAME}-prod-plan \
  --deployment-container-image-name nginx
```

* To facilitate a quick manual rollback we are going to use a *slot*

```bash
az webapp deployment slot create \
  --name ${WEBAPPNAME}-prod \
  --resource-group $RGNAME \
  --slot secondary
```

* The app address can be retreived as before. Also, add `-secondary` to the subdomain to access the secondary slot

```bash
az webapp list \
  --query "[?contains(name, '$WEBAPP-prod')].defaultHostName" \
  --output tsv
```

## Publish to production stage

### Configure the first task

* Create a new stage by clicking on the *+Add* button that **appears under the *publish to staging* stage**
* Select (search for it, if needed) *Azure App Service deployment with slot* as the stage template and click on the *Apply* button
* Rename the stage to "publish to prod" and close the dialog
* On the *publish to prod* box click on its *bolt* icon to prevent automatic execution
* Enable the *Pre-deployment approvals* switch and type the name of the user that will be able to approve the promotion. Then, close the dialog
* Now configure the rest of the stage by clicking on the red admiration flag in the *publish to prod* stage
* Configure the *Deploy Azure App Service to Slot* just as before, but selecting the already existing subscription connectiong, using the `-prod` *App service name* and choosing the `secondary` *Slot*
* **Type `$(Build.BuildId)` as the content of the *Tag* textfield**
* Update the release pipeline by clicking on *Save*

### Add the manual intervention

* Click on the *...* button placed to the right of *publish to prod deployment process* label
* Select *Add an agentless job*
* On the new job, press the *+* button to add a task
* Choose *Manual intervention" and, if you are in the mood, add additional configuration to it

### Finish the stage with the swap task

* Again, click on *...* and *Add an agent job*
* Drag an drop *Manage Azure App Service - Slot Swap* to place it under this new job
* Check how this task is already configured

### Test the release pipeline

* Click on *Save*
* Select the *Pipeline* tab 
* Launch the *build pipeline again*
* Approve the execution of the *publish to prod* stage
* Check how the application has been deployed to the *secondary* slot, but the primary one still holds the *nginx* placeholder
* Approve the *swapp slot* task

## End to end demo

* From your app source folder, update the image shown by the page

```bash
sed -i 's/zFBchOV/RujhVhf/g' Pages/Index.cshtml
```

* Update the repo

```bash
git add -A && git commit -m "Updated logo" && git push origin master
```

