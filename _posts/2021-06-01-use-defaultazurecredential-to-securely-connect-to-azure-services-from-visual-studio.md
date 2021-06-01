---
published: true
featured: false
comments: false
title: Use DefaultAzureCredential to securely connect to Azure services from Visual Studio
tags:
  - Coding
  - Azure
  - Bicep
---

When you want to connect to an Azure service, you want to use a secure way for the authentication. One of the best techniques is to use a managed identity. Your application hosted inside Azure will get an identity managed by the system. This __service account__ is used to access other systems like storage accounts, keyvaults etc.

If you want to use this from your local machine, you do not have a managed identity. Luckily the library will have a couple of fallbacks.

Let's download a certificate from a keyvault:

```csharp
var keyVaultUrl = new Uri("https://" + keyVaultName + ".vault.azure.net");
var credentials = new DefaultAzureCredential();

var client = new CertificateClient(vaultUri: keyVaultUrl, credential: credentials);
var certificate = client.GetCertificate(certificateName).Value;
```

The `CertificateClient` will use the `DefaultAzureCredential` implementation to get a token. Internally it will try to do this using several [different implementations](https://docs.microsoft.com/en-us/dotnet/api/azure.identity.defaultazurecredential?view=azure-dotnet):

1. EnvironmentCredential
2. ManagedIdentityCredential
3. SharedTokenCacheCredential
4. VisualStudioCredential
5. VisualStudioCodeCredential
6. AzureCliCredential
7. InteractiveBrowserCredential

It will try in the order defined above, and if you run this from within Visual Studio, you will most likely not have an environment credential. You will need to tell [Visual Studio which credentials it needs to use](https://docs.microsoft.com/en-gb/dotnet/api/overview/azure/service-to-service-authentication#authenticating-to-azure-services). Within Visual Studio, go to `Tools > Options` to open the Options window. Select `Azure Service Authentication`, choose an account for local development, and select OK.

You might still run into an issue that it cannot find a valid token to use. Reconnecting the account can help, but sometimes it is unclear which account to use when you have multiple identities logged in.
In that scenario, you can explicitly specify a tenant id.

```csharp
var credentials = new DefaultAzureCredential(new DefaultAzureCredentialOptions
  {
      SharedTokenCacheTenantId = "87d576c4-6b5e-5927-b03d-7ea725510d38" // TenantId from the Azure AD
  });
```

Remember that you will need to assign the identity access to the resource, and when run locally, you will need to do the same for your account as that is the one used to access the resource.

One way is via Bicep:

```terraform
resource addAccessPolicyToKeyvault 'Microsoft.KeyVault/vaults/accessPolicies@2018-02-14' = {
  name: '${keyvaultName}/add'
  properties: {
    accessPolicies:  [
      {
        tenantId: '87d576c4-6b5e-5927-b03d-7ea725510d38'
        objectId: monitoringFunctions.outputs.principalId
        permissions: {
            keys: []
            secrets: [
               'Get'
               'List'
              ]
            certificates: [
               'Get' 
               'List'
              ]
          }
      }
    ]
  }
  dependsOn: [
     monitoringFunctions
     keyvault
  ]
}
```

The managed identity from an Azure Function will be added to already defined access policies of a keyvault and allows for Get and List access to secrets and certificates.

We get the principalId from the module using an output:

`output principalId string = functionApp.identity.principalId`

As you can see, we never stored any credentials and the same solution runs in Azure and from your local machine.
