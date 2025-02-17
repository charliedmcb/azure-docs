---
title: TLS termination with Azure Key Vault certificates
description: Learn how you can integrate Azure Application Gateway with Key Vault for server certificates that are attached to HTTPS-enabled listeners.
services: application-gateway
author: vhorne
ms.service: application-gateway
ms.topic: conceptual
ms.date: 11/16/2020
ms.author: victorh
---

# TLS termination with Key Vault certificates

[Azure Key Vault](../key-vault/general/overview.md) is a platform-managed secret store that you can use to safeguard secrets, keys, and TLS/SSL certificates. Azure Application Gateway supports integration with Key Vault for server certificates that are attached to HTTPS-enabled listeners. This support is limited to the v2 SKU of Application Gateway.

Key Vault integration offers two models for TLS termination:

- You can explicitly provide TLS/SSL certificates attached to the listener. This model is the traditional way to pass TLS/SSL certificates to Application Gateway for TLS termination.
- You can optionally provide a reference to an existing Key Vault certificate or secret when you create an HTTPS-enabled listener.

Application Gateway integration with Key Vault offers many benefits, including:

- Stronger security, because TLS/SSL certificates aren't directly handled by the application development team. Integration allows a separate security team to:
  * Set up application gateways.
  * Control application gateway lifecycles.
  * Grant permissions to selected application gateways to access certificates that are stored in your key vault.
- Support for importing existing certificates into your key vault. Or use Key Vault APIs to create and manage new certificates with any of the trusted Key Vault partners.
- Support for automatic renewal of certificates that are stored in your key vault.

Application Gateway currently supports software-validated certificates only. Hardware security module (HSM)-validated certificates are not supported. After Application Gateway is configured to use Key Vault certificates, its instances retrieve the certificate from Key Vault and install them locally for TLS termination. The instances also poll Key Vault at 4-hour intervals to retrieve a renewed version of the certificate, if it exists. If an updated certificate is found, the TLS/SSL certificate currently associated with the HTTPS listener is automatically rotated.

> [!NOTE]
> The Azure portal only supports KeyVault Certificates, not secrets. Application Gateway still supports referencing secrets from KeyVault, but only through non-Portal resources like PowerShell, CLI, API, ARM templates, etc. 

> [!WARNING]
> Azure Application Gateway currently only supports Key Vault accounts in the same subscription as the Application Gateway resource. Choosing a Key Vault under a different subscription than your Application Gateway will result in a failure.

## How integration works

Application Gateway integration with Key Vault requires a three-step configuration process:

1. **Create a user-assigned managed identity**

   You create or reuse an existing user-assigned managed identity, which Application Gateway uses to retrieve certificates from Key Vault on your behalf. For more information, see [Create, list, delete, or assign a role to a user-assigned managed identity using the Azure portal](../active-directory/managed-identities-azure-resources/how-to-manage-ua-identity-portal.md). This step creates a new identity in the Azure Active Directory tenant. The identity is trusted by the subscription that's used to create the identity.

1. **Configure your key vault**

   You then either import an existing certificate or create a new one in your key vault. The certificate will be used by applications that run through the application gateway. In this step, you can also use a Key Vault Secret which also allows storing a password-less, base-64 encoded PFX file. We recommend using a “Certificate” type because of the autorenewal capability that's available with this type of objects in the Key Vault. After you've created a Certificate or a Secret, you must define Access Policies in the Key Vault to allow the identity to be granted get access to the secret.
   
   > [!IMPORTANT]
   > Starting March 15th 2021, Key Vault recognizes Azure Application Gateway as one of the Trusted Services, thus allowing you to build a secure network boundary in Azure. This gives you an ability to deny access to traffic from all networks (including internet traffic) to Key Vault but still make it accessible for Application Gateway resource under your subscription. 

   > You can configure your Application Gateway in a restricted network of Key Vault in the following manner. <br />
   > a) Under Key Vault’s Networking blade <br />
   > b) choose Private endpoint and selected networks in "Firewall and Virtual Networks" tab <br/>
   > c) then using Virtual Networks, add your Application Gateway’s virtual network and Subnet. During the process also configure ‘Microsoft.KeyVault' service endpoint by selecting its checkbox. <br/>
   > d) Finally, select “Yes” to allow Trusted Services to bypass Key Vault’s firewall. <br/>
   > 
   > ![Key Vault Firewall](media/key-vault-certs/key-vault-firewall.png)


   > [!NOTE]
   > If you deploy the application gateway via an ARM template, either by using the Azure CLI or PowerShell, or via an Azure application deployed from the Azure portal, the SSL certificate is stored in the key vault as a base64-encoded PFX file. You must complete the steps in [Use Azure Key Vault to pass secure parameter value during deployment](../azure-resource-manager/templates/key-vault-parameter.md). 
   >
   > It's particularly important to set `enabledForTemplateDeployment` to `true`. The certificate may be passwordless or it may have a password. In the case of a certificate with a password, the following example shows a possible configuration for the `sslCertificates` entry in the `properties` for the ARM template configuration for an app gateway. The values of `appGatewaySSLCertificateData` and `appGatewaySSLCertificatePassword` are looked up from the key vault as described in the section [Reference secrets with dynamic ID](../azure-resource-manager/templates/key-vault-parameter.md#reference-secrets-with-dynamic-id). Follow the references backward from `parameters('secretName')` to see how the lookup happens. If the certificate is passwordless, omit the `password` entry.
   >   
   > ```
   > "sslCertificates": [
   >     {
   >         "name": "appGwSslCertificate",
   >         "properties": {
   >             "data": "[parameters('appGatewaySSLCertificateData')]",
   >             "password": "[parameters('appGatewaySSLCertificatePassword')]"
   >         }
   >     }
   > ]
   > ```

1. **Configure the application gateway**

   After you complete the two preceding steps, you can set up or modify an existing application gateway to use the user-assigned managed identity. For more information see, [Set-AzApplicationGatewayIdentity](/powershell/module/az.network/set-azapplicationgatewayidentity).

   You can also configure the HTTP listener’s TLS/SSL certificate to point to the complete URI of the Key Vault certificate or secret ID.

   ![Key vault certificates](media/key-vault-certs/ag-kv.png)

## Next steps

[Configure TLS termination with Key Vault certificates by using Azure PowerShell](configure-keyvault-ps.md)
