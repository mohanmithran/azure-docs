---
title: Create non-interactive authentication .NET applications in Azure HDInsight 
description: Learn how to create non-interactive authentication Microsoft .NET applications in Azure HDInsight.
ms.reviewer: jasonh
author: hrasheed-msft

ms.service: hdinsight
ms.custom: hdinsightactive
ms.topic: conceptual
ms.date: 05/14/2018
ms.author: hrasheed

---
# Create a non-interactive authentication .NET HDInsight application
You can run your Microsoft .NET Azure HDInsight application either under the application's own identity (non-interactive) or under the identity of the signed-in user of the application (interactive). This article shows you how to create a non-interactive authentication .NET application to connect to Azure and manage HDInsight. For a sample of an interactive application, see [Connect to Azure HDInsight](hdinsight-administer-use-dotnet-sdk.md#connect-to-azure-hdinsight). 

From your non-interactive .NET application, you need:

* Your Azure subscription tenant ID (also called a *directory ID*). See [Get tenant ID](../active-directory/develop/howto-create-service-principal-portal.md#get-tenant-id).
* The Azure Active Directory (Azure AD) application client ID. See [Create an Azure Active Directory application](../active-directory/develop/howto-create-service-principal-portal.md#create-an-azure-active-directory-application) and [Get an application ID](../active-directory/develop/howto-create-service-principal-portal.md#get-application-id-and-authentication-key).
* The Azure AD application secret key. See [Get application authentication key](../active-directory/develop/howto-create-service-principal-portal.md#get-application-id-and-authentication-key).

## Prerequisites
* An HDInsight cluster. See the [getting started tutorial](hadoop/apache-hadoop-linux-tutorial-get-started.md#create-cluster).

## Assign a role to the Azure AD application
Assign your Azure AD application a [role](../role-based-access-control/built-in-roles.md), to grant it permissions to perform actions. You can set the scope at the level of the subscription, resource group, or resource. The permissions are inherited to lower levels of scope. (For example, adding an application to the Reader role for a resource group means that the application can read the resource group and any resources in it.) In this tutorial, you set the scope at the resource group level. For more information, see [Use role assignments to manage access to your Azure subscription resources](../role-based-access-control/role-assignments-portal.md).

**To add the Owner role to the Azure AD application**

1. Sign in to the [Azure portal](https://portal.azure.com).
2. In the left menu, select **Resource groups**.
3. Select the resource group that has the HDInsight cluster on which you will run your Hive query later in this tutorial. If you have a large number of resource groups, you can use the filter to find the one you want.
4. On the resource group menu, select **Access control (IAM)**.
5. Select the **Role assignments** tab to see the current role assignments.
6. At the top of the page, select **Add role assignment**.
7. Follow the instructions to add the Owner role to your Azure AD application. After you successfully add the role, the application is listed under the Owner role. 

## Develop an HDInsight client application

1. Create a C# console application.
2. Add the following [NuGet](https://www.nuget.org/) packages:

        Install-Package Microsoft.Azure.Common.Authentication -Pre
        Install-Package Microsoft.Azure.Management.HDInsight -Pre
        Install-Package Microsoft.Azure.Management.Resources -Pre

3. Run the following code:

    ```csharp
    using System;
    using System.Security;
    using Microsoft.Azure;
    using Microsoft.Azure.Common.Authentication;
    using Microsoft.Azure.Common.Authentication.Factories;
    using Microsoft.Azure.Common.Authentication.Models;
    using Microsoft.Azure.Management.Resources;
    using Microsoft.Azure.Management.HDInsight;
    
    namespace CreateHDICluster
    {
        internal class Program
        {
            private static HDInsightManagementClient _hdiManagementClient;
    
            private static Guid SubscriptionId = new Guid("<Enter your Azure subscription ID>");
            private static string tenantID = "<Enter your tenant ID (also called directory ID)>";
            private static string applicationID = "<Enter your application ID>";
            private static string secretKey = "<Enter the application secret key>";
    
            private static void Main(string[] args)
            {
                var key = new SecureString();
                foreach (char c in secretKey) { key.AppendChar(c); }
    
                var tokenCreds = GetTokenCloudCredentials(tenantID, applicationID, key);
                var subCloudCredentials = GetSubscriptionCloudCredentials(tokenCreds, SubscriptionId);
    
                var resourceManagementClient = new ResourceManagementClient(subCloudCredentials);
                resourceManagementClient.Providers.Register("Microsoft.HDInsight");
    
                _hdiManagementClient = new HDInsightManagementClient(subCloudCredentials);
    
                var results = _hdiManagementClient.Clusters.List();
                foreach (var name in results.Clusters)
                {
                    Console.WriteLine("Cluster Name: " + name.Name);
                    Console.WriteLine("\t Cluster type: " + name.Properties.ClusterDefinition.ClusterType);
                    Console.WriteLine("\t Cluster location: " + name.Location);
                    Console.WriteLine("\t Cluster version: " + name.Properties.ClusterVersion);
                }
                Console.WriteLine("Press Enter to continue");
                Console.ReadLine();
            }
    
            /// Get the access token for a service principal and provided key.          
            public static TokenCloudCredentials GetTokenCloudCredentials(string tenantId, string clientId, SecureString secretKey)
            {
                var authFactory = new AuthenticationFactory();
                var account = new AzureAccount { Type = AzureAccount.AccountType.ServicePrincipal, Id = clientId };
                var env = AzureEnvironment.PublicEnvironments[EnvironmentName.AzureCloud];
                var accessToken =
                    authFactory.Authenticate(account, env, tenantId, secretKey, ShowDialog.Never).AccessToken;
    
                return new TokenCloudCredentials(accessToken);
            }
    
            public static SubscriptionCloudCredentials GetSubscriptionCloudCredentials(SubscriptionCloudCredentials creds, Guid subId)
            {
                return new TokenCloudCredentials(subId.ToString(), ((TokenCloudCredentials)creds).Token);
            }
        }
    }
    ```


## Next steps
* [Create an Azure Active Directory application and service principal in the Azure portal](../active-directory/develop/howto-create-service-principal-portal.md).
* Learn how to [authenticate a service principal with Azure Resource Manager](../active-directory/develop/howto-authenticate-service-principal-powershell.md).
* Learn about [Azure Role-Based Access Control (RBAC)](../role-based-access-control/role-assignments-portal.md).
