# Introduction 
Azure Managed HSM offers a TLS Offload library which is compliant with PKCS#11 version 2.40. We do not support all possible functions listed in the PKCS#11 specification. PKCS#11 is a standard for performing cryptographic operations on HSMs (hardware security modules) which is available in public preview.

# Getting Started
Azure Managed HSM supports PKCS#11 mechanisms and functions for SSL/TLS Offload with F5 (BigIP) and Nginx only!

Installation including configuration and authentication requirements for TLS Offload can be found under *[RELEASES](https://github.com/microsoft/AzureManagedHsmTLSOffload/releases)* 

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
| AES | Generate 128, 192, and 256-bit AES keys. |

#### Mechanisms 
Azure Managed HSM TLS Offload supports the following algorithms. 

| **Mechanisms** | **Description** |
|-----------|-----------|
| Encryption and Decryption | RSA-OAEP, and RSA-PKCS supported. AES-CBC not supported through TLS Offload library. | 
| Sign and Verify | RSA, and ECDSA supported. SignRecover/VerifyRecover not supported. |
| Hash/Digest | SHA256, SHA384, and SHA512 supported. |
| Key Wrap | Not Supported through TLS Offload library. Key Wrap is supported through Managed HSM API only. |
| Triple Des (3DES) | Not Supported |
| Key Derivation | Not Supported |

Azure Managed HSM TLS Offload is compliant with version 2.40 of the PKCS#11 specification. To invoke a cryptographic feature using our TLS Offload library, call a function with a given mechanism. The following table summarizes the combinations of functions and mechanisms supported by Azure Managed HSM.  

X indicates that Azure Managed HSM supports the mechanism for the function. We do not support all possible functions listed in the PKCS#11 specification. 

#### Mechanisms & Functions
| |  **Encrypt & Decrypt** | **Sign  & Verify** | **SR & VR** | **Digest** | **Gen Key/Key Pair** | **Wrap & Unwrap** | **Derive**|
|-----------|-----------|-----------|-----------|-----------|-----------|-----------|-----------|
| CKM_RSA_PKCS_KEY_PAIR_GEN  | | | | |X| | |
| CKM_RSA_PKCS               |X|X| | | | | |
| CKM_RSA_PKCS_OAEP          |X| | | | | | |
| CKM_RSA_PKCS_PSS           | |X| | | | | |
| CKM_EC_KEY_PAIR_GEN        | | | | |X| | |
| CKM_ECDSA                  | |X| | | | | |
| CKM_AES_KEY_GEN            | | | | |X| | |
| CKM_SHA256                 | | | |X| | | |
| CKM_SHA384                 | | | |X| | | |
| CKM_SHA512                 | | | |X| | | |

***
## Supported API Operations 
Azure Managed HSM TLS Offload supports the following PKCS#11 API operations. 
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
- C_GenerateKey 
- C_GetAttributeValue 
- C_SetAttributeValue 
- C_GetOperationState 
- C_SetOperationState 
- C_SignInit 
- C_Sign 
- C_VerifyInit 
- C_Verify 
- C_EncryptInit 
- C_Encrypt 
- C_DecryptInit 
- C_Decrypt 
***

# FAQ
### Can I create a Key in Azure Managed HSM then later use the TLS Offload library for that Key?
No. Keys created without using the mhsm-pkcs11 TLS Offload Library are NOT compatible. A key must be created using either the mhsm_p11_create_key sample or a custom application that loads the mhsm-pkcs11 TLS Offload library and calls the appropriate PKCS#11 interface functions.

## Trademarks
This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft 
trademarks or logos is subject to and must follow 
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.
