# On-Boarding for VM Insights - Private Preview
For any questions, to give feedback, or to have your workspace white-listed for the VM Insights Private Preview:
* email: vminsights@microsoft.com

This readme gives steps on how to On-Board to VM Insights using scripts and ARM templates.

To access VM Insights in Azure Monitor and the VM blade, go to aka.ms/vminsights.  You can on-board for a single VM from the VM blade under the "Insights" item in the table of contents.  The instructions below are for on-boarding at scale.

We can organize setup steps as follows:
- [Log Analytics Workspace setup](#log-analytics-workspace-setup)
- [Per VM and VM Scale Set setup](#per-vm-and-vm-scale-set-setup)

## Supported Log Analytics Workspace Regions

**Important!** Only Log Analytics Workspaces in following regions are supported:
* West Central US
* East US
* Southeast Asia
* West Europe

## Supported VM Operating Systems
| OS Version | Performance | Maps |
| ---------- | ----------- | ---- |
| Windows Server 2016 | X | X |
| Windows Server 2012R2 | X | X |
| Windows 2012, 2008 R2 SP1 | X | X |
| RHEL 6, 7 | X | X |
| Ubuntu 14.04, 16.04  | X | X |
| Cent OS Linux 6, 7 | X | X |
| SLES 11 | X | X |
| SLES 12 | X |  |
| Oracle Linux 6 | X | X |
| Oracle Linux 7 | X |  |
| Debian 8, 9.4 | X |  |

## Log Analytics Workspace setup
If you have already configured a single VM for VM Insights through UI experience, then these steps will already have been taken care of for you and you can move on to enabling for more VM's.

Summary of steps:
- Create a Log Analytics workspace in a [supported region](#supported-log-analytics-workspace-regions)
- Install the ServiceMap and InfrastructureInsights Solutions
- Configure the workspace to collect performance counters

#### Create a Log Analytics workspace in a supported region
See Log Analytics Documentation: [Create a Log Analytics workspace in the Azure portal](https://docs.microsoft.com/en-us/azure/log-analytics/log-analytics-quick-create-workspace)

Reminder, VM Insights [Supported Workspace regions](#supported-log-analytics-workspace-regions)

#### Install the ServiceMap and InfrastructureInsights Solutions
We have provided an ARM template [InstallSolutionsForVMInsights.json](InstallSolutionsForVMInsights.json) that will deploy the ServiceMap and InfrastructureInsights solutions to your Log Analytics workspace.

You can find information on how to deploy ARM templates [here](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-template-deploy)
Example of deploying with PowerShell:
```powershell
New-AzureRmResourceGroupDeployment -Name DeploySolutions -TemplateFile InstallSolutionsForVMInsights.json -ResourceGroupName <workspace resource group name> -WorkspaceName <workspace name> -WorkspaceLocation <workspace location- example: eastus>
```

You can also deploy directly from Azure Portal using <a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fdougbrad%2FOnBoardVMInsights%2Fmaster%2FInstallSolutionsForVMInsights.json" target="_blank">this link</a>
#### Configure the workspace to collect performance counters
Compute Performance provides summarized and per-resource views of key metrics for on-prem VMs, Azure VMs, VM Scale Sets, and cloud services.  These steps are required to configure counters:

**TODO:** Update with script to enable performance counter collection, for now steps to Manually Configure to collect the counters

Find your Log Analytics workspace resource in the Azure portal:

![Find your Log Analytics workspace resource in the Azure portal](images/FindLogAnalyticsWorkspace.png)

Click “Advanced settings” in the “Settings” menu:

![Click “Advanced settings” in the “Settings” menu](images/ClickAdvancedSettings.png)

Click “Data”/”Windows Performance Counters” to configure Windows host collection for VM Insights

![alt text](images/ClickDataWindowsPerformanceCounters.png)

Add the following three counters, setting the Sample Interval to 60 seconds or less.
* Processor(_Total)\% Processor Time
* Memory(*)\Available Mbytes
* LogicalDisk(*)\% Free Space

Select “Data”/”Linux Performance Counters” to configure Linux node collection

Select at least the following three counters. Set the Sample Interval to 60 seconds:
* Processor(*)\% Processor Time
* Memory(*)\Available MBytes Memory
* Logical Disk(*)\% Used Space


Click “Save” in the toolbar.


## Per VM and VM Scale Set setup

On each VM or VM Scale Set the following is needed
- Log Analytics VM Extension pointing to a workspace in a [supported region](#supported-log-analytics-workspace-regions)
- Dependency Agent VM Extension

For the private preview we are providing a script [Install-VMInsights.ps1](Install-VMInsights.ps1) to accomplish this.

To quickly download the powershell to your local filesystem, run following:
```powershell
$client = new-object System.Net.WebClient
$client.DownloadFile(“https://raw.githubusercontent.com/dougbrad/OnBoardVMInsights/master/Install-VMInsights.ps1”,“Install-VMInsights.ps1”) 
``` 

This script will iterate through VM's and VM Scale sets in a Subcription, or further scoped by ResourceGroup or Name.
For each VM or VM ScaleSet it will check the currently installed VM extensions and install if needed.

This script requires Azure PowerShell, you can find instructions to install for Windows at [Install Azure PowerShell](https://docs.microsoft.com/en-us/powershell/azure/install-azurerm-ps?view=azurermps-6.5.0)



You can run Get-Help to get details and an example of usage:

```powershell
Get-Help .\Install-VMInsights.ps1 -Detailed

SYNOPSIS
    Configure VM's and VM Scale Sets for VM Insights:
    - Installs Log Analytics VM Extension configured to supplied Log Analytics Workspace
    - Installs Dependency Agent VM Extension

    Can be applied to:
    - Subscription
    - Resource Group in a Subscription
    - Specific VM/VM Scale Set

    Script will show you list of VM's/VM Scale Sets that will apply to and let you confirm to continue.
    Use -Approve switch to run without prompting, if all required parameters are provided.

    If the extensions are already installed will not install again.
    Use -ReInstall switch if you need to for example update the workspace.

    Use -WhatIf if you would like to see what would happen in terms of installs, what workspace configured to, and status of the
    extension.

SYNTAX
    D:\GitHub\OnBoardVMInsights\Install-VMInsights.ps1 [-WorkspaceId] <String> [-WorkspaceKey] <String> [-SubscriptionId]
    <String> [[-ResourceGroup] <String>] [[-Name] <String>] [-ReInstall] [-TriggerVmssManualVMUpdate] [-Approve] [-WhatIf]
    [-Confirm] [<CommonParameters>]

PARAMETERS
    -WorkspaceId <String>
        Log Analytics WorkspaceID (GUID) for the data to be sent to

    -WorkspaceKey <String>
        Log Analytics Workspace primary or secondary key

    -SubscriptionId <String>
        SubscriptionId for the VMs/VM Scale Sets

    -ResourceGroup <String>
        <Optional> Resource Group to which the VMs or VM Scale Sets belong to

    -Name <String>
        <Optional> To install to a single VM/VM Scale Set

    -ReInstall [<SwitchParameter>]
        <Optional> If VM/VM Scale Set is already configured for a different workspace, set this to change to the
        new workspace

    -TriggerVmssManualVMUpdate [<SwitchParameter>]
        <Optional> Set this flag to trigger update of VM instances in a scale set whose upgrade policy is set to
        Manual

    -Approve [<SwitchParameter>]
        <Optional> Gives the approval for the installation to start with no confirmation prompt for the listed VM's/VM Scale Sets

    -WhatIf [<SwitchParameter>]
        <Optional> See what would happen in terms of installs.
        If extension is already installed will show what workspace is currently configured, and status of the VM
        extension

    -Confirm [<SwitchParameter>]
        <Optional> Confirm every action

    <CommonParameters>
        This cmdlet supports the common parameters: Verbose, Debug,
        ErrorAction, ErrorVariable, WarningAction, WarningVariable,
        OutBuffer, PipelineVariable, and OutVariable. For more information, see
        about_CommonParameters (https:/go.microsoft.com/fwlink/?LinkID=113216).

    -------------------------- EXAMPLE 1 --------------------------

    .\Install-VMInsights.ps1 -WorkspaceId <WorkspaceId>-WorkspaceKey <WorkspaceKey> -SubscriptionId
    <SubscriptionId> -ResourceGroup <ResourceGroup>        
```

Example of running:

```powershell
$WorkspaceId = "<GUID>"
$WorkspaceKey = "<Key>"
$SubscriptionId = "<GUID>"
.\Install-VMInsights.ps1 -WorkspaceId $WorkspaceId -WorkspaceKey $WorkspaceKey -SubscriptionId $SubscriptionId -ResourceGroup db-ws

Getting list of VM's or VM ScaleSets matching criteria specified

VM's or VM ScaleSets matching criteria:

db-ws-1 VM running
db-ws2012 VM running

This operation will install the Log Analytics and Dependency Agent extensions on above 2 VM's or VM Scale Sets.
VM's in a non-running state will be skipped.
Extension will not be re-installed if already installed. Use /ReInstall if desired, for example to update workspace

Confirm
Continue?
[Y] Yes  [N] No  [S] Suspend  [?] Help (default is "Y"): y

db-ws-1 : Deploying DependencyAgentWindows with name DAExtension
db-ws-1 : Successfully deployed DependencyAgentWindows
db-ws-1 : Deploying MicrosoftMonitoringAgent with name MMAExtension
db-ws-1 : Successfully deployed MicrosoftMonitoringAgent
db-ws2012 : Deploying DependencyAgentWindows with name DAExtension
db-ws2012 : Successfully deployed DependencyAgentWindows
db-ws2012 : Deploying MicrosoftMonitoringAgent with name MMAExtension
db-ws2012 : Successfully deployed MicrosoftMonitoringAgent

Summary:

Already Onboarded: (0)

Succeeded: (4)
db-ws-1 : Successfully deployed DependencyAgentWindows
db-ws-1 : Successfully deployed MicrosoftMonitoringAgent
db-ws2012 : Successfully deployed DependencyAgentWindows
db-ws2012 : Successfully deployed MicrosoftMonitoringAgent

Connected to different workspace: (0)

Not running - start VM to configure: (0)

Failed: (0)
```
## Onboarding a PaaS v1 Cloud Service to VM Insights

1.	Get workspace id and workspace key which will be used in the following steps 
2.	Edit the ServiceDefinition.csdef file of your cloud service and add the following two settings:
```powershell
      <Setting name="WorkspaceId" />
      <Setting name="WorkspaceKey" />
```
3.	Add install-mma-and-da.cmd and install-mma-and-da.ps1 to your project. The files, as provided, assume that they will be placed at the root of the Cloud Service project. If you wish to relocate them you must update the provided install-mma-and-da.cmd file and the snippet shown below in step 6.
4.	Add a startup task to you ServiceDefinition.csdef to execute install-mma-and-da.cmd with the correct arguments:
```powershell
      <Task commandLine="install-mma-and-da.cmd" executionContext="elevated" taskType="simple">
        <Environment>
          <Variable name="IsEmulated">
            <RoleInstanceValue xpath="/RoleEnvironment/Deployment/@emulated" />
          </Variable>
          <Variable name="WorkspaceId">
            <RoleInstanceValue xpath="/RoleEnvironment/CurrentInstance/ConfigurationSettings/ConfigurationSetting[@name='WorkspaceId']/@value" />
          </Variable>
          <Variable name="WorkspaceKey">
            <RoleInstanceValue xpath="/RoleEnvironment/CurrentInstance/ConfigurationSettings/ConfigurationSetting[@name='WorkspaceKey']/@value" />
          </Variable>
        </Environment>
      </Task>
```
5.	Update the cscfg file(s) supplying values for these settings
6.	Build and deploy your service
