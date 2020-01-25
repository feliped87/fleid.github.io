---
layout: post
title:  "Retrieving Azure KeyVault secrets with PowerShell in Azure DevOps Pipelines"
date:   2020-01-07 10:00:00 -0700
categories: ALM Azure ASA DevOps
---

# Retrieving Azure Key Vault secrets with PowerShell in Azure DevOps pipelines

## Context

I was [recently challenged](https://www.eiden.ca/asa-alm-104/) while trying to retrieve secrets stored in [Azure Key Vault](https://azure.microsoft.com/en-us/services/key-vault/) from a [PowerShell](https://docs.microsoft.com/en-us/powershell/scripting/overview?view=powershell-7) script running in [Azure DevOps Pipelines](https://azure.microsoft.com/en-us/services/devops/pipelines/).

At the time I was [building a CI/CD pipeline](https://www.eiden.ca/asa-alm-100/) for an [Azure Stream Analytics](https://docs.microsoft.com/en-us/azure/stream-analytics/stream-analytics-introduction) job in Azure DevOps. For that I needed to perform some **ARM Template deployments** via a PowerShell task. Figuring out the syntax to get access to my secrets in the script was not as easy as I expected.

![Schema focusing on the release pipeline](https://github.com/Fleid/fleid.github.io/blob/master/_posts/201912_asa_alm101/asa_alm104_goal.png?raw=true)

*[figure 1 - Schema of the release pipeline](https://github.com/Fleid/fleid.github.io/blob/master/_posts/201912_asa_alm101/asa_alm104_goal.png?raw=true)*

What should be a straightforward scenario takes a bit of planning. The main issue being that Azure Pipelines offers different capabilities, which come with different syntaxes, depending on 2 factors:

- [the experience](https://docs.microsoft.com/en-us/azure/devops/pipelines/get-started/pipelines-get-started?view=azure-devops&tabs=yaml) of the pipeline:  YAML vs Classic
- the script type of the [Azure PowerShell](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/azure-powershell?view=azure-devops) task: inline vs file script

## TL/DR

Here are the wirings that work, see below for details on each syntax:

![Schema of the available options](https://github.com/Fleid/fleid.github.io/blob/master/_posts/202001_azure_devops_keyvault/recap.png?raw=true)

*[figure 2 - Schema of the available options](https://github.com/Fleid/fleid.github.io/blob/master/_posts/202001_azure_devops_keyvault/recap.png?raw=true)*

- **YAML** experience / **Inline** script
  - Input macro
  - Mapped environment variable (recommended in the doc)
  - PowerShell Get-AzKeyVaultSecret

- **YAML** experience / **File Path** script
  - Argument / Parameter mapping
  - Mapped environment variable  (recommended in the doc)
  - PowerShell Get-AzKeyVaultSecret

- **Classic** experience / **Inline** script
  - Input macro
  - PowerShell Get-AzKeyVaultSecret

- **Classic** experience / **File Path** script
  - Argument / Parameter mapping
  - PowerShell Get-AzKeyVaultSecret

## Options

**Before trying anything else**, it's required to create a variable group linked to the Key Vault (see [middle section of that article](https://www.eiden.ca/asa-alm-104/) if necessary).

To be noted:

> When trying to link the KeyVault in the Variable Group, the **authentication** process can hang indefinitely. It can be solved in KeyVault, by manually creating an **access policy** for the Azure DevOps project application principal (service account) with List/Get permissions on Secrets. The application principal id can be found in the Azure DevOps project **settings** (bottom left), **Service Connections** tab, editing the right subscription and going `use the full version of the service connection dialog`. It should be under `Service principal client ID`.

### Input macro

**Only available inline**, [more info](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#set-variables-in-pipeline).

With the variable group `myVariableGroup` linked to KeyVault, giving access to the secret `kvTestSecret`.

#### Input macro : YAML experience

The inline script can reference the secret directly via : `$(kvTestSecret)`.

```YAML
trigger:
  - master
  
pool:
  vmImage: 'windows-latest'

variables:
- group: myVariableGroup
  
steps:

- task: AzurePowerShell@4
  displayName: 'Azure PowerShell script - inline'
  inputs:
    azureSubscription: '...'
    ScriptType: 'InlineScript'
    Inline: |
      # Using an input-macro:
      Write-Host "Input-macro from KeyVault VG: $(kvTestSecret)"
```

#### Input macro : Classic experience

In the classic experience, the variable group must be declared in the `Variables` tab beforehand.
Then the inline script can reference the secret directly via : `$(kvTestSecret)` (as in `Write-Host "Input-macro from KeyVault VG: $(kvTestSecret)"`).

![Screenshot of Azure DevOps : Input macro syntax for inline script in classic experience](https://github.com/Fleid/fleid.github.io/blob/master/_posts/202001_azure_devops_keyvault/macro_inline_classic.png?raw=true)

*[figure 3 - Screenshot of Azure DevOps : Input macro syntax for inline script in classic experience](https://github.com/Fleid/fleid.github.io/blob/master/_posts/202001_azure_devops_keyvault/macro_inline_classic.png?raw=true)*

### Inherited environment variable

This syntax is the default for variables **not** coming from Key Vault (local variable and default variable groups). It will **not** return Key Vault secrets in any configuration.

With the variable group `myDefaultVariableGroup` **not** linked to KeyVault, holding the variable `normalVariable`. Also with the variable `localVariable`.

#### Inherited Env : YAML experience

For Inline and File script the syntax is similar : `$env:normalVariable` (as in `Write-Host "Inherited ENV from normal VG: $env:normalVariable"`)

```YAML

trigger:
  - master
  
pool:
  vmImage: 'windows-latest'

variables:
- group: myDefaultVariableGroup
- name: localVariable
  value: myvalue
  
steps:

- task: AzurePowerShell@4
  displayName: 'Azure PowerShell script - inline'
  inputs:
    azureSubscription: '...'
    ScriptType: 'InlineScript'
    Inline: |
      # Using the env var:
      Write-Host "Inherited ENV from normal VG: $env:normalVariable"
      Write-Host "Inherited ENV from local variable: $env:localVariable"
    azurePowerShellVersion: 'LatestVersion'

- task: AzurePowerShell@4
  displayName: 'Azure PowerShell script - file path'
  inputs:
    azureSubscription: '...'
    ScriptType: 'FilePath'
    ScriptPath: '$(Build.Repository.LocalPath)/myScript.ps1'
    azurePowerShellVersion: 'LatestVersion'
```

#### Inherited Env : Classic experience

In the classic experience, both the local variable and the variable group must be declared in the `Variables` tab beforehand.
Then for Inline and File script the syntax is similar : `$env:normalVariable` (as in `Write-Host "Inherited ENV from normal VG: $env:normalVariable"`).

NB : The variable name will be altered as follow ([ref](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#understand-variable-syntax)):

> Name is upper-cased, . replaced with _

![Screenshot of Azure DevOps : Inherited environment variable for inline script in classic experience](https://github.com/Fleid/fleid.github.io/blob/master/_posts/202001_azure_devops_keyvault/inherited_inline_classic.png?raw=true)

*[figure 2 - Screenshot of Azure DevOps : Inherited environment variable for inline script in classic experience](https://github.com/Fleid/fleid.github.io/blob/master/_posts/202001_azure_devops_keyvault/inherited_inline_classic.png?raw=true)*

### Mapped environment variable

**Only available via the YAML experience.** This is the [recommended solution](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#secret-variables) in YAML.

With the variable group `myVariableGroup` linked to KeyVault, giving access to the secret `kvTestSecret`.
The inline script can reference the secret directly via : `$env:MY_MAPPED_ENV_VAR_KV` (as in `Write-Host "Mapped ENV from KeyVault VG: $env:MY_MAPPED_ENV_VAR_KV"`).

The key statement here being the `env:` [parameter](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema?view=azure-devops&tabs=schema#task) required at each task.

```YAML
trigger:
  - master
  
pool:
  vmImage: 'windows-latest'

variables:
- group: myVariableGroup
  
steps:

- task: AzurePowerShell@4
  env: 
    MY_MAPPED_ENV_VAR_KV: $(kvTestSecret)
  displayName: 'Azure PowerShell script - inline'
  inputs:
    azureSubscription: '...'
    ScriptType: 'InlineScript'
    Inline: |
      # Using the env var:
      Write-Host "Mapped ENV from KeyVault VG: $env:MY_MAPPED_ENV_VAR_KV"
    azurePowerShellVersion: 'LatestVersion'

- task: AzurePowerShell@4
  env: 
    MY_MAPPED_ENV_VAR_KV: $(kvTestSecret)
  displayName: 'Azure PowerShell script - file path'
  inputs:
    azureSubscription: '...'
    ScriptType: 'FilePath'
    ScriptPath: '$(Build.Repository.LocalPath)/myScript.ps1'
    azurePowerShellVersion: 'LatestVersion'
```

### Argument / Parameter mapping

**Only available in File Path experience**.

#### Argument : YAML experience

#### Argument : Classic experience

### PowerShell Get-AzKeyVaultSecret

## Resources

Azure Pipelines >

- [Variable Groups](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/variable-groups?view=azure-devops&tabs=yaml)
- [Variable > Syntax](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#understand-variable-syntax)
- [Variable > Set Secret Variables](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/variables?view=azure-devops&tabs=yaml%2Cbatch#secret-variables)
- [Azure PowerShell Task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/azure-powershell?view=azure-devops#samples)