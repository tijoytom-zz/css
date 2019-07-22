
# 1) Enable Central Secret Store Service Fabric Azure Cluster.
## Upgrade/deploy the cluster with this added to cluster manifest.

```json
               "fabricSettings": [
                    ...
                    {
                        "parameters": [
                            {
                                "name": "IsEnabled",
                                "value": "true"
                              },
                              {
                                "name": "MinReplicaSetSize",
                                "value": "1"
                              },
                              {
                                "name": "TargetReplicaSetSize",
                                "value": "1"
                              },
                              {
                                  "name" : "EncryptionCertificateThumbprint",
                                  "value": "9d9f39a1e563a537991fab9d2f557f8e04495598"
                              },
                        ],
                        "name": "CentralSecretService"
                    }
```
## Make sure to provision the encryption certificate to VMs while depoying the cluster.
```json
                    "osProfile": {
                        ....
                        "secrets": [
                            {
                                "sourceVault": {
                                    "id": "keyVault"
                                },
                                "vaultCertificates": [
                                   ...
                                    {
                                        "certificateStore": "My",
                                        "certificateUrl": "EncryptionCertificateURL"
                                    },
                                ]
                            }
                        ]
                    },
```
## Make sure 'NetworkService' has read permission to encryption certificate private key.
  Once you login in to the VM run `certutil -v  -store My <EncryptionCertificateThumbprint>` and see of NetworkService has read permission. If not, you can use the below script to 
  grant permission. 
  ```powershell
function Add-CertAcl($cert) {
        $rsaFile = $cert.PrivateKey.CspKeyContainerInfo.UniqueKeyContainerName
        $keyPath = "C:\ProgramData\Microsoft\Crypto\RSA\MachineKeys\"
        $fullPath=$keyPath+$rsaFile
        $oldAcl = get-acl $fullPath
        $grantee_name = 'networkservice'
        $grantee = New-Object System.Security.Principal.NTAccount($grantee_name)
        $accessRule = New-Object System.Security.AccessControl.FileSystemAccessRule($grantee, 'Read', 'None', 'None', 'Allow')
        $oldAcl.AddAccessRule($accessRule)
        $newAcl = Set-Acl $fullPath $oldAcl
        get-acl $fullPath
}
$cert = Get-ChildItem 'cert:\LocalMachine\My\9d9f39a1e563a537991fab9d2f557f8e04495598'
Add-CertAcl($cert)
```
Once CSS(Central Secret Store) is enabled you can follow the instructions for local cluster below to set and read secret values.

## Declare secret resources in ARM template.
  You can declare secret resource in your ARM template and later use the REST API to set values for the secret resource. 
```json 
"resources": [
    {
      "apiVersion": "2018-07-01-preview",
      "name": "supersecret",
      "type": "Microsoft.ServiceFabricMesh/secrets",
      "location": "[parameters('location')]", 
      "dependsOn": [],
      "properties": {
        "kind": "inlinedValue",
        "description": "Application Secret",
        "contentType": "text/plain",
      }
    }
  ]
  ```
# 2) Enable Secret Store for local cluster.
Note : Inorder to use secret store the cluster need to be a secure cluster.
## Upgrade/deploy the cluster with this added to cluster manifest.
```xml
<FabricSettings>
    <Section Name="ApplicationGateway/Http">
     ...
    </Section>
    <Section Name="CentralSecretService">
      <Parameter Name="IsEnabled" Value="true" />
      <Parameter Name="MinReplicaSetSize" Value="1" />
      <Parameter Name="TargetReplicaSetSize" Value="1" />
      <!-- if EncryptionCertificateThumbprint is left out ClusterCertificate will be used and is not recommended.-->
      <Parameter Name="EncryptionCertificateThumbprint" Value="THUMBPRINT" />
    </Section>
</FabricSettings>
``` 
After deploy you should see `fabric:/System/CentralSecretService` is listed under `System` services in fabric explorer.  

## Create a secret resource.
*From here on we will use **`PASSWORD`** as our secret resource name and **`v1`** as it's version.*
PUT https://localhost:19080/Resources/Secrets/PASSWORD?api-version=6.4-preview
```json
    {  
   "properties":{  
      "kind":"simple",
      "version":"v1",
      "contentType":"text/plain",
      "description":"database connection string"
   }
}
```

## Set value for your secret
PUT https://localhost:19080/Resources/Secrets/PASSWORD/values/v1?api-version=6.4-preview
```json
{"properties": {"value": "mysecretpassword"}}
```

## View the value
POST https://localhost:19080/Resources/Secrets/PASSWORD/values/v1/list_value?api-version=6.4-preview

## Use the secret in your application.
### settings.xml
```xml 
<?xml version="1.0" encoding="utf-8" ?>
<Settings xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.microsoft.com/2011/01/fabric">
  <Section Name="testsecrets">
  <Parameter Name="TopSecret" Type="SecretsStoreRef" Value="PASSWORD:v1"/
  </Section>
</Settings>
```
### ApplicationManifest.xml
```xml
<ServiceManifestImport>
    <ServiceManifestRef ServiceManifestName="testservicePkg" ServiceManifestVersion="1.0.0" />
    <ConfigOverrides />
    <Policies>
      <ConfigPackagePolicies CodePackageRef="Code">
        <ConfigPackage Name="Config" SectionName="testsecrets" EnvironmentVariableName="SecretPath" />
      </ConfigPackagePolicies>
    </Policies>
  </ServiceManifestImport>
```

### main.go
```go
cstr, err := ioutil.ReadFile(path.Join(os.Getenv("SecretPath"),  "TopSecret"))
```

### Mounting the secrets to containers.

```xml
<ServiceManifestImport>
    <ServiceManifestRef ServiceManifestName="testservicePkg" ServiceManifestVersion="1.0.0" />
    <ConfigOverrides />
    <Policies>
      <ConfigPackagePolicies CodePackageRef="Code">
        <ConfigPackage Name="Config" SectionName="testsecrets" MountPoint = "C:\myapp\secrets" />
      </ConfigPackagePolicies>
    </Policies>
  </ServiceManifestImport>
```
C:\myapp\secrets folder inside the container will have files with <Parameter> names. in this case `C:\myapp\secrets\TopSecret`

### Using a secret in <ContainerHostPolicies> for accessing container repository

```xml
<ServiceManifestImport>
    <ServiceManifestRef ServiceManifestName="containertestPkg" ServiceManifestVersion="1.0.0" />
    <ConfigOverrides />
    <Policies>
      <ContainerHostPolicies CodePackageRef="Code">
        <RepositoryCredentials AccountName="tijoytom" Type="SecretsStoreRef" Password="PASSWORD:v1"/>
       ...
  </ServiceManifestImport>
```

# 3) Using Service Fabric CLI to manage secrets.
  ### Install [sfctl](https://github.com/microsoft) service-fabric-cli.

  ### 3.1) Managing secret resources. 
    `sfctl mesh secret [show, delete, list]
  ### 3.2) Managing secret resource value.
    `sfctl mesh secretvalue [delete, list, show] 
  ### Examples:
  Show value for a secret: `sfctl mesh secretvalue show -n PASSWORD -v v1`
  List all secrets :`sfctl mesh secret list` 

# FAQ 
  1. Secrets are a cluster level resource and are shared among all the application in the cluster. 
  
  2. Secrets are versioned, ie secret PASSWORD can have multiple versioned values, for example PASSWORD:v1 = "abc", PASSWORD:v2 = "vfg" , where PASSWORD is the secret name and v1 and v2 are versions.
  
  3. `sfctl` do not support creating a secret, secrets are created declaratively using the ARM template or using the REST api.
  
  4. 
