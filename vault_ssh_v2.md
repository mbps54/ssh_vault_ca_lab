# VAULT SSH CA LAB (TOKEN-BASED WORKFLOW)

## Goal

Implement temporary SSH access to Linux servers using **Vault as an SSH
Certificate Authority (CA)**.\
Issued SSH certificates will be valid for a short period (TTL = 30
minutes in this lab).
Demonstrate SSH access using AppRole authentication - Short‑lived Vault tokens.

This removes the need for static `authorized_keys` management.

## Infrastructure

| Role            | IP               |
|-----------------|------------------|
| Vault Server   | 172.236.193.139  |
| Target Server  | 172.236.193.212  |
| Client         | 172.238.122.135  |

------------------------------------------------------------------------

# PART 1 --- VAULT SETUP

All commands in this section run on Vault Server (172.236.193.139)

## STEP 1 --- Install Vault

``` bash
apt update
apt install -y wget unzip

wget https://releases.hashicorp.com/vault/1.15.6/vault_1.15.6_linux_amd64.zip
unzip vault_1.15.6_linux_amd64.zip
mv vault /usr/local/bin/

vault version
```

## STEP 2 --- Start Vault (Lab Mode)

``` bash
vault server -dev -dev-root-token-id=root
```

Keep this terminal running.

## STEP 3 --- Login as root

Open new terminal on Vault Server:

``` bash
export VAULT_ADDR=http://127.0.0.1:8200
vault login root
```

## STEP 4 --- Enable SSH Secrets Engine

``` bash
vault secrets enable ssh
```

## STEP 5 --- Generate SSH CA Key

``` bash
vault write ssh/config/ca generate_signing_key=true
```

Vault is now an SSH Certificate Authority.

## STEP 6 --- Create SSH Role (TTL 30m)

``` bash
vault write ssh/roles/dev-role -<<EOH
{
  "key_type": "ca",
  "allow_user_certificates": true,
  "default_user": "root",
  "allowed_users": "root,deploy,app",
  "allowed_extensions": "permit-pty,permit-agent-forwarding",
  "default_extensions": {
    "permit-pty": ""
  },
  "ttl": "30m"
}
EOH
```

------------------------------------------------------------------------

# PART 2 --- CONFIGURE TARGET SERVER TO TRUST VAULT CA

### STEP 1 --- Trust Vault CA

First, on Vault Server (172.236.193.139):

``` bash
vault read -field=public_key ssh/config/ca > ca.pub
scp ca.pub root@172.236.193.212:/etc/ssh/ca.pub
```

Now on Target Server (172.236.193.212):

``` bash
echo "TrustedUserCAKeys /etc/ssh/ca.pub" >> /etc/ssh/sshd_config
systemctl restart ssh
```

Target now trusts Vault‑signed certificates.

------------------------------------------------------------------------

# PART 3 --- TOKEN‑BASED ACCESS (AppRole)

All commands in this section run on Vault Server (172.236.193.139)

## STEP 1 --- Enable AppRole

``` bash
vault auth enable approle
```

## STEP 2 --- Create Signing Policy

``` bash
vault policy write ssh-sign-policy - <<EOF
path "ssh/sign/dev-role" {
  capabilities = ["create", "update"]
}
EOF
```

## STEP 3 --- Create AppRole

``` bash
vault write auth/approle/role/ssh-automation \
token_policies="ssh-sign-policy" \
token_ttl=15m \
token_max_ttl=30m
```

## STEP 4 --- Retrieve role_id

``` bash
vault read auth/approle/role/ssh-automation/role-id
```

Example output:
role_id: XXX

## STEP 5 --- Generate secret_id

``` bash
vault write -f auth/approle/role/ssh-automation/secret-id
```

Example output:
secret_id: XXX


------------------------------------------------------------------------

# PART 4 --- CLIENT KEY GENERATION AND SUBSCRIPTION

All commands in this section run on Client (172.238.122.135)

## STEP 1 --- Generate SSH Key

``` bash
ssh-keygen -t ed25519 -f id_ed25519
```
## STEP 2 --- Login Using AppRole (Get Vault Token)

```bash
export ROLE_ID="XXX"
export SECRET_ID="XXX"
export VAULT_TOKEN=$(vault write -field=token \
            auth/approle/login \
            role_id="$ROLE_ID" \
            secret_id="$SECRET_ID")
```

## STEP 3 --- Sign certificate

```bash
vault write -field=signed_key \
            ssh/sign/dev-role \
            public_key=@id_ed25519.pub \
            > id_ed25519-cert.pub
```

check
```bash
ssh-keygen -L -f id_ed25519-cert.pub
```

## STEP 4 --- Login

```
ssh -i id_ed25519 root@172.236.193.212
```

------------------------------------------------------------------------
# SHORT SUMMARY

Get token
```bash
export ROLE_ID="XXX"
export SECRET_ID="XXX"
export VAULT_TOKEN=$(vault write -field=token \
            auth/approle/login \
            role_id="$ROLE_ID" \
            secret_id="$SECRET_ID")
```

Sign certificate
```bash
vault write -field=signed_key \
            ssh/sign/dev-role \
            public_key=@id_ed25519.pub \
            > id_ed25519-cert.pub
```

check
```bash
ssh-keygen -L -f id_ed25519-cert.pub
```

Login
```bash
ssh -i id_ed25519 root@172.236.193.212
```