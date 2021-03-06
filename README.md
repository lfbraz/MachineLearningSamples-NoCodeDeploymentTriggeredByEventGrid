# Perform Azure Machine Learning No Code Deployment (NCD) in Azure Functions triggered by Azure Event Grid

This sample repo showcases an event-driven way to perform Azure Machine Learning no code deployment. It contains an Azure Function which is triggered by `Microsoft.MachineLearningServices.ModelRegistered` event published from Azure Event Grid.

## Prerequsites:
1. Have an Azure subscription
2. Register Azure Machine Learning and Azure Event Grid resource providers in your subscription if they are not registered yet. (**Note**: If Azure Machine Learning and Azure Event Grid were registered before Nov. 5, 2019, you will need to re-register them in your subscription. This is an one time thing.) 
3. Create an Azure Machine Learning workspace.
4. Create a service principal and grant it contributor access to your Azure Machine Learning workspace. You can do that by running following Azure CLI command (ou will need to have owner role of the Azure Machine Learning workspace):
```
az ad sp create-for-rbac -n "sp-for-nocodedeployment-azFunction" --role contributor --scopes /subscriptions/{SubID}/resourceGroups/{ResourceGroupName}/providers/Microsoft.MachineLearningServices/workspaces/{WorkspaceName}
```

Upon success, Azure CLI will return appId, password, tenant value of the newly provisioned service principal. Write down these values and you will need to use them to set the application settings of your Functions App. See the next section.

## Deploy the Azure Function

1. Install [Azure Functions Core Tools] (https://www.npmjs.com/package/azure-functions-core-tools) if you haven't.
2. go to AzFunction directory, replace <APP_NAME> with the name of your app and run following command:
```
func azure functionapp publish <APP_NAME> --build remote
```
3. go to Azure Portal, your Functions App, choose "Manage application settings".
![Manage application settings](./Images/AzFunc_appSetting1.png)

   Then set following application settings: 
    * TENANT_ID: the tenant ID of your service principal
    * SP_ID: the service principal ID
    * SP_PASSWORD: the service principal password
![Add application settings](./Images/AzFunc_appSetting2.png)

## Configure Event Grid Trigger

You can configure your Function App triggering based on Event Grid events, and set advanced filtering conditions based on model tags. 

1. Go to Azure portal, choose your functions app, and choose the python function just deployed, then choose "Add Event Grid subscription".
![Add Event Grid subscription](./Images/AzFunc_AddEventGridSubAdd.png)

2. Set the event subscription basic settings, including "Topic Type" (choose `Microsoft Machine Learning Services`), "Subscription", "Resource Group", "Resource" (your Azure Machine Learning workspace), and "Filter to Event Types" (choose `Model registered`).
![Event Grid subscription basic](./Images/AzFunc_AddEventGridSubBasic.png)

3. Go to the "Filters" tab, set "Advanced Filters" as below, which essentially means: `data.modelTags.ncd.contains("true") AND data.modelTages.stage.contains("production")`
![Event Grid subscription filters](./Images/AzFunc_AddEventGridSubFilters.png)

4. Click "Create".

## Register Model

Run following Azure Machine Learning CLI command. It registers a model with two tag values (`ncd=true, stage=production`) which match the Event Grid trigger advanced filters configured above.
```
az ml model register -n ncd-sklearn-model -p Model/sklearn_regression_model.pkl --model-framework ScikitLearn --cc 1 --gb 0.5 --tag ncd=true --tag stage=production -g <RESOURCE_GROUP_NAME> -w <WORKSPACE_NAME>
```

That will raise a `Microsoft.MachineLearningServices.ModelRegistered` event from your Azure Machine Learning workspace. The event is published via Azure Event Grid to your Functions App, which performs a no code deployment for the model you just registered.