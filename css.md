
# Enable Secret Store for local cluster.
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

*From here on we will use **`connstring`** as our secret resource name and **`v1`** as it's version.*
## Create a secret resource.
PUT https://localhost:19080/Resources/Secrets/connstring?api-version=6.4-preview
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
PUT https://localhost:19080/Resources/Secrets/connstring/values/v1?api-version=6.4-preview
```json
{"properties": {"value": "conn=bla;password=bla"}}
```

## View the value
POST https://localhost:19080/Resources/Secrets/connstring/values/v1/list_value?api-version=6.4-preview

## Use the secret in your application.
### settings.xml
```xml 
<?xml version="1.0" encoding="utf-8" ?>
<Settings xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.microsoft.com/2011/01/fabric">
  <Section Name="testsecrets">
  <Parameter Name="TopSecret" Type="SecretsStoreRef" Value="connstring:v1"/
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

