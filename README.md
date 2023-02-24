# Bicep deploy of AVD using Win 11 Ent Multi-Session, AADJ and FSLogix Profile containers
Used Bicep as a Domain Specific Language (DSL) to deploy a traditional Hub and Spoke architecture.  In turn, making use of AAD Kerberos for Hybrid identities.  Ensuring the identities originate from Windows Server AD on-premises, then synched through to AAD.  
<br>
Note 1. The deployment assumes an empty Resource Group and provisions a test Windows Server AD using AAD Connect Sync, handles AADJ and Registration of Session Host VMs.  Much of the solution is automated with few manual steps.<br><br>
Note 2.  The solution is dependant on configuring Azure Files Share-level and NTFS File/ Folder permissions.  The NTFS File/ Folder permissions need to be set from a machine that has line of sight of your AD DS.  The domain controller in Azure is mimicking AD DS on-premises.
<br><br><br>
<img src="https://github.com/gamcmaho/avdcloudonlypooled/blob/main/BicepAvdHubSpoke.jpg">
<br><br>
<h3>First generate a Token Expiration (now + 24 hours)</h3>
Using PowerShell run,<br><br>
$((get-date).ToUniversalTime().AddHours(24).ToString('yyyy-MM-ddTHH:mm:ss.fffffffZ'))
<br><br>
Note.  The maximum life time of a token is 30 days.
<br><br>
<h3>Git Clone the repo to your local device</h3>
git clone https://github.com/gamcmaho/avdcloudonlypooled.git
<br><br>
Create a new Resource Group in your Subscription
<br><br>
az login<br>
az account set -s "&ltsubscription name&gt"<br>
az group create --name "&ltresource group name&gt" --location "&ltlocation&gt"<br><br>
<h3>Use existing Azure Compute Gallery, or deploy a new gallery</h3>
To deploy a new gallery:
<br><br>
After cloning the repo, change to the "gallery" subdirectory of the "avd" directory<br>
Modify the gallery "parameters.json" providing values for:
<br><br>
location<br>
azure_compute_gallery_name
<br><br>
Note.  The Azure Compute Gallery name should be unique.
<br><br><br>
Then deploy a new Azure Compute Gallery by running:<br><br>
az deployment group create -g "&ltresource group name&gt" --template-file "gallery.bicep" --parameters "parameters.json"
<br><br>
<h3>Use existing Master image in your Azure Compute Gallery, or capture a new image</h3>
To prepare and capture a new image:
<br><br>
Deploy a Windows 11 Ent Multi-Session VM from the Azure Marketplace, e.g. win11-22h2-avd<br>
Install the latest Windows updates<br>
Depending on the Marketplace image used, FSLogix may already be installed.  If not, please install the latest version<br>
Add FSLogix items to the registry, remembering to update the storage account name below:
<br><br>
$regPath = "HKLM:\SOFTWARE\FSLogix\Profiles"<br>
New-ItemProperty -Path $regPath -Name Enabled -PropertyType DWORD -Value 1 -Force<br>
New-ItemProperty -Path $regPath -Name VHDLocations -PropertyType MultiString -Value \\&ltstorage-account-name&gt.file.core.windows.net\profiles -Force
<br><br>
In addition, create the following registry entries:<br>
https://learn.microsoft.com/en-us/azure/virtual-desktop/create-profile-container-azure-ad#configure-the-session-hosts<br><br>
reg add HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters /v CloudKerberosTicketRetrievalEnabled /t REG_DWORD /d 1<br><br>
reg add HKLM\Software\Policies\Microsoft\AzureADAccount /v LoadCredKeyFromProfile /t REG_DWORD /d 1<br><br>
<br><br>
Sysprep and Generalise by running %WINDIR%\system32\sysprep\sysprep.exe /generalize /shutdown /oobe<br>
From the virtual machine blade, once stopped, capture an image and store in your Azure Compute Gallery<br>
Then make a note of the Image URL for later reference.  See example Image URL below:
<br><br>
/subscriptions/&ltsubscription id&gt/resourceGroups/&ltresource group name&gt/providers/Microsoft.Compute/galleries/&ltAzure compute gallery name&gt/images/&ltimage name&gt
<br><br>
<h3>Deploy the AVD solution</h3>
Change directory to "avdcloudonlypooled" and modify the main "parameters.json" providing values for:<br><br>
location<br>
storage_account_name<br>
vm_gallery_image_id<br>
token_expiration_time<br>
total_instances<br>
vm_size
<br><br>
Note.  The storage account name needs to be globally unique.
<br><br><br>
Update the resource group name below and deploy.  Note.  The BICEP deployment typically takes around 20 minutes.
<br><br>
az deployment group create -g "&ltresource group name&gt" --template-file "main.bicep" --parameters "parameters.json"
<br><br><br>
Note.  During Host Pool creation, the following Custom RDP properties are enabled (targetisaadjoined and enablerdsaadauth).
<br><br>
<h3>Configure Windows Server AD</h3>
Azure Bastion to vm-dc<br>
Create Security Groups and Users to test Desktop Application Group (DAG) and Remote Application Group (RAG)<br>
For testing purposes, deploy AAD Connect Sync on vm-dc using PHS using Express setup.<br>
https://www.microsoft.com/en-us/download/details.aspx?id=47594
<br><br>
<h3>Prepare your Azure Files storage</h3>
Grant "Storage File Data SMB Share Contributor" to your Security Groups scoped to your Resource Group<br>
Navigate to your Azure Files share, then select Active Directory: Not configured<br>
Select Azure AD Kerberos Setup<br>
Check the box for Azure AD Kerberos and Save<br><br>
Navigate to AAD -> App registrations and select the App registration that matches the name of your Azure Files Storage account<br>
Select API permissions, then select Grant admin consent for your Domain name<br><br>
Note 1.  You must configure Azure Files share NTFS directory/ file permissions from a machine that has line of sight of the Windows Server AD.  However, your Session Host VMs do not require line of sight, since they are AAD joined.<br><br>
Note 2.  To access Azure File shares from an AAD joined VM for FSLogix profiles, you must configure the session hosts accordingly.  Either, making use of Intune policy, Group Policy or Registry entry.  One approach would be to include this Registry entry in the golden image deployed from Azure Compute Gallery.<br>
https://learn.microsoft.com/en-us/azure/virtual-desktop/create-profile-container-azure-ad#configure-the-session-hosts<br><br>
HKLM\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters /v CloudKerberosTicketRetrievalEnabled /t REG_DWORD /d 1
<br><br>
Note 3. When you use AAD with a roaming profile solution like FSLogix, the credential keys in Credential Manager must belong to the profile that's currently loading. This will let you load your profile on many different VMs instead of being limited to just one. To satisfy this requirement, please include the below Registry entry in your golden image deployed from Azure Compute Gallery.<br><br>
reg add HKLM\Software\Policies\Microsoft\AzureADAccount /v LoadCredKeyFromProfile /t REG_DWORD /d 1<br><br>
<h3>Grant additional Data RBAC role assignment to enable AAD authorization to your Session Host VMs</h3>
Navigate to the parenting Resource Group<br>
Using IAM, add either: Virtual Machine User Login or Virtual Machine Administrator Login<br>
Choose Data RBAC role assignment based on whether your users should have Administrative access, or not.
<br><br>
<h3>Grant DAG and RAG assignment to your Security Groups</h3>
Use Azure Portal and navigate to AVD -> Host Pools -> Session Hosts and confirm Domain Join and Health check status is healthy<br>
Navigate to AVD -> Application Groups -> dag-avd-> Assignments and add your first Security Group<br>
Then, confirm that your test user member of this security group can access a full desktop using the AVD Web Client<br>
https://client.wvd.microsoft.com/arm/webclient/index.html
<br><br>
Next, navigate to AVD -> Application Groups ->  rag-avd<br>
Add "Wordpad" as an Application and assign to your second Security Group<br>
Then, confirm your test user member of this security group can access "Wordpad" using the AVD Web Client<br>
https://client.wvd.microsoft.com/arm/webclient/index.html
<br><br>
<h3>From a security standpoint, enforce MFA for your new users in AAD</h3>
Using the Azure Portal, navigate to AAD -> Users -> Per-user MFA<br>
For each user, Enforce the use of MFA<br>
On first logon, follow instructions using the Microsoft Authenticator app
<br><br>
<h3>Congratulations, you're up and running with AVD!</h3>
