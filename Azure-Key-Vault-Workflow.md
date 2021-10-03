# HashiCorp Vault with Azure Key Vault Demo

## Prerequisites
- HashiCorp Vault >= 1.8
- `vault`
- `jq` 
- `base64`
- OpenSSL >= 3.0 (Run `openssl version` to validate)
- An Azure Key Vault instance.
- `ARM_TENANT_ID`, `ARM_CLIENT_ID` and `ARM_CLIENT_SECRET` environment variables set with permission on a Azure Key Vault instance.

## High level demo steps
Configuring the KMS Secrets Engine.
- Enabling the KMS Secrets Engine.
- Configure the KMS Secrets Engine with a Azure Key Vault backend.

Generate a demo key
- Generate a demo key.
- Inspect the demo key.
- Use the demo key from HashiCorp Vault to encrypt a value.

Example workflow using both HashiCorp Vault and Azure Key Vault
- Distribute the demo key to the Azure Key Vault backend.
- Use the Azure cli to decrypt a value.

Delete the key material from the cloud provider.
- Delete the key from Azure Key Vault.
- Validate that Azure Key Vault can no longer decrypt our value.
- Redistibute the same key material back to Azure Key Vault.

Rotate the key material stored in the cloud provider.
- Create a new version of the key.
- Disable the previous version of the key.

## Configuring the KMS Secrets Engine
Start a dev instance of HashiCorp Vault locally.
```
$ vault server -dev -dev-root-token-id=root
```

Configure the Vault client.
```
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN=root
```

Enable the KMS Secrets Engine.
```
$ vault secrets enable keymgmt
Success! Enabled the keymgmt secrets engine at: keymgmt/
```

Configure the KMS Secrets Engine with a Azure Key Vault backend.
```
$ vault write keymgmt/kms/azure \
       provider="azurekeyvault" \
       key_collection="keyvault-snapshot-vault" \
       credentials=tenant_id="${ARM_TENANT_ID}" \
       credentials=client_id="${ARM_CLIENT_ID}" \
       credentials=client_secret="${ARM_CLIENT_SECRET}"
Success! Data written to: keymgmt/kms/azure
```
We are now able to generate keys in HashiCorp Vault and push to them a Azure Key Vault instance.


## Generate a demo key
Create a demo key.
```
$ vault write keymgmt/key/demo-key type="rsa-2048"
Success! Data written to: keymgmt/key/demo-key
```

Inspect the demo key.
```
$ vault read -format=json keymgmt/key/demo-key | jq .
{
  "request_id": "50fa47c5-277f-cfe2-d9c3-b48a15878c74",
  "lease_id": "",
  "lease_duration": 0,
  "renewable": false,
  "data": {
    "deletion_allowed": false,
    "latest_version": 1,
    "min_enabled_version": 1,
    "name": "demo-key",
    "type": "rsa-2048",
    "versions": {
      "1": {
        "creation_time": "2021-10-03T12:07:24.954593+11:00",
        "public_key": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA1JfkIRBadaAyHYOTUOOt\nvJlAoe7kzYsEtS6iwRY+eQl04R5kKtoeNvC6+wUV2kHHViStJD/QYNiMfasr9Ube\nZufvRspR292gErS+EwsNvRUmu6jZZDTJ0xP0m9RYDkUNCaUj7USOgr15yHm9ZvAH\nDROk6m/YePho2PLYzNW0cJTQyy8XSTTtBa36rFodFEvdW05BKOfvKcmP8EHF3xDm\nNXpBQiXTBxcQPly8veXNBEC4Oc+VOepJjFubtCnYMBDovQ2Pcc77TmumgG+s4uh2\nyP9YoyyprlhgoqQdAcWNomVHtl0OulsrmF2Q+3UFv1VxmaRkCTdR7AJPb5vISVE4\nFQIDAQAB\n-----END PUBLIC KEY-----\n"
      }
    }
  },
  "warnings": null
}
```

The demo key that we created is an asymmetric key pair. This means that the public key is exportable from HashiCorp Vault. We are going to use this public key to encrypt a value.

Save a copy of the public key for encrypt operation.
```
$ vault read -format=json keymgmt/key/demo-key | \
        jq -r .data.versions.\"1\".public_key | \
        tee public_key.pem
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAs8T6t9QiWFMXwonF9As1
<..>
wQ70P3/on+P3ePJtnJHwN6fom7oCfki0paFUqmhORXD254vY4pK1rnoe9CDs98IA
-----END PUBLIC KEY-----
```

Create a plain text file.
```
$ echo "hashi4lyf" > message.txt
```

Use openssl to encrypt the plaintext with the public key downloaded from HashiCorp Vault.
> :warning: If you are using the version of `openssl` that comes installed with MacOS, then you'll need to ugrade it. Run `brew install openssl` and make sure to use the new binary.

```
$ export ENCODED_CIPHERTEXT=$(/usr/local/opt/openssl@3/bin/openssl \
        pkeyutl \
        -in message.txt \
        -encrypt \
        -pubin \
        -inkey public_key.pem \
        -pkeyopt rsa_padding_mode:oaep \
        -pkeyopt rsa_oaep_md:sha1 \
        -pkeyopt rsa_mgf1_md:sha1 | \
        base64)

$ echo ${ENCODED_CIPHERTEXT}
F9CDsPkIkm2OuE4cSPc96cT6LTVSE <..>
```

Theres a lot going on in this commands, lets look at what each argument is doing.
| Argument | Description |
| --- | --- |
| `pkeyutl` | Module to interact with public keys.  |
| `-in` | File path to the input plain text. |
| `encrypt` | Encrypt the input with the public key. |
| `-pubin` | The input key is a public key. Defaults to private. |
| `-inkey` | Path to the input key. |
| `-pkeyopt rsa_padding_mode:oaep` | Public key algorithm option. Format `digest:algoritm` |
| `-pkeyopt rsa_oaep_md:sha1` | Public key algorithm option. Format `digest:algoritm` |
| `-pkeyopt rsa_mgf1_md:sha1` | Public key algorithm option. Format `digest:algoritm` |

## Example workflow using both HashiCorp Vault and Azure Key Vault
Push our demo key to the Azure Key Vault backend.
```
$ vault write keymgmt/kms/azure/key/demo-key \
       purpose="decrypt" \
       protection="hsm"
Success! Data written to: keymgmt/kms/azure/key/demo-key
```

The Azure cli needs the randomly generated key name to be able to reference the correct key for the decryption operation. Look up the randomly generated key name.
> :warning: This command will only work if you have one key in your Azure Key Vault instance. Alternativly, lookup the key name manually.
```
$ export KEY_NAME=$(az keyvault key list \
        --vault-name "keyvault-snapshot-vault" | \
        jq -r '.[0].tags.key_name')

$ echo ${KEY_NAME}
demo-key-1633226076
```

Using the name of the Azure Key Vault and the randomly generated key name, we are going to decrypt the the encoded ciphertext from the previous step.
```
$ export DECRYPT_RESPONSE=$(az keyvault key decrypt \
        --vault-name "keyvault-snapshot-vault" \
        --name "${KEY_NAME}" \
        --algorithm "RSA-OAEP" \
        --value "${ENCODED_CIPHERTEXT}")
```

The response from the Azure cli is a JSON object with our decrypted ciphertext base64 encoded. Decode the base64 response.
```
$ echo ${DECRYPT_RESPONSE} | jq -r '.result' | base64 -d
hashi4lyf
```

We have now generated a key outside of Azure Key Vault, distributed it to the cloud provider, and shown what a potential encrypt and decrypt operation might look like.

## Delete the key material from the cloud provider
Delete the key from Azure Key Vault

```
$ vault delete keymgmt/kms/azure/key/demo-key
Success! Data deleted (if it existed) at: keymgmt/kms/azure/key/demo-key
```

> If you were to check in the Azure portal, you'll see the key no longer exists.

Try the decrypt operation with the Azure cli again. We are expecting this to fail.
```
$ export DECRYPT_RESPONSE=$(az keyvault key decrypt \
        --vault-name "keyvault-snapshot-vault" \
        --name "${KEY_NAME}" \
        --algorithm "RSA-OAEP" \
        --value "${ENCODED_CIPHERTEXT}")
WARNING: This command is in preview and under development. Reference and support levels: https://aka.ms/CLI_refstatus
ERROR: A key with (name/id) demo-key-1633223361 was not found in this key vault. If you recently deleted this key you may be able to recover it using the correct recovery command. For help resolving this issue, please see https://go.microsoft.com/fwlink/?linkid=2125182
```

Distribute the same key back to the Azure Key Vault.
```
$ vault write keymgmt/kms/azure/key/demo-key \
       purpose="decrypt" \
       protection="hsm"
```

The name of the key inside Azure Key Vault will have a new randomly generated number. Lookup what the new key name is.
```
$ export KEY_NAME=$(az keyvault key list \
        --vault-name "keyvault-snapshot-vault" | \
        jq -r '.[].tags.key_name')
```

Retry the inital decrypt operation with the Azure cli
```
$ export DECRYPT_RESPONSE=$(az keyvault key decrypt \
        --vault-name "keyvault-snapshot-vault" \
        --name "${KEY_NAME}" \
        --algorithm "RSA-OAEP" \
        --value "${ENCODED_CIPHERTEXT}")

$ echo ${DECRYPT_RESPONSE} | jq -r '.result' | base64 -d
```

This shows that a copy of the original key material remained in HashiCorp Vault, we were able to completly remove it from the cloud provider and then redistribute it back. 

## Rotate the key material stored in the cloud provider.
Create a new version of the key.
```
$ vault write -f keymgmt/key/demo-key/rotate
Success! Data written to: keymgmt/key/demo-key/rotate
```

Inspect new version of the key.
```
$ vault read -format=json keymgmt/key/demo-key | jq .
{
  "request_id": "abfd1224-5024-82f4-076d-a4c16f1b9583",
  "lease_id": "",
  "lease_duration": 0,
  "renewable": false,
  "data": {
    "deletion_allowed": false,
    "latest_version": 2,
    "min_enabled_version": 1,
    "name": "demo-key",
    "type": "rsa-2048",
    "versions": {
      "1": {
        "creation_time": "2021-10-03T12:39:28.779884+11:00",
        "public_key": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAs8T6t9QiWFMXwonF9As1\nvIiXuvughEtQvHRkiVmcPOpA8jjfJaDvoyOtsP53/72wFbKsPsjQirpObRvNKyaF\nORDs9ttsOfsTix7kpnliDJfFudmnExgytinsN2db/E94w/cvov+xIaW5BfYjBI9h\nB5D9CRPA5+ftsPrYoDagcOqBAQVkBVNd/hRsbU+e9IcWZScHnnDd+DgZ4wjVU/T4\nNHi4u2YayR8+tv7vGnf1csR6FywFwRWzsPXGtzHKUKtSvWQI616Ef3k1FTB5jays\nwQ70P3/on+P3ePJtnJHwN6fom7oCfki0paFUqmhORXD254vY4pK1rnoe9CDs98IA\n8QIDAQAB\n-----END PUBLIC KEY-----\n"
      },
      "2": {
        "creation_time": "2021-10-03T16:16:55.525912+11:00",
        "public_key": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0izPeD5xWYVQNLDksyQy\nblJSXylufc+EmdM9VxeEXnGU4aUfp/tqWo4Wj0sYQEQ5uEzNK3zF2xUFINEEUZ1b\norDfokTwT9WLtbEbaUOB4JlQfFEWC6HyRRrvTEMzrbYj6LwTtIOLr8MgH3ZZu8p1\nY7OALxGe6StYdrcVp9nAkgJEBgRqFNmpmpUDzGM3vFHWr6Hwwkz1HY6y6JGjuFr8\n2rNMBNz3P0jloWkiS1S7m1auLDJ2Lwb7dCx/0ggZOo7xW7Gc24rU75KVIY8OVqmh\nlDIQAgNOrKA+YpRWO51+RaCy3gqCMyYayLeBNldxcaayaJ5s2YPUtForV/sIDY2N\nlwIDAQAB\n-----END PUBLIC KEY-----\n"
      }
    }
  },
  "warnings": null
}
```

Disable the previous version of the key.
```
$ vault write -f keymgmt/key/demo-key \
    min_enabled_version=2
Success! Data written to: keymgmt/key/demo-key
```

Inspect the key.
```
$ vault read -format=json keymgmt/key/demo-key | jq .
{
  "request_id": "cc936ed4-6f71-9625-8196-e9d3fceba5b4",
  "lease_id": "",
  "lease_duration": 0,
  "renewable": false,
  "data": {
    "deletion_allowed": false,
    "latest_version": 2,
    "min_enabled_version": 2,
    "name": "demo-key",
    "type": "rsa-2048",
    "versions": {
      "2": {
        "creation_time": "2021-10-03T16:16:55.525912+11:00",
        "public_key": "-----BEGIN PUBLIC KEY-----\nMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA0izPeD5xWYVQNLDksyQy\nblJSXylufc+EmdM9VxeEXnGU4aUfp/tqWo4Wj0sYQEQ5uEzNK3zF2xUFINEEUZ1b\norDfokTwT9WLtbEbaUOB4JlQfFEWC6HyRRrvTEMzrbYj6LwTtIOLr8MgH3ZZu8p1\nY7OALxGe6StYdrcVp9nAkgJEBgRqFNmpmpUDzGM3vFHWr6Hwwkz1HY6y6JGjuFr8\n2rNMBNz3P0jloWkiS1S7m1auLDJ2Lwb7dCx/0ggZOo7xW7Gc24rU75KVIY8OVqmh\nlDIQAgNOrKA+YpRWO51+RaCy3gqCMyYayLeBNldxcaayaJ5s2YPUtForV/sIDY2N\nlwIDAQAB\n-----END PUBLIC KEY-----\n"
      }
    }
  },
  "warnings": null
}
```
Notice the second version of the key is the only one returned. This change will also be reflected in the cloud provider backend.
 
This shows how through a standardised API we can rotate and revoke key material that is stored in the cloud providers backend.  
