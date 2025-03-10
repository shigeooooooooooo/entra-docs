---
title: Microsoft Entra passwordless sign-in
description: Learn about options for passwordless sign-in to Microsoft Entra ID using FIDO2 security keys or Microsoft Authenticator

ms.service: entra-id
ms.subservice: authentication
ms.topic: conceptual
ms.date: 01/22/2024

ms.author: justinha
author: justinha
manager: amycolannino
ms.reviewer: calui
---
# Passwordless authentication options for Microsoft Entra ID

Features like multifactor authentication (MFA) are a great way to secure your organization, but users often get frustrated with the additional security layer on top of having to remember their passwords. Passwordless authentication methods are more convenient because the password is removed and replaced with something you have, plus something you are or something you know.

| Authentication  | Something you have | Something you are or know |
| --- | --- | --- |
| Passwordless | Windows 10 Device, phone, or security key | Biometric or PIN |

Each organization has different needs when it comes to authentication. Microsoft Azure and Azure Government offer the following four passwordless authentication options that integrate with Microsoft Entra ID:

- Windows Hello for Business
- Microsoft Authenticator 
- FIDO2 security keys
- Certificate-based authentication

![Authentication: Security versus convenience](./media/concept-authentication-passwordless/passwordless-convenience-security.png)

## Windows Hello for Business

Windows Hello for Business is ideal for information workers that have their own designated Windows PC. The biometric and PIN credentials are directly tied to the user's PC, which prevents access from anyone other than the owner. With public key infrastructure (PKI) integration and built-in support for single sign-on (SSO), Windows Hello for Business provides a convenient method for seamlessly accessing corporate resources on-premises and in the cloud.

![Example of a user sign-in with Windows Hello for Business](./media/concept-authentication-passwordless/windows-hellow-sign-in.jpeg)

The following steps show how the sign-in process works with Microsoft Entra ID:

![Diagram that outlines the steps involved for user sign-in with Windows Hello for Business](./media/concept-authentication-passwordless/windows-hello-flow.png)

1. A user signs into Windows using biometric or PIN gesture. The gesture unlocks the Windows Hello for Business private key and is sent to the Cloud Authentication security support provider, referred to as the *Cloud AP provider*.
1. The Cloud AP provider requests a nonce (a random arbitrary number that can be used just once) from Microsoft Entra ID.
1. Microsoft Entra ID returns a nonce that's valid for 5 minutes.
1. The Cloud AP provider signs the nonce using the user's private key and returns the signed nonce to the Microsoft Entra ID.
1. Microsoft Entra ID validates the signed nonce using the user's securely registered public key against the nonce signature.  Microsoft Entra ID validates the signature and then validates the returned signed nonce. When the nonce is validated, Microsoft Entra ID creates a primary refresh token (PRT) with session key that is encrypted to the device's transport key and returns it to the Cloud AP provider.
1. The Cloud AP provider receives the encrypted PRT with session key. The Cloud AP provider uses the device's private transport key to decrypt the session key and protects the session key using the device's Trusted Platform Module (TPM).
1. The Cloud AP provider returns a successful authentication response to Windows. The user is then able to access Windows as well as cloud and on-premises applications without the need to authenticate again (SSO).

The Windows Hello for Business [planning guide](/windows/security/identity-protection/hello-for-business/hello-planning-guide) can be used to help you make decisions on the type of Windows Hello for Business deployment and the options you'll need to consider.

## Microsoft Authenticator

You can also allow your employee's phone to become a passwordless authentication method. You may already be using the Authenticator app as a convenient multi-factor authentication option in addition to a password. You can also use the Authenticator App as a passwordless option.

![Sign in to Microsoft Edge with the Microsoft Authenticator](./media/concept-authentication-passwordless/concept-web-sign-in-microsoft-authenticator-app.png)

The Authenticator App turns any iOS or Android phone into a strong, passwordless credential. Users can sign in to any platform or browser by getting a notification to their phone, matching a number displayed on the screen to the one on their phone, and then using their biometric (touch or face) or PIN to confirm. Refer to [Download and install the Microsoft Authenticator](https://support.microsoft.com/account-billing/download-and-install-the-microsoft-authenticator-app-351498fc-850a-45da-b7b6-27e523b8702a) for installation details.

Passwordless authentication using the Authenticator app follows the same basic pattern as Windows Hello for Business. It's a little more complicated as the user needs to be identified so that Microsoft Entra ID can find the Authenticator app version being used:

![Diagram that outlines the steps involved for user sign-in with the Microsoft Authenticator App](./media/concept-authentication-passwordless/authenticator-app-flow.png)

1. The user enters their username.
1. Microsoft Entra ID detects that the user has a strong credential and starts the Strong Credential flow.
1. A notification is sent to the app via Apple Push Notification Service (APNS) on iOS devices, or via Firebase Cloud Messaging (FCM) on Android devices.
1. The user receives the push notification and opens the app.
1. The app calls Microsoft Entra ID and receives a proof-of-presence challenge and nonce.
1. The user completes the challenge by entering their biometric or PIN to unlock private key.
1. The nonce is signed with the private key and sent back to Microsoft Entra ID.
1. Microsoft Entra ID performs public/private key validation and returns a token.

To get started with passwordless sign-in, complete the following how-to:

> [!div class="nextstepaction"]
> [Enable passwordless sign using the Authenticator app](howto-authentication-passwordless-phone.md)

## FIDO2 security keys

The FIDO (Fast IDentity Online) Alliance helps to promote open authentication standards and reduce the use of passwords as a form of authentication. FIDO2 is the latest standard that incorporates the web authentication (WebAuthn) standard.

FIDO2 security keys are an unphishable standards-based passwordless authentication method that can come in any form factor. Fast Identity Online (FIDO) is an open standard for passwordless authentication. FIDO allows users and organizations to leverage the standard to sign in to their resources without a username or password using an external security key or a platform key built into a device.

Users can register and then select a FIDO2 security key at the sign-in interface as their main means of authentication. These FIDO2 security keys are typically USB devices, but could also use Bluetooth or NFC. With a hardware device that handles the authentication, the security of an account is increased as there's no password that could be exposed or guessed.

FIDO2 security keys can be used to sign in to their Microsoft Entra ID or Microsoft Entra hybrid joined Windows 10 devices and get single-sign on to their cloud and on-premises resources. Users can also sign in to supported browsers. FIDO2 security keys are a great option for enterprises who are very security sensitive or have scenarios or employees who aren't willing or able to use their phone as a second factor.

We have a reference document for which [browsers support FIDO2 authentication with Microsoft Entra ID](fido2-compatibility.md), and best practices for developers wanting to [support FIDO2 auth in the applications they develop](~/identity-platform/support-fido2-authentication.md).

![Sign in to Microsoft Edge with a security key](./media/concept-authentication-passwordless/concept-web-sign-in-security-key.png)

The following process is used when a user signs in with a FIDO2 security key:

![Diagram that outlines the steps involved for user sign-in with a FIDO2 security key](./media/concept-authentication-passwordless/fido2-security-key-flow.png)

1. The user plugs the FIDO2 security key into their computer.
2. Windows detects the FIDO2 security key.
3. Windows sends an authentication request.
4. Microsoft Entra ID sends back a nonce.
5. The user completes their gesture to unlock the private key stored in the FIDO2 security key's secure enclave.
6. The FIDO2 security key signs the nonce with the private key.
7. The primary refresh token (PRT) token request with signed nonce is sent to Microsoft Entra ID.
8. Microsoft Entra ID verifies the signed nonce using the FIDO2 public key.
9. Microsoft Entra ID returns PRT to enable access to on-premises resources.

### FIDO2 security key providers

The following providers offer FIDO2 security keys of different form factors that are known to be compatible with the passwordless experience. We encourage you to evaluate the security properties of these keys by contacting the vendor as well as the [FIDO Alliance](https://fidoalliance.org/).

| Provider | Biometric | USB | NFC | BLE |
|:-|:-:|:-:|:-:|:-:|
| [AuthenTrend](https://authentrend.com/about-us/#pg-35-3) | ![y] | ![y]| ![y]| ![y]|
| [ACS](https://www.acs.com.hk/) | ![n] | ![y]| ![y]| ![n]|
| [ATOS](https://atos.net/en/solutions/cyber-security/iot-and-ot-security/smart-card-solution-cardos-for-iot) | ![n] | ![y]| ![y]| ![n]|
| [Ciright](https://www.cyberonecard.com/) | ![n] | ![n]| ![y]| ![n]|
| [Composecure](https://www.composecure.com/arculus) | ![n] | ![n]| ![y]| ![n]|
| [Crayonic](https://www.crayonic.com/keyvault) | ![y] | ![n]| ![y]| ![y]|
| [Cryptnox](https://cryptnox.com/) | ![n] | ![y]| ![y]| ![n]|
| [Ensurity](https://www.ensurity.com/contact) | ![y] | ![y]| ![n]| ![n]|
| [Excelsecu](https://www.excelsecu.com/productdetail/esecufido2secu.html) | ![y] | ![y]| ![y]| ![y]|
| [Feitian](https://shop.ftsafe.us/pages/microsoft) | ![y] | ![y]| ![y]| ![y]|
| [Fortinet](https://www.fortinet.com/) | ![n] | ![y]| ![n]| ![n]|
| [Giesecke + Devrient (G+D)](https://www.gi-de.com/en/identities/enterprise-security/hardware-based-authentication) | ![y] | ![y]| ![y]| ![y]|
| [Google](https://store.google.com/us/product/titan_security_key) | ![n] | ![y]| ![y]| ![n]|
| [GoTrustID Inc.](https://www.gotrustid.com/idem-key) | ![n] | ![y]| ![y]| ![y]|
| [HID](https://www.hidglobal.com/products/crescendo-key) | ![n] | ![y]| ![y]| ![n]|
| [HIDEEZ](https://hideez.com/products/hideez-key-4) | ![n] | ![y]| ![y]| ![y]|
| [Hypersecu](https://www.hypersecu.com/hyperfido) | ![n] | ![y]| ![n]| ![n]|
| [Hypr](https://www.hypr.com/true-passwordless-mfa) | ![y] | ![y]| ![n]| ![y]|
| [Identiv](https://www.identiv.com/products/logical-access-control/utrust-fido2-security-keys/nfc) | ![n] | ![y]| ![y]| ![n]|
| [IDmelon Technologies Inc.](https://www.idmelon.com/#idmelon) | ![y] | ![y]| ![y]| ![y]|
| [Kensington](https://www.kensington.com/solutions/product-category/why-biometrics/) | ![y] | ![y]| ![n]| ![n]|
| [KONA I](https://www.konai.com/main) | ![y] | ![n]| ![y]| ![y]|
| [NeoWave](https://neowave.fr/en/products/fido-range/) | ![n] | ![y]| ![y]| ![n]|
| [Nymi](https://www.nymi.com/nymi-band) | ![y] | ![n]| ![y]| ![n]|
| [Octatco](https://octatco.com/) | ![y] | ![y]| ![n]| ![n]|
| [OneSpan Inc.](https://www.onespan.com/products/fido) | ![n] | ![y]| ![n]| ![y]|
| [PONE Biometrics](https://ponebiometrics.com/) | ![y] | ![n]| ![n]| ![y]|
| [Precision Biometric](https://www.innait.com/product/fido/) | ![n] | ![y]| ![n]| ![n]|
| [RSA](https://www.rsa.com/products/securid/) | ![n] | ![y]| ![n]| ![n]|
| [Sentry](https://sentryenterprises.com/) | ![n] | ![n]| ![y]| ![n]|
| [SmartDisplayer](https://www.smartdisplayer.com/fido) | ![y] | ![y]| ![y]| ![y]|
| [Swissbit](https://www.swissbit.com/en/products/ishield-key/) | ![n] | ![y]| ![y]| ![n]|
| [Thales Group](https://cpl.thalesgroup.com/access-management/authenticators/fido-devices) | ![y] | ![y]| ![y]| ![n]|
| [Thetis](https://thetis.io/collections/fido2) | ![y] | ![y]| ![y]| ![y]|
| [Token2 Switzerland](https://www.token2.swiss/shop/product/token2-t2f2-alu-fido2-u2f-and-totp-security-key) | ![y] | ![y]| ![y]| ![n]|
| [Token Ring](https://www.tokenring.com/) | ![y] | ![n]| ![y]| ![n]|
| [TrustKey Solutions](https://www.trustkeysolutions.com/en/sub/product.form) | ![y] | ![y]| ![n]| ![n]|
| [VinCSS](https://passwordless.vincss.net) | ![n] | ![y]| ![n]| ![n]|
| [WiSECURE Technologies](https://wisecure-tech.com/en-us/zero-trust/fido/authtron) | ![n] | ![y]| ![n]| ![n]|
| [Yubico](https://www.yubico.com/solutions/passwordless/) | ![y] | ![y]| ![y]| ![n]|

<!--Image references-->
[y]: ./media/fido2-compatibility/yes.png
[n]: ./media/fido2-compatibility/no.png

> [!NOTE]
> If you purchase and plan to use NFC-based security keys, you need a supported NFC reader for the security key. The NFC reader isn't an Azure requirement or limitation. Check with the vendor for your NFC-based security key for a list of supported NFC readers.

If you're a vendor and want to get your device on this list of supported devices, check out our guidance on how to [become a Microsoft-compatible FIDO2 security key vendor](concept-fido2-hardware-vendor.md).

To get started with FIDO2 security keys, complete the following how-to:

> [!div class="nextstepaction"]
> [Enable passwordless sign using FIDO2 security keys](howto-authentication-passwordless-security-key.md)

## Certificate-based authentication

Microsoft Entra certificate-based authentication (CBA) enables customers to allow or require users to authenticate directly with X.509 certificates against their Microsoft Entra ID for applications and browser sign-in. CBA enables customers to adopt phishing-resistant authentication and sign in with an X.509 certificate against their Public Key Infrastructure (PKI).

:::image type="content" border="false" source="./media/concept-certificate-based-authentication/cloud-native-cert.png" alt-text="Diagram of Microsoft Entra certificate-based authentication.":::

### Key benefits of using Microsoft Entra CBA

| Benefits | Description |
|---------|---------|
| Great user experience |- Users who need certificate-based authentication can now directly authenticate against Microsoft Entra ID and not have to invest in federated AD FS.<br>- Portal UI enables users to easily configure how to map certificate fields to a user object attribute to look up the user in the tenant ([certificate username bindings](concept-certificate-based-authentication-technical-deep-dive.md#understanding-the-username-binding-policy))<br>- Portal UI to [configure authentication policies](concept-certificate-based-authentication-technical-deep-dive.md#understanding-the-authentication-binding-policy) to help determine which certificates are single-factor versus multifactor. |
| Easy to deploy and administer |- Microsoft Entra CBA is a free feature, and you don't need any paid editions of Microsoft Entra ID to use it. <br>- No need for complex on-premises deployments or network configuration.<br>- Directly authenticate against Microsoft Entra ID. |
| Secure |- On-premises passwords don't need to be stored in the cloud in any form.<br>- Protects your user accounts by working seamlessly with Microsoft Entra Conditional Access policies, including Phishing-Resistant [multifactor authentication](concept-mfa-howitworks.md) (MFA requires [licensed edition](concept-mfa-licensing.md)) and blocking legacy authentication.<br>- Strong authentication support where users can define authentication policies through the certificate fields, such as issuer or policy OID (object identifiers), to determine which certificates qualify as single-factor versus multifactor.<br>- The feature works seamlessly with [Conditional Access features](~/identity/conditional-access/overview.md) and authentication strength capability to enforce MFA to help secure your users. |


### Supported scenarios

The following scenarios are supported:

- User sign-ins to web browser-based applications on all platforms.
- User sign-ins to Office mobile apps on iOS/Android platforms as well as Office native apps in Windows, including Outlook, OneDrive, and so on.
- User sign-ins on mobile native browsers.
- Support for granular authentication rules for multifactor authentication by using the certificate issuer **Subject** and **policy OIDs**.
- Configuring certificate-to-user account bindings by using any of the certificate fields:
  - Subject Alternate Name (SAN) PrincipalName and SAN RFC822Name
  - Subject Key Identifier (SKI) and SHA1PublicKey
- Configuring certificate-to-user account bindings by using any of the user object attributes:
  - User Principal Name
  - onPremisesUserPrincipalName
  - CertificateUserIds

## Supported scenarios

The following considerations apply:

- Administrators can enable passwordless authentication methods for their tenant.
- Administrators can target all users or select users/Security groups within their tenant for each method.
- Users can register and manage these passwordless authentication methods in their account portal.
- Users can sign in with these passwordless authentication methods:
   - Authenticator app: Works in scenarios where Microsoft Entra authentication is used, including across all browsers, during Windows 10 setup, and with integrated mobile apps on any operating system.
   - Security keys: Work on lock screen for Windows 10 and the web in supported browsers like Microsoft Edge (both legacy and new Edge).
- Users can use passwordless credentials to access resources in tenants where they are a guest, but they may still be required to perform MFA in that resource tenant. For more information, see [Possible double multi-factor authentication](~/external-id/current-limitations.md#possible-double-multi-factor-authentication).  
- Users may not register passwordless credentials within a tenant where they are a guest, the same way that they do not have a password managed in that tenant.  

## Unsupported scenarios

We recommend no more than 20 sets of keys for each passwordless method for any user account. As more keys are added, the user object size increases, and you may notice degradation for some operations. In that case, you should remove unnecessary keys. For more information and the PowerShell cmdlets to query and remove keys, see 
[Using WHfBTools PowerShell module for cleaning up orphaned Windows Hello for Business Keys](https://support.microsoft.com/topic/using-whfbtools-powershell-module-for-cleaning-up-orphaned-windows-hello-for-business-keys-779d1f3f-bb2d-c495-0f6b-9aeb940eeafb). The topic uses **/UserPrincipalName** optional parameter to query only keys for a specific user. The permissions required are to run as an administrator or the specified user.

When you use PowerShell to create a CSV file with all of the existing keys, carefully identify the keys that you need to keep, and remove those rows from the CSV. Then use the modified CSV with PowerShell to delete the remaining keys to bring the account key count under the limit.
 
It is safe to delete any key reported as "Orphaned"="True" in the CSV. An orphaned key is one for a device that is not longer registered in Entra ID. If removing all Orphans still doesn't bring the User account below the limit it is necessary to look at the "DeviceId" and "CreationTime" columns to identify which keys to target for deletion. Be careful to remove any row in the CSV for keys you want to keep. Keys for any DeviceID corresponding to devices the user actively uses should be removed from the CSV before the deletion step.

## Choose a passwordless method

The choice between these three passwordless options depends on your company's security, platform, and app requirements.

Here are some factors for you to consider when choosing Microsoft passwordless technology:

||**Windows Hello for Business**|**Passwordless sign-in with the Authenticator app**|**FIDO2 security keys**|
|:-|:-|:-|:-|
|**Pre-requisite**| Windows 10, version 1809 or later<br>Microsoft Entra ID| Authenticator app<br>Phone (iOS and Android devices)|Windows 10, version 1903 or later<br>Microsoft Entra ID|
|**Mode**|Platform|Software|Hardware|
|**Systems and devices**|PC with a built-in Trusted Platform Module (TPM)<br>PIN and biometrics recognition |PIN and biometrics recognition on phone|FIDO2 security devices that are Microsoft compatible|
|**User experience**|Sign in using a PIN or biometric recognition (facial, iris, or fingerprint) with Windows devices.<br>Windows Hello authentication is tied to the device; the user needs both the device and a sign-in component such as a PIN or biometric factor to access corporate resources.|Sign in using a mobile phone with fingerprint scan, facial or iris recognition, or PIN.<br>Users sign in to work or personal account from their PC or mobile phone.|Sign in using FIDO2 security device (biometrics, PIN, and NFC)<br>User can access device based on organization controls and authenticate based on PIN, biometrics using devices such as USB security keys and NFC-enabled smartcards, keys, or wearables.|
|**Enabled scenarios**| Password-less experience with Windows device.<br>Applicable for dedicated work PC with ability for single sign-on to device and applications.|Password-less anywhere solution using mobile phone.<br>Applicable for accessing work or personal applications on the web from any device.|Password-less experience for workers using biometrics, PIN, and NFC.<br>Applicable for shared PCs and where a mobile phone is not a viable option (such as for help desk personnel, public kiosk, or hospital team)|

Use the following table to choose which method will support your requirements and users.

|Persona|Scenario|Environment|Passwordless technology|
|:-|:-|:-|:-|
|**Admin**|Secure access to a device for management tasks|Assigned Windows 10 device|Windows Hello for Business and/or FIDO2 security key|
|**Admin**|Management tasks on non-Windows devices| Mobile or non-windows device|Passwordless sign-in with the  Authenticator app|
|**Information worker**|Productivity work|Assigned Windows 10 device|Windows Hello for Business and/or FIDO2 security key|
|**Information worker**|Productivity work| Mobile or non-windows device|Passwordless sign-in with the Authenticator app|
|**Frontline worker**|Kiosks in a factory, plant, retail, or data entry|Shared Windows 10 devices|FIDO2 Security keys|

## Next steps

To get started with passwordless in Microsoft Entra ID, complete one of the following how-tos:

* [Enable FIDO2 security key passwordless sign-in](howto-authentication-passwordless-security-key.md)
* [Enable phone-based passwordless sign-in with the Authenticator app](howto-authentication-passwordless-phone.md)

### External Links

* [FIDO Alliance](https://fidoalliance.org/)
* [FIDO2 CTAP specification](https://fidoalliance.org/specs/fido-v2.0-id-20180227/fido-client-to-authenticator-protocol-v2.0-id-20180227.html)
