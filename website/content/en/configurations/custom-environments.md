---
type: docs
title: "Custom Azure Environments"
linkTitle: "Custom Azure Environments"
weight: 5
description: >
  Pull secret content from KeyVault instances hosted on air-gapped and/or on-prem Azure clouds
---

In order to pull secret content from Keyvault instances hosted on air-gapped and/or on-prem Azure clouds, there are two steps needed

1. Mount the Custom Cloud Environment file to the Azure KeyVault Provider Pods
2. Configure the Secret Provider Class

## Mount Custom Cloud Environment File

The Custom Cloud Environment file is a JSON file that contains the custom cloud environment details that [azure-sdk-for-go](https://github.com/Azure/azure-sdk-for-go) needs to interact with the target Keyvault instance. Typically, the custom cloud environment file is stored in the file system of the Kubernetes node and made accessible to the Azure Key Vault provider pods through a mounted volume.

If you are installing the Azure KeyVault Provider via Helm charts, set the following values to mount the Environment File

- `linux.volumes` / `windows.volumes` - A volume that contains the custom cloud environment file
- `linux.volumeMounts` / `windows.volumeMounts` - A volume mount allowing the KeyVault provider pod to access the custom cloud environment file

Example:

```yaml
linux:
  volumes:
    - name: cloudenvfile-vol
      hostPath:
        path: "/etc/kubernetes"
    - name: sslcerts
      hostPath:
        path: "/etc/ssl/certs"
  volumeMounts:
    - name: cloudenvfile-vol
      mountPath: "/cloudEnv/myCustomEnvironmentFile.json"
      subPath: "myCustomEnvironmentFile.json"
    - name: sslcerts
      mountPath: "/etc/ssl/certs"
      readOnly: true
```

## Update Secret Provider class

The `SecretProviderClass` resource must include the following:

```yaml
parameters:
  cloudName: "AzureStackCloud"
  cloudEnvFileName: "/path/to/custom/environment.json"
```

The `cloudEnvFileName` parameter should match the volumeMount that was configured in the previous step.

Even if the target cloud is not an Azure Stack Hub cloud, cloud name must be set to `"AzureStackCloud"` to signal `azure-sdk-for-go` to load the custom cloud environment details from `cloudEnvFileName`.

If the target cloud's identity provider system is [AD FS][adfs] (instead of Azure AD), then the `tenantId` property in `SecretProviderClass` should be set to `"adfs"`.


```yaml
parameters:
  cloudName: "AzureStackCloud"
  cloudEnvFileName: "/path/to/custom/environment.json"
  tenantId: "adfs"
```

## Environment files

The custom cloud environment sample below shows the minimum set of properties required:

```json
{
  "name": "AzureStackCloud",
  "activeDirectoryEndpoint": "https://login.microsoftonline.com/",
  "keyVaultEndpoint": "https://vault.azure.net/",
  "keyVaultDNSSuffix": "vault.azure.net"
}
```

### Azure Stack Hub Environment Files

The environment file for most ARM-based Azure clouds can be generated by using as input the target cloud metadata. The following script shows how to generate the environment file for Azure Stack Hub clouds (both Azure AD and AD FS deployments).

> Learn more about Azure Stack Hub's fully qualified domain names (FQDN) [here][ash-dns].

```bash
curl -s https://management.${FQDN}/metadata/endpoints?api-version=1.0 -o cloudMeta.json

AD_EP=$(jq -r .authentication.loginEndpoint cloudMeta.json | sed -e 's|adfs$||1')
KV_EP=$(jq -r .authentication.audiences[0] cloudMeta.json | sed "s|management.|vault.|1")
KV_DNS=vault.${FQDN}

cat << EOF
{
  "name": "AzureStackCloud",
  "activeDirectoryEndpoint": "${AD_EP}",
  "keyVaultEndpoint": "${KV_EP}",
  "keyVaultDNSSuffix": "${KV_DNS}"
}
EOF
```

[adfs]: https://learn.microsoft.com/windows-server/identity/active-directory-federation-services
[ash-dns]: https://learn.microsoft.com/azure-stack/operator/azure-stack-integrate-dns?#azure-stack-hub-dns-namespace
