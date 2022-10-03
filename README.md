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
