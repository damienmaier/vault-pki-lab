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

## Questions
#### 4.1. What is the goal of the unseal process? Why are they more than one unsealing key?
The data stored by Vault is encrypted. The goal is to allow Vault to obtain the key to decrypt the data.

The key needed by Vault is split into shards (6 in this lab), of which a minimum threshold (2 in this lab) is required to recover the root key.

Each key must be given to a different person. Having a minimum threshold of required keys increases security, as an attacker would have to obtain multiple keys from multiple persons to access the vault.
Having a total number of keys greater than the threshold protects against loosing access to the Vault if a key becomes inaccessible.

#### 4.2. What is a security officer ? What do you do if one leaves the company?

Security officers are the trusted persons that will each receive a key shard.

It is not possible to revoke only one key shard. If a security officer leaves the company, the key shards must be replaced by new ones and distributed to the remaining security officers.
This operation is called rekeying.