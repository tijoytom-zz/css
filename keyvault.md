# Secret Store Keyvault support.

## 1. Current SF mesh secret store support.
Before talking about keyvault support let's look at how mesh secret store is used.
* Declare your secrets in cluster manifest.
    ```json 
        "resources":[
            "type" : "Microsoft.ServiceFabricMesh/secrets",
             ..
             "properties" :{
                 "kind" : "inlinedvalue",
                 ..
             }
        ]
    ```
* Setup the secret value.
    ```bash
    sfctl mesh secretvalue --secretname "supersecret" --version "version1" "value1"
    ```
* Reference it in your application 'settings.xml'
    ```xml
    <Parameter Name="TopSecret2" Type="SecretStoreRef" Value="supersecret:version1"/>
    ```
Couple of points to note.
* Secrets are resources and they are versioned just like any other resources. 
* Secrets are cluster level resources and any application running in the cluster have access to it.
* Secrets are encrypted using secret encryption certificate specified in cluster manifest, if no encryption certificate 
  is specified we fallback to cluster certificate.

## 2. Keyvault support 
The goal is to extend secret store to support keyvault references. Importantly we need to scope keyvault references to 
applciationtype/service in a (multi-tenant) cluster. Current secret store secrets are cluster level resources and there 
has been discussion about removing secerts as a resource, but this means undoing/redoing secret store, so in order to 
limit the scope we will re-use most of the existing interfaces and modify CSS to support keyvault. 

Let's start with how an Atlas application will use keyvault reference.

Add this to settings.xml and hosting will automatically resolve `TopSecret2` keyvault reference
```xml
<Parameter Name="TopSecret2" Type="KeyVaultRef" Value="https://ttkvault.vault.azure.net/secrets/supersecret/8f642b17bf95453a9fa611925a9c3c89"/>
```
As you might have noticed, the user no longer has to create a secret resource neither do they have to setup value using 
another commandline tool. `KeyVaultRef` is the new type that hosting will understand and `Value` is a valid KeyVaultURL.

## 3. Under the Hood
### Prerequisites for using keyvault references.
    * CentralSecretStore service is enabled.
    * Applciation is using Managed Identity and the identity have read permission to the keyvault.
    * ManagedIdentityToken service is enabled.
### Workflow
    * User defines parameters with Type='KeyVaultRef' and Value='<keyvaulturl>' in settings/applicationmanifest
    * Hosting activation code path see the keyvault/secretstore references. ie Type='KeyVaultRef' or Type='SecretStoreRef'
    * Hosting gather all secret store references and makes a request to CSS(Central Secert Store) to resolve them.
    * CSS resolves the known secrets values, but for unknown keyvault references it does the below.
        * All secerts resolved, do nothing.
        * One of the secret with Type='SecretStoreRef' then fail. [These are secretstore resources 
          and they need be created upfront]
        * One of the secret with Type='KeyVaultRef' is 'NOTFOUND', proceed to resolve them.

    * CSS keyvault secret resolution 
        * HTTP(S) get for an access token for keyvault resource from ManagedIdentityService.
        * HTTP(S) get for the keyvault URL(HTTP Header `Authorization Bearer <token>`)
        * Fails if can't resolve keyvault secret
        * Create a secret resource corresponsing to the resolved keyvault secret.(think of this as cache with scoping and encryption)
        * secret resource key is computed as SHA256 of {ParameterName,ApplicationType,ServiceName,KeyVaultURL}
        * Set value for the newly created secret resource.
        * Any subsequent request for the same KeyVault secret will be served from CSS
    * Hosting gets the results back and check if all secrets are resolved, if yes proceed with activation else fail.

![Workflow](https://github.com/tijoytom/css/blob/master/kvault.png)

### Advantages
    * Minor changes for Central Secret Store.
    * Simple and straight forward usage.
    * Keyvault secrets scoped to application and service using SHA256 secret key.
    * No major change required for Atlas applciation model(as far as i know, Atlas people comment?).
    * Code changes are not exhaustive and works well with existing secret store implemenation.

### Disadvantages
    * This is not bullet proof, any one how know applicationtype and servicename can compute the secret key. To an 
      extend mitigated by filtering our all KeyVault secret type from public API.
    * CSS need to know about keyvault references, but that's it job anyways.
    * Remote possibility that two totally different cluster might have the same applicationtype/service/keyvaulurl/parametername. 
      Consider adding something that identify the subscription to the hash?

# FAQ
 
    1. How do i refresh/version the keyvault secret?
    Keyvault create different URL for different versions of the same secret, so once the applciation developer change 
    the URL in settings.xml, the key that we generate will be different(hopefully no SHA256 collision) and hosting will 
    automatically retrive the new secret from keyvault.

    2.Atlas specific changes?
    I dont see any changes specific to Atlas other than the new Parameter Type=`KeyVaultRef`. I might be missing 
    something here.
    
    3. How do we handle burst traffic for keyvault ie 100s of containers spinning up at the same time?
    This is mitigated in two ways, make sure not to make multiple request to keyvault for the same URL and also check CSS 
    if the value is already retrived by another node before making a request. 
    
    4. Do we need new internal API for secret store?
    The current design do not require any new API other than slight modification to behaviour of existing API. But we 
    might need new API in future if was want to lock down secret store to admins only.(dragos mentioned this)
    
    5. What about mounting to container, binding to envirnoment variable etc?
    All this work exactly as it is today for secret store, that's another advantage of treating secret store as a blackbox.
    
    6. What about provisioning secrets in keyvault and granting permission etc?
    The only thing that we care about is the application identity have read access to the keyvault secret, how the 
    admin set it up is up to them and totally outside the scope of SF cluster.



