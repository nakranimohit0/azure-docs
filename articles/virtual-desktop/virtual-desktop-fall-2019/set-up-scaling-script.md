---
title: Scale session hosts Azure Automation - Azure
description: How to automatically scale Windows Virtual Desktop session hosts with Azure Automation.
services: virtual-desktop
author: Heidilohr

ms.service: virtual-desktop
ms.topic: conceptual
ms.date: 03/30/2020
ms.author: helohr
manager: lizross
---
# Scale session hosts using Azure Automation

>[!IMPORTANT]
>This content applies to the Fall 2019 release that doesn't support Azure Resource Manager Windows Virtual Desktop objects.

You can reduce your total Windows Virtual Desktop deployment cost by scaling your virtual machines (VMs). This means shutting down and deallocating session host VMs during off-peak usage hours, then turning them back on and reallocating them during peak hours.

In this article, you'll learn about the scaling tool built with Azure Automation Account and Azure Logic App that will automatically scale session host VMs in your Windows Virtual Desktop environment. To learn how to use the scaling tool, skip ahead to [Prerequisites](#Prerequisites).

## How the scaling tool works

The scaling tool provides a low-cost automation option for customers who want to optimize their session host VM costs.

You can use the scaling tool to:
 
- Schedule VMs to start and stop based on Peak and Off-Peak business hours.
- Scale out VMs based on number of sessions per CPU core.
- Scale in VMs during Off-Peak hours, leaving the minimum number of session host VMs running.

The scaling tool uses a combination of Azure Automation Account, PowerShell runbook, webhook, and Azure Logic App to function. When the tool runs, Azure Logic App calls a webhook to start the Azure Automation runbook. The runbook then creates a job.

During peak usage time, the job checks the current number of sessions and the VM capacity of the current running session host for each host pool. It uses this information to calculate if the running session host VMs can support existing sessions based on the *SessionThresholdPerCPU* parameter defined for the **CreateOrUpdateAzLogicApp.ps1** file. If the session host VMs can't support existing sessions, the job starts additional session host VMs in the host pool.

>[!NOTE]
>*SessionThresholdPerCPU* doesn't restrict the number of sessions on the VM. This parameter only determines when new VMs need to be started to load-balance the connections. To restrict the number of sessions, you need to follow the instructions [Set-RdsHostPool](/powershell/module/windowsvirtualdesktop/set-rdshostpool/) to configure the *MaxSessionLimit* parameter accordingly.

During the off-peak usage time, the job determines how many session host VMs should be shut down based on the *MinimumNumberOfRDSH* parameter. If you set the *LimitSecondsToForceLogOffUser* parameter to a non-zero positive value, the job will set the session host VMs to drain mode to prevent new sessions from connecting to the hosts. It will then notify any currently signed in users to save their work, wait the configured amount of time, and then force the users to sign out. Once all user sessions on the session host VM have been signed out, the job will shut down the VM. After the VM is shut down, the job will reset its session host drain mode.

>[!NOTE]
>If you manually set the session host VM to drain mode, the session host VM will not be managed by the job. If the session host VM is running and set to drain mode, it will be treated as unavailable and so the job may start additional VMs to handle the load.

If you set the *LimitSecondsToForceLogOffUser* parameter to zero, the job will allow the session configuration setting in specified group policies to handle signing off user sessions. To see these group policies, go to **Computer Configuration** > **Policies** > **Administrative Templates** > **Windows Components** > **Remotes Desktop Services** > **Remote Desktop Session Host** > **Session Time Limits GPOs**. If there are any active sessions on a session host VM, the job will leave the session host VM running. If there are no active sessions i.e only after all sessions are logged off, the job will shut down the session host VM.

During any time, the job also takes host pool's *MaxSessionLimit* into account to determine if the current number of sessions is more than 90% of the maximum capacity. If it is, the job will start additional session host VMs.

The job runs periodically based on a set recurrence interval. You can change this interval based on the size of your Windows Virtual Desktop environment, but remember that starting and shutting down VMs can take some time, so remember to account for the delay. We recommend setting the recurrence interval to every 15 minutes.

However, the tool also has the following limitations:

- This solution applies only to pooled multi-session session host VMs.
- This solution manages VMs in any region, but can only be used in the same subscription as your Azure Automation Account and Azure Logic App.
- Max runtime of a job in the runbook is 3 hours. In case starting/stopping of VMs in a host pool takes longer than that, the job would fail. [More details](../../automation/automation-runbook-execution.md#fair-share)

>[!NOTE]
>The scaling tool controls the load balancing mode of the host pool it is scaling. It sets it to breadth-first load balancing for both peak and off-peak hours.

## Prerequisites

Before you start setting up the scaling tool, make sure you have the following things ready:

- A [Windows Virtual Desktop tenant and host pool](create-host-pools-arm-template.md)
- Session host pool VMs configured and registered with the Windows Virtual Desktop service
- A user with [Contributor access](../../role-based-access-control/role-assignments-portal.md) on Azure subscription

The machine you use to deploy the tool must have: 

- Windows PowerShell 5.1 or later
- The Microsoft Az PowerShell module

If you have everything ready, then let's get started.

## Create or update an Azure Automation Account

>[!NOTE]
>If you already have an existing Azure Automation Account with a runbook running an older version of the script, you can re-run just this step to update it to latest.

First, you'll need an Azure Automation Account to run the PowerShell runbook. Below procedure is valid even if you have an existing Azure Automation Account which you would like to use to setup the PowerShell runbook. Here's how to set it up:

1. Open Windows PowerShell.

2. Run the following cmdlet to sign in to your Azure account.

     ```powershell
     Login-AzAccount
     ```

     >[!NOTE]
     >Your account must have contributor rights on the Azure subscription that you want to deploy the scaling tool on.

3. Run the following cmdlet to download the script for creating the Azure Automation Account:

     ```powershell
     New-Item -ItemType Directory -Path "C:\Temp" -Force
     Set-Location -Path "C:\Temp"
     $uri = "https://raw.githubusercontent.com/Azure/RDS-Templates/wvd_scaling/wvd-templates/wvd-scaling-script/CreateOrUpdateAzAutoAccount.ps1"
     # Download the script
     Invoke-WebRequest -Uri $uri -OutFile ".\CreateOrUpdateAzAutoAccount.ps1"
     ```

4. Run the following cmdlet to execute the script and create the Azure Automation Account:

     ```powershell
     $params = @{
          "AADTenantId"           = "<Azure_Active_Directory_tenant_ID>"   # Optional. If not specified, it will use the current Azure context
          "SubscriptionId"        = "<Azure_subscription_ID>"              # Optional. If not specified, it will use the current Azure context
          "ResourceGroupName"     = "<Resource_group_name>"                # Optional. Default: "WVDAutoScaleResourceGroup"
          "AutomationAccountName" = "<Automation_account_name>"            # Optional. Default: "WVDAutoScaleAutomationAccount"
          "Location"              = "<Azure_region_for_deployment>"        # Optional. Default: "West US2"
          "WorkspaceName"         = "<Log_analytics_workspace_name>"       # Optional. If not specified, log analytics workspace will not be used
     }

     .\CreateOrUpdateAzAutoAccount.ps1 @params
     ```

5. The cmdlet's output will include a webhook URI. Make sure to keep a record of the URI because you'll use it as a parameter when you set up the execution schedule for the Azure Logic App.

6. After you've set up your Azure Automation Account, sign in to your Azure subscription and check to make sure your Azure Automation Account and the relevant runbook have appeared in your specified resource group, as shown in the following image:

![An image of the Azure overview page showing the newly created Azure Automation Account and runbook.](media/automation-account.png)

  To check if your webhook is where it should be, select the name of your runbook. Next, go to your runbook's Resources section and select **Webhooks**.

## Create an Azure Automation Run As Account

Now that you have an Azure Automation Account, you'll also need to create an Azure Automation Run As Account if you don't have one for the tool to access your Azure resources.

An [Azure Automation Run As Account](../../automation/manage-runas-account.md) provides authentication for managing resources in Azure with the Azure cmdlets. When you create a Run As Account, it creates a new service principal user in Azure Active Directory and assigns the Contributor role to the service principal user at the subscription level, the Azure Run As Account is a great way to authenticate securely with certificates and a service principal name without needing to store a username and password in a credential object. To learn more about Run As Account authentication, see [Limit Run As Account permissions](../../automation/manage-runas-account.md#limit-run-as-account-permissions).

Any user who's a member of the Subscription Admins role and coadministrator of the subscription can create a Run As Account by following the next section's instructions.

To create a Run As Account in your Azure Automation Account:

1. In the Azure portal, select **All services**. In the list of resources, enter and select **Automation Accounts**.

2. On the **Automation Accounts** page, select the name of your Azure Automation Account.

3. In the pane on the left side of the window, select **Run As Accounts** under the **Account Settings** section.

4. Select **Azure Run As Account**. When the **Add Azure Run As Account** pane appears, review the overview information, and then select **Create** to start the account creation process.

5. Wait a few minutes for Azure to create the Run As Account. You can track the creation progress in the menu under Notifications.

6. When the process finishes, it will create an asset named **AzureRunAsConnection** in the specified Azure Automation Account. The connection asset holds the application ID, tenant ID, subscription ID, and certificate thumbprint. Remember the application ID, because you'll use it later.

### Create a role assignment in Windows Virtual Desktop

Next, you need to create a role assignment so that **AzureRunAsConnection** can interact with Windows Virtual Desktop. Make sure to use PowerShell to sign in with an account that has permissions to create role assignments.

First, download and import the [Windows Virtual Desktop PowerShell module](/powershell/windows-virtual-desktop/overview/) to use in your PowerShell session if you haven't already. Run the following PowerShell cmdlets to connect to Windows Virtual Desktop and display your tenants.

```powershell
Add-RdsAccount -DeploymentUrl "https://rdbroker.wvd.microsoft.com"

Get-RdsTenant
```

When you find the tenant with the host pools you want to scale, follow the instructions in [Create an Azure Automation Run As Account](#Create-an-Azure-Automation-Run-As-Account) to gather the **AzureRunAsConnection** ApplicationID and use the WVD tenant name you got from the previous cmdlet in the following cmdlet to create the role assignment:

```powershell
New-RdsRoleAssignment -RoleDefinitionName "RDS Contributor" -ApplicationId <applicationid> -TenantName <tenantname>
```

## Create the Azure Logic App and execution schedule

Finally, you'll need to create the Azure Logic App and set up an execution schedule for your new scaling tool.

1.  Open Windows PowerShell.

2.  Run the following cmdlet to sign in to your Azure account.

     ```powershell
     Login-AzAccount
     ```

3. Run the following cmdlet to download the script for creating the Azure Logic App.

     ```powershell
     New-Item -ItemType Directory -Path "C:\Temp" -Force
     Set-Location -Path "C:\Temp"
     $uri = "https://raw.githubusercontent.com/Azure/RDS-Templates/wvd_scaling/wvd-templates/wvd-scaling-script/CreateOrUpdateAzLogicApp.ps1"
     # Download the script
     Invoke-WebRequest -Uri $uri -OutFile ".\CreateOrUpdateAzLogicApp.ps1"
     ```

4. Run the following cmdlet to sign into Windows Virtual Desktop with an account that has RDS Owner or RDS Contributor permissions.

     ```powershell
     Add-RdsAccount -DeploymentUrl "https://rdbroker.wvd.microsoft.com"
     ```

5. Run the following PowerShell script to create the Azure Logic App and execution schedule for your host pool (Note: You will need to run this for each of the host pools you want to auto scale, but you need only 1 Azure Automation Account).

     ```powershell
     $aadTenantId = (Get-AzContext).Tenant.Id
     
     $azureSubscription = Get-AzSubscription | Out-GridView -PassThru -Title "Select your Azure Subscription"
     Select-AzSubscription -Subscription $azureSubscription.Id
     $subscriptionId = $azureSubscription.Id
     
     $resourceGroup = Get-AzResourceGroup | Out-GridView -PassThru -Title "Select the resource group for the new Azure Logic App"
     $resourceGroupName = $resourceGroup.ResourceGroupName
     $location = $resourceGroup.Location
     
     $wvdTenant = Get-RdsTenant | Out-GridView -PassThru -Title "Select your WVD tenant"
     $tenantName = $wvdTenant.TenantName
     $tenantGroupName = $wvdTenant.TenantGroupName
     
     $wvdHostpool = Get-RdsHostPool -TenantName $wvdTenant.TenantName | Out-GridView -PassThru -Title "Select the host pool you'd like to scale"
     $hostPoolName = $wvdHostpool.HostPoolName
     
     $recurrenceInterval = Read-Host -Prompt "Enter how often you'd like the job to run in minutes, e.g. '15'"
     $beginPeakTime = Read-Host -Prompt "Enter the start time for peak hours in local time, e.g. 9:00"
     $endPeakTime = Read-Host -Prompt "Enter the end time for peak hours in local time, e.g. 18:00"
     $timeDifference = Read-Host -Prompt "Enter the time difference between local time and UTC in hours, e.g. +5:30"
     $sessionThresholdPerCPU = Read-Host -Prompt "Enter the maximum number of sessions per CPU that will be used as a threshold to determine when new session host VMs need to be started during peak hours"
     $minimumNumberOfRdsh = Read-Host -Prompt "Enter the minimum number of session host VMs to keep running during off-peak hours"
     $limitSecondsToForceLogOffUser = Read-Host -Prompt "Enter the number of seconds to wait before automatically signing out users. If set to 0, any session host VM that has user sessions, will be left untouched"
     $logOffMessageTitle = Read-Host -Prompt "Enter the title of the message sent to the user before they are forced to sign out"
     $logOffMessageBody = Read-Host -Prompt "Enter the body of the message sent to the user before they are forced to sign out"
     
     $automationAccount = Get-AzAutomationAccount -ResourceGroupName $resourceGroup.ResourceGroupName | Out-GridView -PassThru -Title "Select the Azure Automation Account"
     $automationAccountName = $automationAccount.AutomationAccountName
     $automationAccountConnection = Get-AzAutomationConnection -ResourceGroupName $resourceGroup.ResourceGroupName -AutomationAccountName $automationAccount.AutomationAccountName | Out-GridView -PassThru -Title "Select the Azure RunAs connection asset"
     $connectionAssetName = $automationAccountConnection.Name
     
     $webhookURI = Read-Host -Prompt "Enter the URI of the webhook returned by when you created the Azure Automation Account"
     $maintenanceTagName = Read-Host -Prompt "Enter the name of the Tag associated with VMs you don't want to be managed by this scaling tool"

     $params = @{
          "AADTenantId"                   = $aadTenantId                             # Optional. If not specified, it will use the current Azure context
          "SubscriptionID"                = $subscriptionId                          # Optional. If not specified, it will use the current Azure context
          "ResourceGroupName"             = $resourceGroupName                       # Optional. Default: "WVDAutoScaleResourceGroup"
          "Location"                      = $location                                # Optional. Default: "West US2"
          "RDBrokerURL"                   = "https://rdbroker.wvd.microsoft.com"     # Optional. Default: "https://rdbroker.wvd.microsoft.com"
          "TenantGroupName"               = $tenantGroupName                         # Optional. Default: "Default Tenant Group"
          "TenantName"                    = $tenantName
          "HostPoolName"                  = $hostPoolName
          "LogAnalyticsWorkspaceId"       = "<Log_analytics_workspace_ID>"           # Optional. If not specified, script will not log to the log analytics workspace
          "LogAnalyticsPrimaryKey"        = "<Log_analytics_primary_key>"            # Optional. If not specified, script will not log to the log analytics workspace
          "ConnectionAssetName"           = $connectionAssetName                     # Optional. Default: "AzureRunAsConnection"
          "RecurrenceInterval"            = $recurrenceInterval                      # Optional. Default: 15
          "BeginPeakTime"                 = $beginPeakTime                           # Optional. Default: "09:00"
          "EndPeakTime"                   = $endPeakTime                             # Optional. Default: "17:00"
          "TimeDifference"                = $timeDifference                          # Optional. Default: "-7:00"
          "SessionThresholdPerCPU"        = $sessionThresholdPerCPU                  # Optional. Default: 1
          "MinimumNumberOfRDSH"           = $minimumNumberOfRdsh                     # Optional. Default: 1
          "MaintenanceTagName"            = $maintenanceTagName                      # Optional.
          "LimitSecondsToForceLogOffUser" = $limitSecondsToForceLogOffUser           # Optional. Default: 1
          "LogOffMessageTitle"            = $logOffMessageTitle                      # Optional. Default: "Machine is about to shutdown."
          "LogOffMessageBody"             = $logOffMessageBody                       # Optional. Default: "Your session will be logged off. Please save and close everything."
          "WebhookURI"                    = $webhookURI
     }

     .\CreateOrUpdateAzLogicApp.ps1 @params
     ```

     After you run the script, the Azure Logic App should appear in a resource group, as shown in the following image.

     ![An image of the overview page for an example Azure Logic App.](../media/logic-app.png)

To make changes to the execution schedule, such as changing the recurrence interval or time zone, go to the Azure Logic App autoscale scheduler and select **Edit** to go to the Azure Logic App Designer.

![An image of the Azure Logic App Designer. The Recurrence and webhook menus that let the user edit recurrence times and the webhook file are open.](../media/logic-apps-designer.png)

## Manage your scaling tool

Now that you've created your scaling tool, you can access its output. This section describes a few features you might find helpful.

### View job status

You can view a summarized status of all runbook jobs or view a more in-depth status of a specific runbook job in the Azure portal.

On the right of your selected Azure Automation Account, under "Job Statistics," you can view a list of summaries of all runbook jobs. Opening the **Jobs** page on the left side of the window shows current job statuses, start times, and completion times.

![A screenshot of the job status page.](media/jobs-status.png)

### View logs and scaling tool output

You can view the logs of scale-out and scale-in operations by opening your runbook and selecting the job.

Navigate to the runbook in your resource group hosting the Azure Automation Account and select **Overview**. On the overview page, select a job under **Recent Jobs** to view its scaling tool output, as shown in the following image.

![An image of the output window for the scaling tool.](media/tool-output.png)

