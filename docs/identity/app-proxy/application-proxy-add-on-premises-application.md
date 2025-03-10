---
title:  Add an on-premises application for remote access through application proxy in Microsoft Entra ID.
description:  Microsoft Entra ID has an application proxy service that enables users to access on-premises applications by signing in with their Microsoft Entra ID account. This tutorial shows you how to prepare your environment for use with application proxy. Then, it uses the Microsoft Entra admin center to add an on-premises application to your Microsoft Entra tenant.
author: kenwith
manager: amycolannino
ms.service: entra-id
ms.subservice: app-proxy
ms.topic: tutorial
ms.date: 02/22/2024
ms.author: kenwith
ms.reviewer: ashishj
---

# Add an on-premises application for remote access through application proxy in Microsoft Entra ID

Microsoft Entra ID has an application proxy service that enables users to access on-premises applications by signing in with their Microsoft Entra account. To learn more about application proxy, see [What is application proxy?](overview-what-is-app-proxy.md). This tutorial prepares your environment for use with application proxy. Once your environment is ready, use the Microsoft Entra admin center to add an on-premises application to your tenant.

:::image type="content" source="./media/application-proxy-add-on-premises-application/app-proxy-diagram.png" alt-text="Application proxy Overview Diagram" lightbox="./media/application-proxy-add-on-premises-application/app-proxy-diagram.png":::

Connectors are a key part of application proxy. To learn more about connectors, see [Understand Microsoft Entra application proxy connectors](application-proxy-connectors.md).

In this tutorial, you:
- Open ports for outbound traffic and allows access to specific URLs.
- Install the connector on your Windows server, and registers it with application proxy.
- Verify the connector installed and registered correctly.
- Add an on-premises application to your Microsoft Entra tenant.
- Verify a test user can sign on to the application by using a Microsoft Entra account.

## Prerequisites

To add an on-premises application to Microsoft Entra ID, you need:
- An [Microsoft Entra ID P1 or P2 subscription](https://azure.microsoft.com/pricing/details/active-directory).
- An application administrator account.
- A synchronized set of user identities with an on-premises directory. Or create them directly in your Microsoft Entra tenants. Identity synchronization allows Microsoft Entra ID to preauthenticate users before granting them access to application proxy published applications. Synchronization also provides the necessary user identifier information to perform single sign-on (SSO).
- An understanding of application management in Microsoft Entra, see [View enterprise applications in Microsoft Entra](~/identity/enterprise-apps/view-applications-portal.md).
- An understanding of single sign-on (SSO), see [Understand single sign-on](~/identity/enterprise-apps/what-is-single-sign-on.md).

### Windows server

Application proxy requires Windows Server 2012 R2 or later. You install the application proxy connector on the server. The connector server communicates with the application proxy services in Microsoft Entra ID, and the on-premises applications that you plan to publish.

Use more than one Windows server for high availability in your production environment. One Windows server is sufficient for testing.

> [!IMPORTANT]
> **.NET Framework**
>
> You must have .NET version 4.7.1 or higher to install, or upgrade, application proxy version 1.5.3437.0 or later. Windows Server 2012 R2 and Windows Server 2016 don't have this by default.
>
> See [How to: Determine which .NET Framework versions are installed](/dotnet/framework/migration-guide/how-to-determine-which-versions-are-installed) for more information.
> 
> **HTTP 2.0**
>
>  If you're installing the connector on *Windows Server 2019* or later, you must disable `HTTP2` protocol support in the `WinHttp` component for *Kerberos Constrained Delegation* to properly work. This is disabled by default in earlier versions of supported operating systems. Adding the following registry key and restarting the server disables it on Windows Server 2019. Note that this is a machine-wide registry key.
>
> ```
> Windows Registry Editor Version 5.00
>
> [HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings\WinHttp]
> "EnableDefaultHTTP2"=dword:00000000
> ```
>
> The key can be set via PowerShell with the following command:
>
> ```
> Set-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings\WinHttp\' -Name EnableDefaultHTTP2 -Value 0
> ```

#### Recommendations for the connector server

- Optimize performance between the connector and the application. Physically locate the connector server close to the application servers. For more information, see [Optimize traffic flow with Microsoft Entra application proxy](application-proxy-network-topology.md).
- Make sure the connector server and the web application servers are in the same Active Directory domain or span trusting domains. Having the servers in the same domain or trusting domains is a requirement for using single sign-on (SSO) with integrated Windows authentication (IWA) and Kerberos Constrained Delegation (KCD). If the connector server and web application servers are in different Active Directory domains, use resource-based delegation for single sign-on. For more information, see [KCD for single sign-on with application proxy](how-to-configure-sso-with-kcd.md).

> [!WARNING]
> If you've deployed Microsoft Entra Password Protection Proxy, do not install Microsoft Entra application proxy and Microsoft Entra Password Protection Proxy together on the same machine. Microsoft Entra application proxy and Microsoft Entra Password Protection Proxy install different versions of the Microsoft Entra Connect Agent Updater service. These different versions are incompatible when installed together on the same machine.

#### Transport Layer Security (TLS) requirements

The Windows connector server must have TLS 1.2 enabled before you install the application proxy connector.

To enable TLS 1.2:

1. Set registry keys.

   ```
   Windows Registry Editor Version 5.00

   [HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2]
   [HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Client]
   "DisabledByDefault"=dword:00000000
   "Enabled"=dword:00000001
   [HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\TLS 1.2\Server]
   "DisabledByDefault"=dword:00000000
   "Enabled"=dword:00000001
   [HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\.NETFramework\v4.0.30319]
   "SchUseStrongCrypto"=dword:00000001
   ```

1. Restart the server.

> [!NOTE]
> Microsoft is updating Azure services to use TLS certificates from a different set of Root Certificate Authorities (CAs). This change is being made because the current CA certificates do not comply with one of the CA/Browser Forum Baseline requirements. For more information, see [Azure TLS certificate changes](/azure/security/fundamentals/tls-certificate-changes).

## Prepare your on-premises environment

Enable communication to Microsoft data centers to prepare your environment for Microsoft Entra application proxy. The connector must make HTTPS (TCP) requests to application proxy. A common issue occurs when a firewall is blocking network traffic.

> [!IMPORTANT]
> If you're installing the connector for Azure Government cloud follow the [prerequisites](~/identity/hybrid/connect/reference-connect-government-cloud.md#allow-access-to-urls) and [installation steps](~/identity/hybrid/connect/reference-connect-government-cloud.md#install-the-agent-for-the-azure-government-cloud). Azure Government cloud requires access to a different set of URLs and an additional parameter to run the installation.

### Open ports

Open the following ports to **outbound** traffic.

| Port number | Use |
| ----------- | ------------------------------------------------------------ |
| 80          | Downloading certificate revocation lists (CRLs) while validating the TLS/SSL certificate |
| 443         | All outbound communication with the application proxy service |

If your firewall enforces traffic according to originating users, also open ports 80 and 443 for traffic from Windows services that run as a Network Service.

> [!NOTE]
> Errors occur when there's a networking issue. Check if the required ports are open. For more information about troubleshooting issues related to connector errors, see [Troubleshoot application proxy problems and error messages](application-proxy-troubleshoot.md#connector-errors).

### Allow access to URLs

Allow access to the following URLs:

| URL | Port | Use |
| --- | --- | --- |
| `*.msappproxy.net` <br> `*.servicebus.windows.net` | `443/HTTPS` | Communication between the connector and the application proxy cloud service. |
| `crl3.digicert.com` <br> `crl4.digicert.com` <br> `ocsp.digicert.com` <br> `crl.microsoft.com` <br> `oneocsp.microsoft.com` <br> `ocsp.msocsp.com`<br> | `80/HTTP`   | The connector uses these URLs to verify certificates.        |
| `login.windows.net` <br> `secure.aadcdn.microsoftonline-p.com` <br> `*.microsoftonline.com` <br> `*.microsoftonline-p.com` <br> `*.msauth.net` <br> `*.msauthimages.net` <br> `*.msecnd.net` <br> `*.msftauth.net` <br> `*.msftauthimages.net` <br> `*.phonefactor.net` <br> `enterpriseregistration.windows.net` <br> `management.azure.com` <br> `policykeyservice.dc.ad.msft.net` <br> `ctldl.windowsupdate.com` <br> `www.microsoft.com/pkiops` | `443/HTTPS` | The connector uses these URLs during the registration process. |
| `ctldl.windowsupdate.com` <br> `www.microsoft.com/pkiops` | `80/HTTP` | The connector uses these URLs during the registration process. |

Allow connections to `*.msappproxy.net`, `*.servicebus.windows.net`, and other URLs if your firewall or proxy lets you configure access rules based on domain suffixes. Or, allow access to the [Azure IP ranges and Service Tags - Public Cloud](https://www.microsoft.com/download/details.aspx?id=56519). The IP ranges are updated each week.

> [!IMPORTANT]
> Avoid all forms of inline inspection and termination on outbound TLS communications between Microsoft Entra application proxy connectors and Microsoft Entra application proxy services.

### Domain Name System (DNS) for Microsoft Entra application proxy endpoints

Public DNS records for Microsoft Entra application proxy endpoints are chained CNAME records pointing to an A record. Setting up the records this way ensures fault tolerance and flexibility. The Microsoft Entra application proxy connector always accesses host names with the domain suffixes `*.msappproxy.net` or `*.servicebus.windows.net`. However, during the name resolution the CNAME records might contain DNS records with different host names and suffixes. Due to the difference, you must ensure that the device (depending on your setup - connector server, firewall, outbound proxy) can resolve all the records in the chain and allows connection to the resolved IP addresses. Since the DNS records in the chain might be changed from time to time, we can't provide you with any list DNS records.

## Install and register a connector

To use application proxy, install a connector on each Windows server you're using with the application proxy service. The connector is an agent that manages the outbound connection from the on-premises application servers to application proxy in Microsoft Entra ID. The connector can install on servers with authentication agents such as Microsoft Entra Connect.

To install the connector:

1. Sign in to the [Microsoft Entra admin center](https://entra.microsoft.com) as at least a [Application Administrator](~/identity/role-based-access-control/permissions-reference.md#application-administrator).
1. Select your username in the upper-right corner. Verify you're signed in to a directory that uses application proxy. If you need to change directories, select **Switch directory** and choose a directory that uses application proxy.
1. Browse to **Identity** > **Applications** > **Enterprise applications** > **Application proxy**.
1. Select **Download connector service**.
1. Read the Terms of Service. When you're ready, select **Accept terms & Download**.
1. At the bottom of the window, select **Run** to install the connector. An install wizard opens.
1. Install the service by following the instructions in the wizard. When you're prompted to register the connector with the application proxy for your Microsoft Entra tenant, provide your application administrator credentials.

    - If **IE Enhanced Security Configuration** is set to **On** in Internet Explorer (IE), the registration screen isn't visible. To get access, follow the instructions in the error message. Make sure that **Internet Explorer Enhanced Security Configuration** is set to **Off**.

### General remarks

If you already installed a connector, reinstall it to get the latest version. To see information about previously released versions and what changes they include, see [Application proxy: Version Release History](application-proxy-release-version-history.md).

If you choose to have more than one Windows server for your on-premises applications, you need to install and register the connector on each server. You can organize the connectors into connector groups. For more information, see [Connector groups](application-proxy-connector-groups.md).

If you install connectors in different regions, you should optimize traffic by selecting the closest application proxy cloud service region with each connector group. To learn more, see [Optimize traffic flow with Microsoft Entra application proxy](application-proxy-network-topology.md).

If your organization uses proxy servers to connect to the internet, you need to configure them for application proxy. For more information, see [Work with existing on-premises proxy servers](application-proxy-configure-connectors-with-proxy-servers.md).

For information about connectors, capacity planning, and how they stay up-to-date, see [Understand Microsoft Entra application proxy connectors](application-proxy-connectors.md).

## Verify the connector installed and registered correctly

You can use the Microsoft Entra admin center or your Windows server to confirm that a new connector installed correctly. For information about troubleshooting application proxy issues, see [Debug application proxy application issues](application-proxy-debug-apps.md).

### Verify the installation through Microsoft Entra admin center

To confirm the connector installed and registered correctly:

1. Sign in to the [Microsoft Entra admin center](https://entra.microsoft.com) as at least a [Application Administrator](~/identity/role-based-access-control/permissions-reference.md#application-administrator).
1. Select your username in the upper-right corner. Verify you're signed in to a directory that uses application proxy. If you need to change directories, select **Switch directory** and choose a directory that uses application proxy.
1. Browse to **Identity** > **Applications** > **Enterprise applications** > **Application proxy**.
1. Verify the details of the connector. The connectors should be expanded by default. An active green label indicates that your connector can connect to the service. However, even if the label is green, a network issue could still block the connector from receiving messages.

    ![Microsoft Entra application proxy connectors](./media/application-proxy-add-on-premises-application/app-proxy-connectors.png)

For more help with installing a connector, see [Problem installing the application proxy connector](application-proxy-connector-installation-problem.md).

### Verify the installation through your Windows server

To confirm the connector installed and registered correctly:

1. Open the Windows Services Manager by clicking the **Windows** key and entering *services.msc*.
1. Check to see if the status for the following two services is **Running**.
   - **Microsoft Entra application proxy connector** enables connectivity.
   - **Microsoft Entra application proxy connector Updater** is an automated update service. The updater checks for new versions of the connector and updates the connector as needed.

1. If the status for the services isn't **Running**, right-click to select each service and choose **Start**.

## Add an on-premises app to Microsoft Entra ID

Add on-premises applications to Microsoft Entra ID.
1. Sign in to the [Microsoft Entra admin center](https://entra.microsoft.com) as at least a [Application Administrator](~/identity/role-based-access-control/permissions-reference.md#application-administrator).
1. Browse to **Identity** > **Applications** > **Enterprise applications**.
1. Select **New application**.
1. Select **Add an on-premises application** button, which appears about halfway down the page in the **On-premises applications** section. Alternatively, you can select **Create your own application** at the top of the page and then select **Configure application proxy for secure remote access to an on-premises application**.
1. In the **Add your own on-premises application** section, provide the following information about your application:

    | Field  | Description |
    | :--------------------- | :----------------------------------------------------------- |
    | **Name** | The name of the application that appears on My Apps and in the Microsoft Entra admin center. |
    | **Maintenance Mode** | Select if you would like to enable maintenance mode and temporarily disable access for all users to the application. |
    | **Internal URL** | The URL for accessing the application from inside your private network. You can provide a specific path on the backend server to publish, while the rest of the server is unpublished. In this way, you can publish different sites on the same server as different apps, and give each one its own name and access rules.<br><br>If you publish a path, make sure that it includes all the necessary images, scripts, and style sheets for your application. For example, if your app is at `https://yourapp/app` and uses images located at `https://yourapp/media`, then you should publish `https://yourapp/` as the path. This internal URL doesn't have to be the landing page your users see. For more information, see [Set a custom home page for published apps](application-proxy-configure-custom-home-page.md). |
    | **External URL** | The address for users to access the app from outside your network. If you don't want to use the default application proxy domain, read about [custom domains in Microsoft Entra application proxy](./how-to-configure-custom-domain.md). |
    | **Pre Authentication** | How application proxy verifies users before giving them access to your application.<br><br>**Microsoft Entra ID** - Application proxy redirects users to sign in with Microsoft Entra ID, which authenticates their permissions for the directory and application. We recommend keeping this option as the default so that you can take advantage of Microsoft Entra security features like Conditional Access and multifactor authentication. **Microsoft Entra ID** is required for monitoring the application with Microsoft Defender for Cloud Apps.<br><br>**Passthrough** - Users don't have to authenticate against Microsoft Entra ID to access the application. You can still set up authentication requirements on the backend. |
    | **Connector Group** | Connectors process the remote access to your application, and connector groups help you organize connectors and apps by region, network, or purpose. If you don't have any connector groups created yet, your app is assigned to **Default**.<br><br>If your application uses WebSockets to connect, all connectors in the group must be version 1.5.612.0 or later. |

1. If necessary, configure **Additional settings**. For most applications, you should keep these settings in their default states.

    | Field | Description |
    | :------------------------------ | :----------------------------------------------------------- |
    | **Backend Application Timeout** | Set this value to **Long** only if your application is slow to authenticate and connect. At default, the backend application timeout has a length of 85 seconds. When set too long, the backend timeout is increased to 180 seconds. |
    | **Use HTTP-Only Cookie** | Select to have application proxy cookies include the HTTPOnly flag in the HTTP response header. If using Remote Desktop Services, keep the option unselected. |
    | **Use Persistent Cookie**| Keep the option unselected. Only use this setting for applications that can't share cookies between processes. For more information about cookie settings, see [Cookie settings for accessing on-premises applications in Microsoft Entra ID](./application-proxy-configure-cookie-settings.md).
    | **Translate URLs in Headers** | Keep the option selected unless your application required the original host header in the authentication request. |
    | **Translate URLs in Application Body** | Keep the option unselected unless HTML links are hardcoded to other on-premises applications and don't use custom domains. For more information, see [Link translation with application proxy](./application-proxy-configure-hard-coded-link-translation.md).<br><br>Select if you plan to monitor this application with Microsoft Defender for Cloud Apps. For more information, see [Configure real-time application access monitoring with Microsoft Defender for Cloud Apps and Microsoft Entra ID](./application-proxy-integrate-with-microsoft-cloud-application-security.md). |
    | **Validate Backend TLS/SSL Certificate** | Select to enable backend TLS/SSL certificate validation for the application. |

1. Select **Add**.

## Test the application

You're ready to test the application is added correctly. In the following steps, you add a user account to the application, and try signing in.

### Add a user for testing

Before adding a user to the application, verify the user account already has permissions to access the application from inside the corporate network.

To add a test user:

1. Select **Enterprise applications**, and then select the application you want to test.
2. Select **Getting started**, and then select **Assign a user for testing**.
3. Under **Users and groups**, select **Add user**.
4. Under **Add assignment**, select **Users and groups**. The **User and groups** section appears.
5. Choose the account you want to add.
6. Choose **Select**, and then select **Assign**.

### Test the sign-on

To test authentication to the application:

1. From the application you want to test, select **application proxy**.
2. At the top of the page, select **Test Application** to run a test on the application and check for any configuration issues.
3. Make sure to first launch the application to test signing into the application, then download the diagnostic report to review the resolution guidance for any detected issues.

For troubleshooting, see [Troubleshoot application proxy problems and error messages](./application-proxy-troubleshoot.md).

## Clean up resources

Don't forget to delete any of the resources you created in this tutorial when you're done.

## Troubleshooting

Learn about common issues and how to troubleshoot them.

### Create the Application/Setting the URLs
Check the error details for information and suggestions for how to fix the application. Most error messages include a suggested fix. To avoid common errors, verify:

- You're an administrator with permission to create an application proxy application
- The internal URL is unique
- The external URL is unique
- The URLs start with http or https, and end with a “/”
- The URL should be a domain name, not an IP address

The error message should display in the top-right corner when you create the application. You can also select the notification icon to see the error messages.

### Configure connectors/connector groups

If you're having difficulty configuring your application because of warning about the connectors and connector groups, see instructions on enabling application proxy for details on how to download connectors. If you want to learn more about connectors, see the [connectors documentation](application-proxy-connectors.md).

If your connectors are inactive, they're unable to reach the service. Often because all of the required ports aren't open. To see a list of required ports, see the prerequisites section of the enabling application proxy documentation.

### Upload certificates for custom domains

Custom Domains allow you to specify the domain of your external URLs. To use custom domains, you need to upload the certificate for that domain. For information on using custom domains and certificates, see [Working with custom domains in Microsoft Entra application proxy](how-to-configure-custom-domain.md).

If you're encountering issues uploading your certificate, look for the error messages in the portal for additional information on the problem with the certificate. Common certificate problems include:

- Expired certificate
- Certificate is self-signed
- Certificate is missing the private key

The error message display in the top-right corner as you try to upload the certificate. You can also select the notification icon to see the error messages.

## Next steps

- [Publish applications using Microsoft Entra application proxy](application-proxy-add-on-premises-application.md)
- [Enable application proxy in the Microsoft Entra admin center](application-proxy-add-on-premises-application.md)
- [Work with custom domains in Microsoft Entra application proxy](how-to-configure-custom-domain.md)
- [Configure single sign-on](~/identity/enterprise-apps/plan-sso-deployment.md#choosing-a-single-sign-on-method)
