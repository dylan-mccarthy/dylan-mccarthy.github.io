---
layout: post
title: Setting up CICD pipelines for Azure Synapse Workspaces
date: 2023-09-20
categories: ["DevOps","CICD","Azure","Blog"]
---

# Setting up CICD pipelines for Azure Synapse Workspaces
Recently I have been working on a project to help deploy a data platform, this is not really something that I have done a lot of work on before and was excited to get stuck in and learn some new things.

The platform was build around the use of Azure Synapse Workspaces and I was tasked with setting up a CICD pipeline to deploy the workspaces. This was a bit of a challenge as there is not a lot of documentation around how to do this, so I thought I would write up a bit of a guide on how I went about doing this.

## What is Azure Synapse Workspaces?
Azure Synapse Workspaces is a managed service that allows you to create a workspace that can be used to develop and deploy data pipelines. It is a fully managed service that allows you to use a variety of languages to develop your pipelines, including SQL, Python, Scala and Spark.

It also allows you to use a variety of tools to develop your pipelines, including Azure Data Studio, Visual Studio Code and Synapse Studio.

## Integration with data sources
One of the key features of Azure Synapse Workspaces is the ability to integrate with a variety of data sources, including Azure Data Lake Storage, Azure SQL Database, Azure Cosmos DB, Azure Blob Storage and Azure Event Hubs.

To create these integrations you need to create a linked service, this is a connection to a data source that can be used in your pipelines. These linked services can be created manually or by using ARM/Bicep infrastructure as code.

## Deploying Azure Synapse Workspaces
For deployments we used Bicep to create the infrastructure as code, this allowed us to create the workspace and all the other Azure services that were required to support it. This included Storage Accounts, Key Vault and a Azure SQL Database.

The Bicep code was not too complex though in the context of this project there was some networking complexity that needed to be taken into account.

The deployment of the infrastructure was automated with Azure DevOps, this allowed us to create a pipeline that would deploy the infrastructure into the various environments. This was done by using a parameter file that would be used to define the environment that was being deployed to, as well as using variable groups to store values for each environment.

## Deploying Linked Services
Once the base infrastructure had been deployed the workspace was configured from the Synapse Studio UI. This allowed the data engineerse to create the linked services that they needed as well as define notebooks and pipelines. To limit the amount of hard coded values in the linked services we stored a lot of the details within the Key Vault and used the Key Vault linked service to access these values.

## Source Control
This is where the first challenge came in, when creating the Synapse Workspace for the development environment we wanted to ensure that the code being created was stored in source control. This was done by using the Git integration that is built into Synapse Studio.

Engineers work from withing either Synapse Studio or VS code and are able to commit their changes to the Git repo. This is a great feature and allows for a lot of flexibility in how you work with the code.

The challenge comes in with how do you publish changes into the development environment, and then how do you promote these changes into the other environments.

## Synapse Workspace CICD
Using Azure Devops the pipelines were created to first validate the Synapse code stored within the git repository, this was done using the Synapse workspace deployment task. This was installed from the Visual Studio marketplace here: https://marketplace.visualstudio.com/items?itemName=AzureSynapseWorkspace.synapsecicd-deploy

Using the yaml below you can see how this task is used to validate the Synapse workspace code.

```yaml
- task: SynapseWorkspaceDeployment@0
  displayName: 'Validate Synapse Workspace'
  inputs:
    operation: validate
    azureSubscription: 'AzureServiceConnection'
    ArtifactsFolder: '$(System.DefaultWorkingDirectory)/_SynapseWorkspace'
    ResourcegroupName: 'synapse-workspace-rg'
    TargetWorkspaceName: 'synapse-workspace'
```

This task will validate the code and if there are any errors it will fail the pipeline. This is a great way to ensure that the code is valid before deploying it into the environment.

The Artifact path is the path to the Synapse workspace code that is stored in the git repository.

Once the code has been validated it can then be deployed into the environment, this is done using the same task but with a different operation.

```yaml
- task: SynapseWorkspaceDeployment@0
  displayName: 'Deploy Synapse Workspace'
  inputs:
    operation: validateDeploy
    azureSubscription: 'AzureServiceConnection'
    ArtifactsFolder: '$(System.DefaultWorkingDirectory)/_SynapseWorkspace'
    ResourcegroupName: 'synapse-workspace-rg'
    TargetWorkspaceName: 'synapse-workspace'
```
In this code the operations is changed to validateDeploy, this will validate the code again and if valid it will deploy the code into the Synapse workspace. In this case you would use the Developement environment as the target.

## Promoting code to other environments
To enable a bit more control over when changes were pushed into the Test/UAT/Prod environments we used a Release branch in the git repository. This allows for changes to be batched up and then deployed into the other environments when required.

To do this we created a new pipeline that would be triggered when a commit was made to the Release branch. This pipeline would then validate the code and if valid deploy it into the Test environment.

```yaml
- task: SynapseWorkspaceDeployment@0
  displayName: 'Deploy Synapse Workspace'
  inputs:
    operation: validateDeploy
    azureSubscription: 'AzureServiceConnection'
    ArtifactsFolder: '$(System.DefaultWorkingDirectory)/_SynapseWorkspace'
    ResourcegroupName: 'synapse-workspace-rg'
    TargetWorkspaceName: 'synapse-workspace-test'
    OverrideArmParameters: |
        '-LS_Key_Vault_properties_typeProperties_url $(KeyVaultUrl)'
```

In this task you can see that the TargetWorkspaceName is set to the Test environment, this will deploy the code into the Test environment. You can also see that the KeyVaultUrl is being passed in as a parameter, this is a value that is stored in the variable group for the Test environment.

The override parameter in this example is referencing the Key Vault Linked service within the workspace that has the name 'LS_Key_Vault' and the property that we are overriding is the url property.

To see what properties are available to override you can look at the workspace_publish branch in the git repository, this is where the code is stored when you publish it.

In this branch look for the TemplateForWorkspace.json file and open it up, this will show you all the properties that can be overridden.

For further environments you can use the above task and change the TargetWorkspaceName to the relevant environment. You can also override any other properties that you need to, for the example above we used variable groups from Azure DevOps to store the values for each environment.

## Difficulties and Challenges
There were a few challenges that we faced while trying to configured the Synapse Workspaces that I thought it would be good to share.

### Spark Pool storage account access
When accessing storage accounts from the spark pool we found that we were having trouble with access, we had initially been using a managed identity to managed this access but we were getting errors saying this was not supported. We had some sucess when adding the Synapse System Assigned Identity to the storage account when connecting over the DFS endpoint but access to blob storage was still not working.

In the end we had to use a SAS token to access the storage account, this was not ideal as it meant that we had to store the SAS token in the Key Vault and then pass it into the linked service. This was not ideal as it meant that someone had to keep track of the token expirey and update it when it expired. This was somewhat mitigated by creating an automation pipeline for updating the values in the Key Vault.

### Private Networking
Another challenge was around networking and trying to access resources over private endpoints. We had intended to use the Integration Runtimes to do this as when looking at the bicep definitions would could bind these runtimes to virtual networks and subnets. Unfortunatly this turned out not to be the case as this feature is not something that is currently supported.

So in the end we used Managed Private Endpoints which is a feature of the Synapse Workspace, this allows for direct connections to be made to the data sources over private endpoints. Once this was set up however it worked well and we were able to access the data sources without any issues.

## Conclusion
Overall I found the Synapse Workspaces to be a great tool for developing and deploying data pipelines, it is a very flexible tool that allows you to use a variety of languages and tools to develop your pipelines. It also has a lot of great features that allow you to integrate with a variety of data sources.

The challenges that we faced were not insurmountable and we were able to find solutions to them, though it would be great to see some of these features added in the future.

I hope that this post has been useful and that it helps you to set up your own CICD pipelines for Azure Synapse Workspaces. 

As this project was focused on using Azure DevOps I have not covered how to do this using GitHub, if that is something you would like to see please let me know on LinkedIn and I'll put up some examples.