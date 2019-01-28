# Deploy your bot to Azure from Source Control
This article will walk you through the deployment process to get your bot deployed from a git repository to Azure Bot Service.

## 1. Create a bot
Follow [this doc](https://docs.microsoft.com/en-us/azure/bot-service/javascript/bot-builder-javascript-quickstart?view=azure-bot-service-4.0) to create simple js bot using yeoman generator.

## 2. Azure - Create a new Web App Bot
1. Navigate to [https://portal.azure.com](https://portal.azure.com)
1. Click on the green `+ Create a resource` sign
1. Navigate to `AI + Machine Learning` category and create a new `Web App Bot`.
1. Fill with all the appropriate settings like the location, template langugage, etc.

## 3. Update `.bot` file
Make some minor changes to your `.bot` file before publishing to Azure.
1. Add a production endpoint, either by using the below snippet or using the CLI:
    1. Snippet
        ````javascript
            {
                "name": "jsbot",
                "description": "",
                "services": [
                    {
                        "type": "endpoint",
                        "name": "development",
                        "endpoint": "http://localhost:3978/api/messages",
                        "appId": "",
                        "appPassword": "",
                        "id": "1"
                    },
                    {
                        "type": "endpoint",
                        "name": "production",
                        "endpoint": "https://<something>.azurewebsites.net/api/messages",
                        "appId": "<App Id>",
                        "appPassword": "<App Password>",
                        "id": "2"
                    }
                ],
                "padlock": "Y3J03Lk3Ykg6/4jY1GVd4Q==!CsNWpsgt+DBSzswXVyJe5IrlpyF79oHppZnuhX0pOwyKahr059qfpMz2UdyeaYSj",
                "version": "2.0"
            }
        ````
    
    1. MSBot CLI command - follow [the instructions on the MSBot doc](https://github.com/Microsoft/botbuilder-tools/blob/master/packages/MSBot/docs/add-services.md#connecting-to-a-endpoint-service). For example: 
    ````cmmd
    msbot connect endpoint --name "Debug TestBot" --appId <APP-ID> --appPassword 1abHDN3421342 --endpoint http://localhost:9090/api/messages
    ````
2. Encrypt your `.bot` file
Encrypt your `.bot` file using `msbot secret --new`. Copy the secret and save it for the next step.

    > <i>Learn more about encrypting `.bot` files [here](https://github.com/Microsoft/botbuilder-tools/blob/master/packages/MSBot/docs/bot-file-encryption.md).</i>

## 4. Setup a git repository
Create a git repository using your favourite git source control provider. Make sure to encrypt your `.bot` file before commiting your code into a repository.

## 5. Azure App settings
1. Navigate to the new Web App Bot in [Step 2](#2. -zure---Create-a-new-Web-App-Bot) that you created earlier.
1. Click on `All App Service Settings` as marked by the red box in the below image:

    ![Azure Blade Navbar](https://github.com/ryanvolum/menu-bot/blob/master/wiki_assets/AzureBladeNavbar.png)

1. Click on `App Settings` and update the `botFilePath` (if the path is different on your repo) and `botFileSecret` (from [Step 3](#3.-Update-`.bot`-file)) values as marked by the red box below. 

    ![Azure App Settings](https://github.com/ryanvolum/menu-bot/blob/master/wiki_assets/AzureAppSettings.png)

## 6. Deploy using Azure Deployment Center
Now, you need to connect your git repository with Azure App Services. Follow [this doc](https://docs.microsoft.com/en-us/azure/app-service/deploy-continuous-deployment) to setup Continuous Deployment for your bot. Note that it is recommended to build using `App Service Kudu build server`.

## 7. Test your deployment
Wait for a few seconds after a succesful deployment and optionally restart your Web App to clear any cache. Go back to your `Web App Bot` blade and test using the Web Chat provided in the Azure portal or using the emulator on your machine.