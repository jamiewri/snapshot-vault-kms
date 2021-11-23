# HashiCorp Vault with Google Cloud KMS Demo

## Prerequisites
- HashiCorp Vault >= 1.9
- `vault`
- `jq` 
- `base64`
- OpenSSL >= 3.0 (Run `openssl version` to validate)
- A GCP KMS Keyring.
- `gcloud` configured with permissions on a GCP KMS Keyring.

## High level demo steps
Configuring the KMS Secrets Engine.
- Enabling the KMS Secrets Engine.
- Configure the KMS Secrets Engine with a GCP KMS Keyring backend.

Generate a demo key
- Generate a demo key.
- Inspect the demo key.
- Use the demo key from HashiCorp Vault to encrypt a value.

Example workflow using both HashiCorp Vault and GCP Keyring
- Distribute the demo key to the GCP KMS Keyring.
- Use the `gcloud` cli to decrypt a value.

Delete the key material from the cloud provider.
- Delete the key from GCP KMS Keyring.
- Validate that the GCP Keyring can no longer decrypt our value.
- Redistibute the same key material back to the GCP KMS Keyring.

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

Set some environment variables
```
$ export GCP_PROJECT=devopstower
$ export GCP_LOCATION=global
$ export GCP_KEYRING_NAME=vault-kms
```

Enable the KMS Secrets Engine.
```
$ vault secrets enable keymgmt
Success! Enabled the keymgmt secrets engine at: keymgmt/
```

Configure the KMS Secrets Engine with a GCP Keyring backend.
```
$ vault write keymgmt/kms/gcp \
    provider="gcpckms" \
    key_collection="projects/${GCP_PROJECT}/locations/${GCP_LOCATION}/keyRings/${GCP_KEYRING_NAME}" \
    credentials=service_account_file="$HOME/.credentials/devopstower.json"
```

We are now able to generate keys in HashiCorp Vault and push to them to a GCP KMS Keyring.

## Generate a demo key
Create the demo key.
```
$ vault write keymgmt/key/demo-key type="rsa-4096"
```

Inspect the demo key
```
$ vault read -format=json keymgmt/key/demo-key | jq .
{
  "request_id": "00a9e171-813d-4e10-7cff-8a462eb4bc45",
  "lease_id": "",
  "lease_duration": 0,
  "renewable": false,
  "data": {
    "deletion_allowed": false,
    "latest_version": 1,
    "min_enabled_version": 1,
    "name": "demo-key",
    "type": "rsa-4096",
    "versions": {
      "1": {
        "creation_time": "2021-11-22T15:09:08.406551+11:00",
        "public_key": "-----BEGIN PUBLIC KEY-----\nMIICIjA<..>ECAwEAAQ==\n-----END PUBLIC KEY-----\n"
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
MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAn3yvBho3J9d8MReShLUP
<..>
/kp8Ed/Tq3IS8KoSAJH6pYECAwEAAQ==
-----END PUBLIC KEY-----
```

Create a plain text file.
```
$ echo "hashi4lyf" > message.txt
```

Use openssl to encrypt the plaintext with the public key downloaded from HashiCorp Vault.
> :warning: If you are using the version of `openssl` that comes installed with MacOS, then you'll need to ugrade it. Run `brew install openssl` and make sure to use the new binary.

Unlike the Azure and AWS workflows in this repository, we are not going to base64 encode the returned ciphertext. The `gcloud` cli expects the binary ciphertext to be written to a file.
```
$ openssl pkeyutl \
        -in message.txt \
        -encrypt \
        -pubin \
        -inkey public_key.pem \
        -pkeyopt rsa_padding_mode:oaep \
        -pkeyopt rsa_oaep_md:sha256 \
        -pkeyopt rsa_mgf1_md:sha256 > cipher.txt.enc
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
| `-pkeyopt rsa_oaep_md:sha256` | Public key algorithm option. Format `digest:algoritm` |
| `-pkeyopt rsa_mgf1_md:sha256` | Public key algorithm option. Format `digest:algoritm` |

## Example workflow using both HashiCorp Vault and GCP KMS Keyring
Distribute our demo key to the GCP KMS Keyring.
```
$ vault write keymgmt/kms/gcp/key/demo-key \
    purpose="decrypt" \
    protection="hsm"
Success! Data written to: keymgmt/kms/gcp-kms/key/demo-key
```

The gcloud cli needs the randomly generated key name to be able to reference the correct key for the decryption operation. Look up the randomly generated key name.
```
$ export KEY_NAME=$(vault  read \
                      -format=json \
                      keymgmt/kms/gcp/key/demo-key | \
                      jq -r '.data.name')

$ echo -n ${KEY_NAME}
demo-key-1637554988
```

Using the name of the GCP Keyring and the randomly generated key name, we are going to decrypt the binary ciphertext from the previous step.
```
$ gcloud kms asymmetric-decrypt \
     --project ${GCP_PROJECT} \
     --location ${GCP_LOCATION} \
     --keyring ${GCP_KEYRING_NAME} \
     --key ${KEY_NAME} \
     --version 1 \
     --ciphertext-file cipher.txt.enc \
     --plaintext-file  - 
hashi4lyf
```

We have now generated a key outside of GCP KMS, distributed it to the cloud provider, and shown what a potential encrypt and decrypt operation might look like.

## Delete the key material from the cloud provider
Delete the key from GCP KMS Keyring
```
$ vault delete keymgmt/kms/gcp/key/demo-key
Success! Data deleted (if it existed) at: keymgmt/kms/gcp/key/demo-key
```

> If you were to check in the Google Cloud console, you'll see the key has been scheduled for deletion.

Try the decrypt operation with the `gcloud` cli again. We are expecting this to fail.
```
$ gcloud kms asymmetric-decrypt \
     --project ${GCP_PROJECT} \
     --location ${GCP_LOCATION} \
     --keyring ${GCP_KEYRING_NAME} \
     --key ${KEY_NAME} \
     --version 1 \
     --ciphertext-file cipher.txt.enc \
     --plaintext-file  -
ERROR: (gcloud.kms.asymmetric-decrypt) FAILED_PRECONDITION: projects/devopstower/locations/global/keyRings/vault-kms/cryptoKeys/demo-key-1637554988/cryptoKeyVersions/1 is not enabled, current state is: DESTROY_SCHEDULED.
```

Distribute the same key back to the GCP KMS Keyring.
```
$ vault write keymgmt/kms/gcp/key/demo-key \
    purpose="decrypt" \
    protection="hsm"
Success! Data written to: keymgmt/kms/gcp/key/demo-key
```

The name of the key inside GCP KMS Keyring will have a new randomly generated name. Lookup what the new key name is.
```
$ export KEY_NAME=$(vault  read \
                      -format=json \
                      keymgmt/kms/gcp/key/demo-key | \
                      jq -r '.data.name')

$ echo -n ${KEY_NAME}
```

Retry the inital decrypt operation with the key we've just redistributed.
```
$ gcloud kms asymmetric-decrypt \
     --project ${GCP_PROJECT} \
     --location ${GCP_LOCATION} \
     --keyring ${GCP_KEYRING_NAME} \
     --key ${KEY_NAME} \
     --version 1 \
     --ciphertext-file cipher.txt.enc \
     --plaintext-file  -
hashi4lyf
```

This shows that a copy of the original key material remained in HashiCorp Vault, we were able to completly remove it from the cloud provider and then redistribute it back. 
