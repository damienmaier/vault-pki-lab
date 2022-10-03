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

