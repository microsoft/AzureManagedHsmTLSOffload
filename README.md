# Introduction 
Azure Managed HSM offers a TLS Offload library which is compliant with PKCS#11 version 2.40. We do not support all possible functions listed in the PKCS#11 specification. Our TLS Offload library supports a limited set of functions and mechanisms for SSL/TLS Offload with F5 (BigIP) and Nginx only!

# Getting Started
Installation including configuration and authentication requirements for TLS Offload can be found in the readme file within each .deb and .rpm release package under *[RELEASES](https://github.com/microsoft/AzureManagedHsmTLSOffload/releases)* 

Additional information can be found from *[Get Started](https://github.com/microsoft/AzureManagedHsmTLSOffload/blob/main/GET_STARTED.md)*

## Supported Operating Systems

- CentOS 7
- Ubuntu 18.04

*CentOS 8 is not supported! Red Hat no longer supports CentOS 8 as it reached end of life December 31st, 2021.*
*** 

## Supported Key Types & Mechanisms
#### Key Types
Azure Managed HSM TLS Offload supports the following key types. 

| **Key Types** | **Description** |
|-----------|-----------|
| RSA | Generate 2048, 3072 and 4096-bit RSA keys. |
| ECDSA | Generate keys with P-256, P-256K, P-384, P-521 curves. |
| AES | Not Supported |

#### Mechanisms 
Azure Managed HSM TLS Offload supports the following algorithms. 

| **Mechanisms** | **Description** |
|-----------|-----------|
| Encryption and Decryption | Not supported through TLS Offload library. Supported through Managed HSM API only. | 
| Sign and Verify | RSA, and ECDSA supported. SignRecover/VerifyRecover not supported. |
| Hash/Digest | SHA256, SHA384, and SHA512 supported. |
| Key Wrap | Not Supported through TLS Offload library. Key Wrap is supported through Managed HSM API only. |
| Triple Des (3DES) | Not Supported |
| Key Derivation | Not Supported |

To invoke a cryptographic feature using our TLS Offload library, call a function with a given mechanism. The following table summarizes the combinations of functions and mechanisms supported by Azure Managed HSM.  

X indicates that Azure Managed HSM supports the mechanism for the function. We do not support all possible functions listed in the PKCS#11 specification. 

#### Mechanisms & Functions
| |  **Encrypt & Decrypt** | **Sign  & Verify** | **SR & VR** | **Digest** | **Gen Key/Key Pair** | **Wrap & Unwrap** | **Derive**|
|-----------|-----------|-----------|-----------|-----------|-----------|-----------|-----------|
| CKM_RSA_PKCS_KEY_PAIR_GEN  | | | | |X| | |
| CKM_RSA_PKCS               | |X| | | | | |
| CKM_RSA_PKCS_OAEP          | | | | | | | |
| CKM_RSA_PKCS_PSS           | |X| | | | | |
| CKM_EC_KEY_PAIR_GEN        | | | | |X| | |
| CKM_ECDSA                  | |X| | | | | |
| CKM_SHA256                 | | | |X| | | |
| CKM_SHA384                 | | | |X| | | |
| CKM_SHA512                 | | | |X| | | |

***
## Supported API Operations 
Azure Managed HSM TLS Offload supports the following API operations. 
- C_GetFunctionList 
- C_GetSlotList 
- C_GetSlotInfo 
- C_GetInfo 
- C_GetTokenInfo 
- C_GetSessionInfo 
- C_GetMechanismList 
- C_GetMechanismInfo 
- C_Initialize 
- C_Finalize 
- C_Login 
- C_Logout 
- C_OpenSession 
- C_CloseSession 
- C_CloseAllSessions 
- C_FindObjectsInit 
- C_FindObjects 
- C_FindObjectsFinal 
- C_GenerateKeyPair 
- C_GenerateRandom 
- C_GetAttributeValue 
- C_SetAttributeValue 
- C_SignInit 
- C_Sign 
- C_VerifyInit 
- C_Verify 
***

# FAQ
### Can I create a Key in Azure Managed HSM then later use the TLS Offload library for that Key?
No. Keys created without using the mhsm-pkcs11 TLS Offload Library are NOT compatible. A key must be created using either the mhsm_p11_create_key sample or a custom application that loads the mhsm-pkcs11 TLS Offload library and calls the appropriate interface functions.

### Does TLS Offload library support multiple Managed HSM resources?
Yes. You can declare multiple resources in the mhsm-pkcs.conf. SlotId is the identifier for the Managed HSM resource which is unique.

### Does TLS Offload library support multiple service principals?
No. Our TLS Offload library does not support multiple service principles.

### Does the TLS Offload Library support AES Keys? 
No. Our TLS Offload Library does not support AES keys. Customers that require AES keys should use the Azure Managed HSM REST API.

### Does the TLS Offload Library support TLS V1.0 or TLS 1.1?
No. We only support TLS 1.2 and TLS 1.3.

### Does the TLS Offload Library support Azure Key Vault and Azure Managed HSM?
No. Azure Key Vault is not supported.  Only Azure Managed HSM is supported through our TLS Offload Library.

### How do I file production support incidents or get help?
All production incident support tickets for Azure Managed HSM or TLS Offload Library should be submitted through the Azure Portal under Help+Support. This TLS Offload Library project uses GitHub issues to only track bugs and feature requests not production live site support incidents. 

- For help and questions about using this project for SSL Offloading / Keyless TLS with Azure Managed HSM please send email to [mhsmp11@microsoft.com](mailto:mhsmp11@microsoft.com).

- For help and questions about F5 BigIP or Nginx issues or configuration please send email to [support@f5.com](mailto:support@f5.com) or submit through case management at *[my.f5.com](https://my.f5.com)* 

## Trademarks
This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft 
trademarks or logos is subject to and must follow 
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.
