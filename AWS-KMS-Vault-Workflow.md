# HashiCorp Vault with AWS KMS Demo

## Prerequisites
- HashiCorp Vault >= 1.8
- `vault`
- `jq` 
- `base64`
- `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `AWS_SESSION_TOKEN` environment variables set with permission on a AWS KMS instance.

## High level demo steps
Configuring the KMS Secrets Engine.
- Enabling the KMS Secrets Engine.
- Configure the KMS Secrets Engine with a AWS KMS backend.

Generate a demo key
- Generate a demo key.
- Inspect the demo key.
- Use the demo key from HashiCorp Vault to encrypt a value.

Example workflow using both HashiCorp Vault and AWS KMS
- Distribute the demo key to the AWS KMS backend.
- Use the AWS cli to encrypt a value.
- Use the AWS cli to decrypt a value.

Delete the key material from the cloud provider.
- Delete the key from AWS KMS.
- Validate that AWS KMS can no longer decrypt our value.

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

Configure the KMS Secrets Engine with a AWS KMS backend.
```
$ vault write keymgmt/kms/aws \
      key_collection="ap-southeast-2" \
      provider="awskms" \
      credentials=access_key="${AWS_ACCESS_KEY_ID}" \
      credentials=secret_key="${AWS_SECRET_ACCESS_KEY}" \
      credentials=session_token="${AWS_SESSION_TOKEN}"
Success! Data written to: keymgmt/kms/aws
```
We are now able to generate keys in HashiCorp Vault and push to them a AWS KMS.

## Generate a demo key
Create a demo key.
```
$ vault write keymgmt/key/demo-key type="aes256-gcm96"
Success! Data written to: keymgmt/key/demo-key
```

> :warning: As of HashiCorp Vault 1.8, the only supported key type that can be pushed to AWS KMS is `aes256-gcm96`.

See the HashiCorp [documentation](https://www.vaultproject.io/docs/secrets/key-management#compatibility)

Inspect the demo key.
```
$ vault read -format=json keymgmt/key/demo-key | jq .
{
  "request_id": "560f87f4-4026-80a0-2c19-7a7f8a4b5897",
  "lease_id": "",
  "lease_duration": 0,
  "renewable": false,
  "data": {
    "deletion_allowed": false,
    "latest_version": 1,
    "min_enabled_version": 1,
    "name": "demo-key",
    "type": "aes256-gcm96",
    "versions": {
      "1": {
        "creation_time": "2021-10-03T19:33:08.250676+11:00"
      }
    }
  },
  "warnings": null
}
```

The demo key that we created is a symmetric key, this means that there is not a public key that we could use for an encrypt operation like in the Azure workflow. Instead, we are going to encrypt with the AWS cli.

Push our demo key to the AWS KMS backend.
```
$ vault write keymgmt/kms/aws/key/demo-key \
    purpose=encrypt,decrypt \
    protection=hsm
Success! Data written to: keymgmt/kms/aws/key/demo-key
```

Find the randomly generated AWS KMS Key Id.

> :warning: This command will only work if you have one key that contains the name 'demo-key' in your customer managed keys region. Alternatively, lookup the key ID manually in the AWS console.

```
export KEY_ID=$(aws kms list-aliases | \
    jq -r '.Aliases[] | select(.AliasName | contains("demo-key")).TargetKeyId')

echo ${KEY_ID}
cd5e659e-33f2-4e8a-9203-19a0066a2d5b
```

Encrypt plain text with the `aws` cli
```
$ export ENCODED_CIPHERTEXT=$(aws kms encrypt \
    --no-cli-pager \
    --key-id ${KEY_ID} \
    --plaintext $(echo hashi4lyf | base64) | \
    jq -r .CiphertextBlob)

$ echo ${ENCODED_CIPHERTEXT}
AQICAHiHEQZvGjFlxXQn3WYylJS+0LpDXD02 <..>
```

Using the ID of the AWS KMS Key, we are going to decrypt the the encoded ciphertext from the previous step.
```
$ export DECRYPT_RESPONSE=$(aws kms decrypt \
    --no-cli-pager \
    --ciphertext-blob ${ENCODED_CIPHERTEXT} \
    --key-id ${KEY_ID})
```

The response from the AWS cli is a JSON object with our decrypted ciphertext base64 encoded. Decode the base64 response.
```
$ echo ${DECRYPT_RESPONSE} | jq -r '.Plaintext' | base64 -d
hashi4lyf
```

We have now generated a key outside of AWS KMS, distributed it to the cloud provider, and shown what a potential encrypt and decrypt operation might look like.

## Delete the key material from the cloud provider
Delete the key from AWS KMS.

```
$ vault delete keymgmt/kms/aws/key/demo-key
Success! Data deleted (if it existed) at: keymgmt/kms/aws/key/demo-key
```

> :warning: If you were to check in the AWS Console, you'll see the key no longer exists.

Try the decrypt operation with the AWS cli again. We are expecting this to fail.
```
$ aws kms decrypt \
    --no-cli-pager \
    --ciphertext-blob ${ENCODED_CIPHERTEXT} \
    --key-id ${KEY_ID}
An error occurred (KMSInvalidStateException) when calling the Decrypt operation: arn:aws:kms:ap-southeast-2:711129375688:key/cd5e659e-33f2-4e8a-9203-19a0066a2d5b is pending deletion.
```
