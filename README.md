# Vault PKI lab
## Starting the Vault Server

A config file must be provided when starting the Vault server.

The file `config.hcl` specifies that :
- Vault data is located in `./vault/data`
- Vault will listen for HTTP requests at `127.0.0.1:8200`

If it does not exist, create the vault data directory :
```shell
mkdir -p ./vault/data
```

Then start the server :
```shell
vault server -confg=config.hcl
```

## Set up the client
The `vault` command is also used to act as a client.

The Vault client uses the `VAULT_ADDR` environment variable to know the server address.

In the terminal that will be used to act as client, export the environment variable :
```shell
export VAULT_ADDR='http://127.0.0.1:8200'
```
## Initialize the Vault
### Initialize
We request the unseal key to be split into 6 shards, of which 2 are required to unseal the Vault.
```shell
vault operator init -key-shares=6 -key-threshold=2
```
The following information is printed in the terminal :
- The 6 key shards
- A root token, that allows to authenticate as root in the vault and have full control of it. We will use it to do the first configurations.

### Unseal the Vault

Two different key shards must be provided to Vault.

To provide a key shard we run :
```shell
vault operator unseal
```
and then we enter the key shard.

The two key shards can be provided from different terminals.

### Authenticate as root

We authenticate as root by running :
```shell
vault login
```
and then entering the root token.

## Switch to an admin policy
### Create the admin policy

To write a policy into Vault, we must provide a file that describes it.

The policy we add is described in `admin-policy.hcl`.

This file contains the admin policy provided in the [Vault policies tutorial](https://learn.hashicorp.com/tutorials/vault/policies?in=vault/policies) plus some additional capabilities as described in the [Vault pki tutorial](https://learn.hashicorp.com/tutorials/vault/pki-engine). 

To create the admin policy we run:
```shell
vault policy write admin admin-policy.hcl
```
### Switch token
We create a token associated with the new admin policy :
```shell
vault token create -orphan -policy=admin
```
Without the `-orphan` flag the root token would be the parent of the admin token i.e. revoking the root token would revoke the admin token.
This flag allows the admin token to stay valid after the revocation of the root token.

We revoke the root token :
```shell
vault token revoke <root token>
```
According to [the documentation](https://www.vaultproject.io/docs/concepts/tokens), root tokens should be revoked immediately after they are no longer needed.
If a new root token is needed in the feature, it can be generated by a quorum of unseal key holders.

We log in again but this time using the admin token :
```shell
vault login
```
and we provide the admin token.
## PKI
### Root certificate
We generate the root certificate :
```shell
vault secrets enable pki
vault secrets tune -max-lease-ttl=87600h pki
vault write -field=certificate pki/root/generate/internal common_name="heig-vd.ch" issuer_name="root-2022" ttl=87600h > root_2022_ca.crt
```
This :
* Creates a PKI engine at the location `pki`
* Sets the maximal certificate TTL for this PKI engine to 10 years
* Creates a root certificate with a TTL of 10 years and the common name `heig-vd.ch`
* Writes the certificate to the file `root_2022_ca.crt`

We configure the URLs for this PKI engine :
```shell
vault write pki/config/urls issuing_certificates="$VAULT_ADDR/v1/pki/ca" crl_distribution_points="$VAULT_ADDR/v1/pki/crl"
```
Those URLs will be included in the certificates signed by the root certificate.
They indicate [where to find information about the root certificate](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.2.1) and [where to find the CRL](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.13).
### Intermediate certificate
We create a PKI engine for the intermediate certificate at the location `pki_int` and with a maximal TTL of 5 years :
```shell
vault secrets enable -path=pki_int pki
vault secrets tune -max-lease-ttl=43800h pki_int
```
We generate a CSR for the intermediate certificate that is stored in the file `pki_intermediate.csr`:
```shell
vault write -format=json pki_int/intermediate/generate/internal \
     common_name="heig-vd.ch Intermediate Authority" \
     issuer_name="heig-vd-ch-intermediate" \
     | jq -r '.data.csr' > pki_intermediate.csr
```
We sign the intermediate certificate with the root certificate and save it in the file `intermediate.cert.pem`:
```shell
vault write -format=json pki/root/sign-intermediate \
     issuer_ref="root-2022" \
     csr=@pki_intermediate.csr \
     format=pem_bundle ttl="43800h" \
     | jq -r '.data.certificate' > intermediate.cert.pem
```
We import the signed intermediate certificate in the `pki_int` PKI engine :
```shell
vault write pki_int/intermediate/set-signed certificate=@intermediate.cert.pem
```
## Questions
#### 4.1. What is the goal of the unseal process? Why are they more than one unsealing key?
The data stored by Vault is encrypted. The goal of the unseal process is to provide to Vault the key to decrypt the data.

The key needed by Vault is split into shards (6 in this lab), of which a minimum threshold (2 in this lab) is required to recover the root key.

Each key must be given to a different person. Having a minimum threshold of required keys increases security, as an attacker would have to obtain multiple keys from multiple persons to access the vault.
Having a total number of keys greater than the threshold protects against loosing access to the Vault if a key becomes inaccessible.

A group of key holders as much as large as the threshold is called a quorum. 

#### 4.2. What is a security officer ? What do you do if one leaves the company?

Security officers are the trusted persons that will each receive a key shard.

It is not possible to revoke only one key shard. If a security officer leaves the company, the key shards must be replaced by new ones and distributed to the remaining security officers.
This operation is called rekeying.