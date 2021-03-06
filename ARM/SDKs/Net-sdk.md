> Azure Resource Manager Community Documentation - Beta Version

> Work in progress - This community driven documentation is considered to be in preview stage at this time. Documentation might contain errors, might not cover every aspect or might lack complete parts, even important parts. Please help us make this documentation better by [contributing](CONTRIBUTING.md) anything that you think would make it better.


---

#Azure SDK for .NET

## Introduction
WARNING!
At the time of writing this documentation the Azure SDK for .NET is still in preview. Things might not be as stable as you would expect or might even change in the near future. Use the documentation at your own risk.

Azure SDK for .NET is provided as a set of NuGet Packages that helps you call most of the APIs exposed by Azure Resrouce Manager. If the SDK doesn't expose the requried functionality you can easily combine the SDK with regular calls to the ARM REST API behind the scenes.

This documentation is not intended to describe all aspects of Azure SDK for .NET, Azure ARM APIs or Visual Studio, but is rather provided as a fast way for you to get started.

A full downloadable sample project from where all code snippets below have been taken, can be found [here](Samples/Net).

## Setting up the environment

The following NuGet Packages are needed to complete the tasks in this documentation. Install them from the graphical version of NuGet Manager or from the NuGet Manager Console. 

```bash
Install-Package Microsoft.Azure.Common.Authentication -Pre
Install-Package Microsoft.Azure.Management.Compute -Pre
Install-Package Microsoft.Azure.Management.Network -Pre
Install-Package Microsoft.Azure.Management.Resources -Pre
Install-Package Microsoft.Azure.Management.Storage -Pre
```

(As of the time of writing, these packages are only provided as "preview packages" hence you must add the "-Pre" switch in order to have them installed)

## Authentication

Authentication against the Azure APIs are done by passign a token to your requests. You can receive that token by first authenticating against Azure AD. A more indept explanation how it all works can be found in the [REST API documentation](Rest-api.md).

### Registering your application for programatic access

The process of regestering an application for programatic access to Azure and Azure AD is described in [the Authentication documentation](../security/Authentication.md).
Please make sure you follow that instruction or make sure you have allready registered an Azure AD Application and have the corresponding:

* Azure AD Tenant ID
* Application Client ID
* Application Client Secret

### Receiving the AccessToken from code

The authentication token can easily be aquired with the below lines of code, passing in only your Azure AD Tenant ID, your Azure AD Application Client ID and the Azure AD Application Client Secret. Save the token for several requests since it by default is valid for 1 hour.

```csharp
private static AuthenticationResult GetAccessToken(string tenantId, string clientId, string clientSecret)
{
    Console.WriteLine("Aquiring Access Token from Azure AD");
    AuthenticationContext authContext = new AuthenticationContext
        ("https://login.windows.net/" /* AAD URI */
            + $"{tenantId}.onmicrosoft.com" /* Tenant ID or AAD domain */);

    var credential = new ClientCredential(clientId, clientSecret);

    AuthenticationResult token = authContext.AcquireToken("https://management.azure.com/", credential);

    Console.WriteLine($"Token: {token.AccessToken}");
    return token;
}
```

### Querying Azure subscriptions attached to the authenticated application

One of the first things you might want to do is querying what Azure Subscriptions that are associated with the just authenticated application. The Subscription ID for your targeted subscription will be mandatory to pass to each API call you do from now on.

The below sample code queries Azure APIs directly using the REST API, i.e. not using any features in Azure SDK for .NET.

```csharp
async private static Task<List<string>> GetSubscriptionsAsync(string token)
{
    Console.WriteLine("Querying for subscriptions");
    const string apiVersion = "2015-01-01";

    var client = new HttpClient();
    client.BaseAddress = new Uri("https://management.azure.com/");
    client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);
    client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));

    var response = await client.GetAsync($"subscriptions?api-version={apiVersion}");

    var jsonResponse = response.Content.AsString();

    var subscriptionIds = new List<string>();
    dynamic json = JsonConvert.DeserializeObject(jsonResponse);

    for (int i = 0; i < json.value.Count; i++)
    {
        subscriptionIds.Add(json.value[i].subscriptionId.Value);
    }

    Console.WriteLine($"Found {subscriptionIds.Count} subscription(s)");
    return subscriptionIds;
}
```

Notice taht we do get a JSON response from Azure that we then extract the subscription IDs from in order to return a list of IDs. All the subsequent calls to Azure ARM APIs in this documentation only uses a single Azure Subscription ID, so if your application are associated with several subscriptions, just pick the right one of them and pass as a parameter going forward.

From here on every call we do against the Azure APIs will use the Azure SDK for .NET so the code do look a little bit different.

### Wrapping the token as a TokenCredentials Object

All of the following API calls will need the token you received from Azure AD in the format of a "TokenCredentials" object. Such an object is easily created by just passing the raw token as a parameter to the class' constructor.

```csharp
var credentials = new TokenCredentials(token);
```

## Creating a Resource Group

Everything in Azure is focused around the Resource Groups, so let's start by creating one. General resources and resource groups are handled by the ResourceManagementClient and as any of the following more specialiced Management Clients that we will use, you need to provide your credentials as well as a Subscription ID to identify what subscription you want to work with.

```csharp
private static async Task<ResourceGroup> CreateResourceGroupAsync(TokenCredentials credentials, string subscriptionId, string resourceGroup, string location)
{
    Console.WriteLine($"Creating Resource Group {resourceGroup}");
    var resourceClient = new ResourceManagementClient(credentials) { SubscriptionId = subscriptionId };
    return await resourceClient.ResourceGroups.CreateOrUpdateAsync(resourceGroup,
        new ResourceGroup
        {
            Location = location
        });
}
```

## Creating resources manually or using templates

There are several different ways of interacting with the Azure Resource Manager APIs but the two big differnt ways of doing it is:

* Manually, by calling specific Resource Providers manually or
* By using an Azure Resource Manager Template (aka. ARM Template)

Using the ARM Templates have the following benefits:

* You declaratively specify what you want the end result should look like, rather than how it should be achieved
* You don't have to manually handle parallel execution of your deployments, ARM will do that for you.
* You don't have to learn C# or any other language in order to deploy an ARM Template even though you can use any language to start a templated deployment.
* The domain specific language, DSL, that is used in the templates are build using JSON and is easy enough to understans by anyone that have worked with JSON.

Even though, with all the benefits of the templates we will start by showing you how to call the API's manually.

### Creating a Virtual Machine, piece by piece

So now we have our subscription and our resource group. If we now want to deploy a Virtual Machine we need to figure out what parts actually make up a Virtual Machine and it turns out it's quite a few parts:

* 1 or many Storage Accounts, for storing persistent disks
* 1 or many Public IP Address, PIP, for beeing accessable from the Internet (includes a DNS name)
* 1 or many Virtual Networks, VNET, for internal communication between your resources
* 1 or many Network Interface Cards, NIC, to allow the VM to communicate
* 1 or many Virtual Machines, VM, to run our software

Another interesting piece is also how some of these resources can be created in parallel while other once can not. For example:

* NICs, depend on PIP and VNet
* VMs, depend on NICs and Storage Accounts

You need to make sure you don't try to instantiate any resources before the required dependencies have been created. The full [sample](Samples/Net) provided with this documentation shows how you can efficiently create your resources in parallel while still keeping track on what's been created.

#### Creating the Storage Account

You need a storage account to store the Virtual HDDs for you Virtual Machine. If you have an existing storage account you can use it for several VMs, but remember to spread your load across several storage accounts not to run into limits. Remember that the type of Storage Account and it's location can limit the VM Size you can chose since not all VM Sizes are available in all regions and/or for all storage account types.

```csharp
private static async Task<StorageAccount> CreateStorageAccountAsync(TokenCredentials credentials, string subscriptionId, string resourceGroup, string location, string storageAccountName, AccountType accountType = AccountType.StandardLRS)
{
    Console.WriteLine("Creating Storage Account");
    var storageClient = new StorageManagementClient(credentials) { SubscriptionId = subscriptionId };
    return await storageClient.StorageAccounts.CreateAsync(resourceGroup, storageAccountName,
        new StorageAccountCreateParameters
        {
            Location = location,
            AccountType = accountType,
        });
}
```

#### Creating the Public IP Address, PIP

The public IP Address is what makes your resources in Azure accessable from Internet. Together with the IP Address you'll also be assigned a fully qualified domain name, FQDN, that you can use for easier access.

```csharp
private static Task<PublicIPAddress> CreatePublicIPAddressAsync(TokenCredentials credentials, string subscriptionId, string resourceGroup, string location, string pipAddressName, string pipDnsName)
{
    Console.WriteLine("Creating Public IP");
    var networkClient = new NetworkManagementClient(credentials) { SubscriptionId = subscriptionId };
    var createPipTask = networkClient.PublicIPAddresses.CreateOrUpdateAsync(resourceGroup, pipAddressName,
        new PublicIPAddress
        {
            Location = location,
            DnsSettings = new PublicIPAddressDnsSettings { DomainNameLabel = pipDnsName },
            PublicIPAllocationMethod = "Dynamic" // This sample doesn't support Static IP Addresses but could be extended to do so
        });

    return createPipTask;
}
```

#### Creating the Virtual Network, VNET

Every VM created with the ARM APIs need to be part of a Virtual Network, even if the VM is alone in it. The virtual network must contain at least one subnet, but you can have many do divide and protect your resources in several subnets.

```csharp
private static Task<VirtualNetwork> CreateVirtualNetworkAsync(TokenCredentials credentials, string subscriptionId, string resourceGroup, string location, string vNetName, string vNetAddressPrefix, Subnet[] subnets)
{
    Console.WriteLine("Creating Virtual Network");
    var networkClient = new NetworkManagementClient(credentials) { SubscriptionId = subscriptionId };
    var createVNetTask = networkClient.VirtualNetworks.CreateOrUpdateAsync(resourceGroup, vNetName,
        new VirtualNetwork
        {
            Location = location,
            AddressSpace = new AddressSpace(new[] { vNetAddressPrefix }),
            Subnets = subnets
        });

    return createVNetTask;
}
```

#### Creating the Network Interface Card, NIC

The Network Interface Card, NIC, is what connects your VM with the Virtual Network it recides in. A VM can have many NICs and hence be associated with several Virtual Networks. This sample assumes you are only attaching your VMs to one VNET.

```csharp
private static Task<NetworkInterface> CreateNetworkInterfaceAsync(TokenCredentials credentials, string subscriptionId, string resourceGroup, string location, string nicName, string nicIPConfigName, PublicIPAddress pip, Subnet subnet)
{
    Console.WriteLine("Creating Network Interface");
    var networkClient = new NetworkManagementClient(credentials) { SubscriptionId = subscriptionId };
    var createNicTask = networkClient.NetworkInterfaces.CreateOrUpdateAsync(resourceGroup, nicName,
        new NetworkInterface()
        {
            Location = location,
            IpConfigurations = new[] {
                new NetworkInterfaceIPConfiguration
                {
                    Name = nicIPConfigName,
                    PrivateIPAllocationMethod = "Dynamic",
                    PublicIPAddress = pip,
                    Subnet = subnet
                }
            }
        });

    return createNicTask;
}
```

#### Creating the Virtual Machine, VM

Finally it's time to create the actual Virtual Machine. The VM depends direcly or inderectly of all of the above created resources, so you need to wait for all of the above to be ready before you try to provision a VM. Provisioning a VM is what takes the longest time of the above resources so expect your application to be waiting a while for this to happen.

```csharp
private static async Task<VirtualMachine> CreateVirtualMachineAsync(TokenCredentials credentials, string subscriptionId, string resourceGroup, string location, string storageAccountName, string vmName, string vmSize, string vmAdminUsername, string vmAdminPassword, string vmImagePublisher, string vmImageOffer, string vmImageSku, string vmImageVersion, string vmOSDiskName, string nicId)
{
    Console.WriteLine("Creating Virtual Machine (this may take a while)");
    var computeClient = new ComputeManagementClient(credentials) { SubscriptionId = subscriptionId };
    var vm = await computeClient.VirtualMachines.CreateOrUpdateAsync(resourceGroup, vmName,
        new VirtualMachine
        {
            Location = location,
            HardwareProfile = new HardwareProfile(vmSize),
            OsProfile = new OSProfile(vmName, vmAdminUsername, vmAdminPassword),
            StorageProfile = new StorageProfile(
                new ImageReference
                {
                    Publisher = vmImagePublisher,
                    Offer = vmImageOffer,
                    Sku = vmImageSku,
                    Version = vmImageVersion
                },
                new OSDisk
                {
                    Name = vmOSDiskName,
                    Vhd = new VirtualHardDisk($"http://{storageAccountName}.blob.core.windows.net/vhds/{vmOSDiskName}.vhd"),
                    Caching = "ReadWrite",
                    CreateOption = "FromImage"
                }),
            NetworkProfile = new NetworkProfile(
                new[] { new NetworkInterfaceReference { Id = nicId } }),
            DiagnosticsProfile = new DiagnosticsProfile(
                new BootDiagnostics
                {
                    Enabled = true,
                    StorageUri = $"http://{storageAccountName}.blob.core.windows.net"
                })
        });

    return vm;
}
```

### Using a templated deployment

Please read and follow the [Deploy Azure resources using .NET libraries and a template](https://azure.microsoft.com/en-us/documentation/articles/arm-template-deployment/#step-4-create-the-credentials-that-are-used-to-authenticate-requests) tutorial for detailed instructions on how to deploy a template.

In short deploying a template is much easier than doing in manual and the below code show you how to do it by pointing at the URIs where you have the template and a parameters file. 

```csharp
private static async Task<DeploymentExtended> CreateTemplatedDeployment(TokenCredentials credentials, string subscriptionId, string resourceGroup, string templateUri, string parametersUri)
{
    var resourceClient = new ResourceManagementClient(credentials) { SubscriptionId = subscriptionId };

    return await resourceClient.Deployments.BeginCreateOrUpdateAsync(resourceGroup, "mytemplateddeployment", new Deployment(
        new DeploymentProperties()
        {
            Mode = DeploymentMode.Incremental,
            TemplateLink = new TemplateLink(templateUri),
            ParametersLink = new ParametersLink(parametersUri)
        }));

}
```

