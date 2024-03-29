# Azure Managed HSM TLS Offload Library  
Azure Managed HSM offers a TLS Offload library which is compliant with PKCS#11 version 2.40. We do not support all possible functions listed in the PKCS#11 specification. Our TLS Offload library supports a limited set of mechanisms and interface functions for SSL/TLS Offload with F5 (BigIP) and Nginx only, primarily to generate TLS server certificate keys and generate digital signatures during TLS handshakes.

The TLS Offload Library internally uses the Azure Key Vault REST API to interact with Azure Managed HSM.

Installation including configuration and authentication requirements for TLS Offload can be found in the readme file within each .deb and .rpm release package under *[RELEASES](https://github.com/microsoft/AzureManagedHsmTLSOffload/releases)* 

You can find here the list of *[TLS Offload supported Key Types & Mechanisms](https://github.com/microsoft/AzureManagedHsmTLSOffload#supported-key-types--mechanisms)*

# Get Started 
### PKCS#11 Attributes
To properly integrate with PKCS#11, generating keys (via C_GenerateKeyPair) and locating key objects (via C_FindObjectsInit/C_FindObjects) requires a solution for storing PKCS#11 attributes on the Azure Key Vault key object. The TLS Offload Library converts these necessary PKCS#11 attributes into Azure Key Vault Tags.

These “Attribute Tags” have a special prefix:
-	p11_pri_{P11 Attribute Name} - Private Key Attributes
-	p11_pub_{P11 Attribute Name} - Public Key Attributes

**Note:** The TLS Offload library properly sets the Azure Key Vault Key Operations and Key Lifetime attributes so the service can properly enforce these restrictions on the generated keys. These attributes are also stored as tags like other PKCS#11 attributes to support query capabilities.  

Applications that use the TLS Offload Library use one or more PKCS#11 attributes to locate and use the key objects.

**Warning:** Keys generated by the TLS Offload Library and their Tags are accessible over Azure Key Vault REST API. Manipulating these P11 Attribute Tags using Azure Key Vault REST API may break the TLS Offload Library applications.

### Key Generation   
The TLS Offload Library includes a key creation tool -  mhsm_p11_create_key. Running the tool without any command line arguments shows the correct usage of the tool.

The key creation tool requires a Service Principal which is assigned to the “Managed HSM Crypto User” role at the “/keys” scope.

The key creation tool reads the Service Principal credentials from the environment variables MHSM_CLIENT_ID and MHSM_CLIENT_SECRET.
-	MHSM_CLIENT_ID – must be set to the Service Principal’s Application (Client) ID
-	MHSM_CLIENT_SECRET – must be set to the Service Principal’s Password (Client Secret)

For managed identities, these environment variables are not needed.  
- Use the “--identity” argument to enable managed identity with mhsm_p11_create_key tool.  
- The “client_id” of user-assigned managed identity should be mentioned in mhsm configuration file (mhsm-pkcs11.conf). If “client_id” of user-assigned managed identity is not provided it will consider it as system-assigned managed identity. 

The key creation tool randomly generates a name for the key at the time of creation. The full Azure Key Vault Key ID and the Key Name are printed to the console for your convenience.

> MHSM_CLIENT_ID="Service Principal Application Id" \\\
>    MHSM_CLIENT_SECRET="Service Principal Password" \\\
>    mhsm_p11_create_key --RSA 4K --label tlsKey
>
> Key is generated successfully. \
> Managed HSM Key ID: https://myhsm.managedhsm.azure.net/keys/p11-6a2155dc40c94367a0f97ab452dc216f/92f8aa2f1e2f4dc1be334c09a2639908 \
> Key Name: p11-6a2155dc40c94367a0f97ab452dc216f
 
The --label argument to the key creation tool specifies the desired CKA_LABEL for the private and public keys generated. These attributes are typically required to configure supported TLS Offload solutions (e.g., the nginx SSL configuration setting `ssl_certificate_key’).

You’ll need the key name for any role assignment changes via the Azure CLI.

### Access Control
The TLS Offload Library translates the C_FindObjectsInit into an Azure Key Vault REST API call which operates at the /keys scope. The MHSM service will require the Read permission at this scope for the TLS Offload Library User to authorize the find operation for the keys created via the key creation tool.

Refer to the following documentation for more information on Azure Managed HSM local RBAC.
- *[Azure Managed HSM local RBAC built-in roles](https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/built-in-roles)*
- *[Azure Managed HSM role management](https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/built-in-roles)*

_The following section below describes different approaches to implement access control for the TLS Offload Library Service Principal. Managed identities are under TLS Offload Library Service Principal._

#### TLS Offload Service Principal
Service Principal used by the application that uses TLS Offload Library to access keys. This Service Principal should have at minimum the following permission via role assignments:
- KeyRead permission to all the keys in the Managed HSM
-	KeySign permission to the keys necessary for TLS offloading

#### Admin User
Admin User will be creating a custom role definition and role assignments. Hence the Admin User should be assigned to one of the following Built-in roles at the “/” scope.
- Managed HSM Crypto Officer
- Managed HSM Policy Administrator
- Managed HSM Administrator

#### Key Generation Service Principal 
Service Principal that will be used with the key creation tool (mhsm_p11_create_key) to generate TLS offload keys. This Service Principal should be assigned to the “Managed HSM Crypto User” role at the “/keys” scope. 

#### Azure CLI
In the description below, Azure CLI is used to perform tasks like Role Assignment, etc. 

### Permissive Approach
This is a simpler approach and suitable when the Azure Managed HSM is exclusively used for TLS offloading.

Assign the Crypto User role to TLS Offload Service Principal at the “/keys” scope. This gives the TLS Offload Service Principal the permission to generate keys and find them for TLS Offloading.
 
> az keyvault role assignment create --hsm-name ContosoMHSM \\\
>      --role "Managed HSM Crypto User"  \\\
>       --assignee TLSOffloadServicePrincipal@contoso.com  \\\
>       --scope /keys

For managed identities specify command arguments as follows. 
> az keyvault role assignment create --hsm-name ContosoMHSM \\\
>      --role "Managed HSM Crypto User"  \\\
>      --assignee-object-id <object_id> \\\
>      --assignee-principal-type MSI  \\\
>      --scope /keys

### Granular Approach
The granular approach implements fine grained access control and requires two Service Principals (TLS Offload Service Principal and Key Generation Service Principal), and an Admin User. 

The objective is to restrict the TLS Offload Service Principal’s permissions to support the minimum required for TLS offload. Concretely this requires the user to have the Read permission for other keys to support the library’s C_FindObject* function.

#### TLS Offload Library User Read Role
The first step in implementing the granular approach requires creating a custom role. This is a one-time operation.

The Admin User (with Managed HSM Crypto Officer or Managed HSM Administrator or Managed HSM Policy Administrator role) creates a custom “TLS Library User Read Role” role definition:
  
> az keyvault role definition create --hsm-name ContosoMHSM --role-definition '{ \
>      "roleName": "TLS Library User Read Role", \
>      "description": "Grant Read access to keys", \
>      "actions": [], \
>      "notActions": [], \
>      "dataActions": ["Microsoft.KeyVault/managedHsm/keys/read/action"], \
>      "notDataActions": [] \
> }'

#### Generate Keys
Keys can be generated using the Key Generation Service Principal with the key creation tool (mhsm_p11_create_key).

#### Grant Permission
The Admin User assigns the following roles to the TLS Offload Service Principal.
- Assign “TLS Library User Read Role” role at the “/keys” scope
- Assign “Managed HSM Crypto User” role at the “/keys/{key name}” scope

In the following example, the key name is “p11-6a2155dc40c94367a0f97ab452dc216f”.
   
> az keyvault role assignment create --hsm-name ContosoMHSM  \\\
>      --role "TLS Library User Read Role"  \\\
>      --assignee TLSOffloadServicePrincipal@contoso.com  \\\
>      --scope /keys
>
> az keyvault role assignment create --hsm-name ContosoMHSM  \\\
>      --role "Managed HSM Crypto User"  \\\
>      --assignee TLSOffloadServicePrincipal@contoso.com  \\\
>      --scope /keys/p11-6a2155dc40c94367a0f97ab452dc216f

### Connection Caching
To improve the performance of Sign calls to the Managed HSM Service, TLS Offload Library caches its TLS connections to the Managed HSM service servers. By default, TLS Offload Library caches up to 20 TLS connections. 

Connection Caching can be controlled through MHSM configuration file (mhsm-pkcs11.conf).  
> "ConnectionCache": { \
> "Disable": false, \
> "MaxConnections": 20 \
> } 

#### Disable 
If this value is true, Connection Caching will be disabled. It is enabled by default. 

#### MaxConnections 
Specifies maximum number of connections to cache. The maximum connection limit should be configured based on the number of concurrent PKCS11 sessions being used by the application. Applications typically create a pool of PKCS11 sessions and use them from a pool of threads to generate signing requests in parallel. The MaxConnections should match the number of concurrent signing requests generated by the applications. 

The Signing Request Per Second (RPS) is dependent on the number of concurrent requests and the number of connections cached. Specifying a higher number or even the default limit will not improve the signing RPS if the number of concurrent PKCS11 Signing requests is lower than this limit. 

The maximum number of concurrent connections to achieve burst mode of Standard B1 HSM pool is about 30 depending on the instance type. But you should try with different numbers to figure out the optimal number concurrent connections. 

Refer to your application documentation or contact your application vendor to learn more about how the application uses the PKCS11 library. 

# How To
### How to generate keys using the TLS Offload Library?
The TLS Offload Library includes a key creation tool -  mhsm_p11_create_key. Running the tool without any command line arguments shows the correct usage of the tool.

The key creation tool requires a Service Principal which is assigned to the “Managed HSM Crypto User” role at the “/keys” scope.

The key creation tool reads the Service Principal credentials from the environment variables MHSM_CLIENT_ID and MHSM_CLIENT_SECRET.
- MHSM_CLIENT_ID – must be set to the Service Principal’s Application (Client) ID
- MHSM_CLIENT_SECRET – must be set to the Service Principal’s Password (Client Secret)

For managed identities, these environment variables are not needed.  
- Use the “--identity” argument to enable managed identity with mhsm_p11_create_key tool. 
- The  “client_id” of user-assigned managed identity should be mentioned in mhsm configuration file (mhsm-pkcs11.conf). If “client_id” of user-assigned managed identity is not provided it will consider it as system-assigned managed identity. 

The key creation tool randomly generates a name for the key at the time of creation. The full Azure Key Vault Key ID and the Key Name are printed to the console for your convenience.
  
> MHSM_CLIENT_ID="Service Principal Application Id" \\\
>    MHSM_CLIENT_SECRET="Service Principal Password" \\\
>    mhsm_p11_create_key --RSA 4K --label tlsKey
>
> Key is generated successfully. \
> Managed HSM Key ID: https://myhsm.managedhsm.azure.net/keys/p11-6a2155dc40c94367a0f97ab452dc216f/92f8aa2f1e2f4dc1be334c09a2639908 \
> Key Name: p11-6a2155dc40c94367a0f97ab452dc216f

The --label argument to the key creation tool specifies the desired CKA_LABEL for the private and public keys generated. These attributes are typically required to configure supported TLS Offload solutions (e.g., the nginx SSL configuration setting `ssl_certificate_key’).

The key name is required if you are planning to implement granular access to keys.

### How to implement Key Less TLS?
There are two approaches to generating a key and using the key for the Key Less TLS. The approaches differ in implementation effort and security enforcement.
- Simpler, more permissive approach
- Granular which offers better security

#### Simpler Permissive Approach
1.	Create a Service Principal for the TLS Offload Library (e.g., TLSOffload ServicePrincipal)
2.	Assign “Managed HSM Crypto User” role to the TLS Offload Service Principal at the “/keys” scope.
 
> az keyvault role assignment create --hsm-name ContosoMHSM \\\
>      --role "Managed HSM Crypto User"  \\\
>       --assignee TLSOffloadServicePrincipal@contoso.com  \\\
>       --scope /keys

3.	Generate key with required label following How to generate keys using the TLS Offload Library.  
4.	Configure the TLS server to use the Managed HSM TLS Offload Library as the PKCS#11 interface library
5.	Configure the TLS server (e.g., the nginx SSL configuration setting `ssl_certificate_key’) with the key label and the TLS Offload Service Principal credentials. For MSI (managed service identity) use empty credentials or enable it via TLS offload mhsm configuration file (mhsm-pkcs11.conf) and “client_id” of user-assigned managed identities. If MSI is enabled via TLS offload mhsm configuration file (mhsm-pkcs11.conf), then the Service principal credentials will be ignored.

#### Granular Approach 
1.	Create an Admin User (e.g., TLSOffloadAdminUser) with the following role:
a.	“Managed HSM Crypto Officer” role at the “/” scope
2.	Create a Key Generation Service Principal (e.g., TLSOffloadKeyGenServicePrincipal) for the TLS Offload Key generation and assign the following role:
a.	“Managed HSM Crypto User” role at the “/keys” scope.
3.	Create a Service Principal for the TLS Offloading (e.g., TLSOffload   ServicePrincipal)
4.	The Admin User creates the following custom role definition:
  
> az keyvault role definition create --hsm-name ContosoMHSM --role-definition '{ \
>      "roleName": "TLS Library User Read Role", \
>      "description": "Grant Read access to keys", \
>      "actions": [], \
>      "notActions": [], \
>      "dataActions": ["Microsoft.KeyVault/managedHsm/keys/read/action"], \
>      "notDataActions": []
> }'

5.	Generate key with required label following "How to generate keys using the TLS Offload Library". Use the Key Generation Service Principal (e.g., TLSOffloadKeyGenServicePrincipal) created above while generating keys. Note down the Key Label and Key Name. E.g.,
  a.	Key Label: tlsKey
  b.	Key Name: p11-6a2155dc40c94367a0f97ab452dc216f  
6.	Admin User assigns the following roles to the TLS Offload Service Principal
  a.	“TLS Library User Read Role” role at the “/keys” scope
  b.	“Managed HSM Crypto User” role at the “/keys/{key name}” scope
  
> az keyvault role assignment create --hsm-name ContosoMHSM  \\\
>      --role " TLS Library User Read Role"  \\\
>      --assignee TLSOffloadServicePrincipal @contoso.com  \\\
>      --scope /keys
>
> az keyvault role assignment create --hsm-name ContosoMHSM  \\\
>      --role "Managed HSM Crypto User"  \\\
>      --assignee TLSOffloadServicePrincipal@contoso.com  \\\
>      --scope /keys/p11-6a2155dc40c94367a0f97ab452dc216f

7.	Configure the TLS server to use the Managed HSM TLS Offload Library as the PKCS#11 interface library
8.	Configure the TLS server (e.g., the nginx SSL configuration setting `ssl_certificate_key’) with the key label and the TLS Offload Service Principal credentials. For MSI (managed service identity) use empty credentials or enable it via TLS offload mhsm configuration file (mhsm-pkcs11.conf) and “client_id” of user-assigned managed identities. If MSI is enabled via TLS offload mhsm configuration file (mhsm-pkcs11.conf), then the Service principal credentials will be ignored.


