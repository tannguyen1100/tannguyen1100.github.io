---
title:  "How to set up Vault"
date:   2026-06-02 15:00:00 +0700
tags: coding
toc: true
toc_sticky: true
---

**tl;dr** Here is how I set up Vault on my local environment.

## My current setup

- OS: Windows 11 with WSL2 (Ubuntu)
- docker, openssl, vault cli installed on WSL2

## How to

``` text
~/vault-setup
|-- vault
    |-- audit
    |-- config
          |-- config.hcl
    |-- data
    |-- file
    |-- logs
    |-- userconfig
          |-- tls
                |-- vault.crt
                |-- vault.key
    |-- plugins
|-- docker-compose.yml
```

### 1. Create a self-signed certificate for Vault

``` bash
openssl req -x509 -newkey rsa:2048 -nodes -keyout vault.key -out vault.crt -days 365 -subj "/CN=[example.com]" -addext "subjectAltName = DNS:[example.com],IP:127.0.0.1"
```

Copy the `vault.crt` file to `/usr/share/ca-certificates/extra/`, update `/etc/ca-certificates.conf` to include vault.crt

``` text
# /etc/ca-certificates.conf
... other certificates ...
extra/vault.crt
```

then run `update-ca-certificates` to trust the certificate.

### 2. Set up Vault with docker-compose

Create `docker-compose.yml` and `vault/config/config.hcl`

``` yaml
# docker-compose.yml

services:
  vault:
    privileged: true
    image: "hashicorp/vault:2.0"
    restart: unless-stopped
    container_name: vault
    ports:
    - "8200:8200"
    environment:
      VAULT_ADDR: "https://127.0.0.1:8200"
      # VAULT_CACERT: /vault/userconfig/tls/ca.crt'
    user: root
    cap_add:
    - IPC_LOCK
    volumes:
    - ./vault/audit:/vault/audit/:rw
    - ./vault/config:/vault/config/:rw
    - ./vault/data:/vault/data/:rw
    - ./vault/file:/vault/file/:rw
    - ./vault/logs:/vault/logs/:rw
    - ./vault/userconfig:/vault/userconfig/:rw
    - ./vault/plugins:/vault/plugins/:rw
    # - /etc/ssl/certs/ca-certificates.crt:/etc/ssl/certs/ca-certificates.crt
    command: vault server -config=./vault/config/config.hcl
    networks:
    - services_net

networks:
  services_net:
    driver: bridge
```

``` yaml
# vault/config/config.hcl

disable_cache       = true
disable_mlock       = true
ui                  = true
max_lease_ttl       = "2h"
default_lease_ttl   = "20m"
raw_storage_endpoint = "true"
disable_printable_check = "true"
cluster_addr        = "https://127.0.0.1:8201"
api_addr            = "http://127.0.0.1:8200"

listener "tcp" {
  address                   = "0.0.0.0:8200"
  tls_disable               = false
  tls_cert_file             = "./vault/userconfig/tls/vault.crt"
  tls_key_file              = "./vault/userconfig/tls/vault.key"
  # tls_client_ca_file        = ""
  # tls_disable_client_certs  = "true"
}

storage "raft" {
  node_id  = "vault-1"
  path     = "/vault/data"
}
```

run `docker-compose up -d` to start Vault.

### 3. Initialize Vault

Now Vault is running and you need to initialize it.

``` bash
vault operator init

# Example output:
Unseal Key 1: 0dVEXn1uHc01mtjzzJ1FdqZCCG6mVcwez4HUbYvwiGiT
Unseal Key 2: HXkIyeoVZrH5BiFA6B5ysQoNpbOyfFcta30TYdvLuH90
Unseal Key 3: jJAZMtvCzs68gE8hRsRaZpgAhrAyBEny0YAeCbMmwwtg
Unseal Key 4: Dytmg+vLyRoSzaiw+46X0kK2AOiOwytmFxKMeXNt/RC8
Unseal Key 5: aStaONYWYiYWMi/zaVDz6OWFdT2jHY5JApFZlZzFuoMP

Initial Root Token: hvs.JTOvgkSyAv1QDT8fcsV3nLqL
```

--> run `vault operator unseal` 3 times with 3 different unseal keys to unseal the vault.

``` bash
vault operator unseal <unseal_key_1>
vault operator unseal <unseal_key_2>
vault operator unseal <unseal_key_3>
```

--> run `vault login` with the initial root token to log in to the vault.

``` bash

vault login <initial_root_token>
```

And you can access the vault UI at `https://127.0.0.1:8200` with the initial root token.
