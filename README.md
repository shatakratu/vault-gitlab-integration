# GitLab CE + Vault OSS Integration

Minimal guide to integrating HashiCorp Vault OSS with GitLab CE using AppRole authentication.

---

## Overview

Steps covered in this guide:

1.  Enable KV v2 secrets engine
2.  Create a Vault policy
3.  Configure AppRole authentication
4.  Generate credentials for GitLab
5.  Add CI/CD variables
6.  Retrieve secrets in a pipeline

---

## 1. Enable KV v2 and Create Secrets

Enable the KV v2 secrets engine:

```bash
vault secrets enable -path=kv-v2 kv-v2
```

Create a secret:

```bash
vault kv put kv-v2/myapp DB_USER=secret1 DB_PASS=secret2
```

---

## 2. Create Vault Policy

Create a policy file:

``` bash
vi gitlab-policy.hcl
```

`gitlab-policy.hcl`

``` hcl
path "kv-v2/data/myapp" {
  capabilities = ["read"]
}
```

Apply the policy:

``` bash
vault policy write gitlab-policy gitlab-policy.hcl
```

---

## 3. Enable AppRole

```bash
vault auth enable approle
```

Create a role:

```bash
vault write auth/approle/role/gitlab-role \
    token_policies="gitlab-policy" \
    token_ttl=10m \
    token_max_ttl=20m \
    secret_id_ttl=0 # WARNING: never expires
```

---

## 4. Get Role Credentials

Get the Role ID:

```bash
vault read -field=role_id auth/approle/role/gitlab-role/role-id
```

Generate a Secret ID:

```bash
vault write -f -field=secret_id auth/approle/role/gitlab-role/secret-id
```

These values will be stored in GitLab CI/CD variables.

---

## 5. Configure GitLab CI/CD Variables

In your GitLab project:

Project → **Settings → CI/CD → Variables**

Add the following variables:

| Variable Name      | Description              | Masked | Protected |
|--------------------|--------------------------|--------|-----------|
| VAULT_ADDR         | Vault server address     | No     | Optional  |
| VAULT_ROLE_ID      | AppRole Role ID          | Yes    | Yes       |
| VAULT_SECRET_ID    | AppRole Secret ID        | Yes    | Yes       |

Example:

```
VAULT_ADDR=https://vault.example.com:8200
```

---

## 6. Pipeline Example

Create `.gitlab-ci.yml`:

```yaml
stages:
  - build
build-job:
  stage: build
  image: hashicorp/vault:1.14.0
  script:
    # Login to Vault using AppRole
    - echo "Logging into Vault..."
    - export VAULT_TOKEN=$(vault write -field=token auth/approle/login role_id=$VAULT_ROLE_ID secret_id=$VAULT_SECRET_ID)
    # Retrieve secrets
    - export DB_USER=$(vault kv get -field=DB_USER kv-v2/myapp)
    - export DB_PASS=$(vault kv get -field=DB_PASS kv-v2/myapp)
    # Example usage
    - echo "DB_USER=$DB_USER"
    - echo "DB_PASS=$DB_PASS"
  tags:
    - docker-runner
```


---


## Result

The GitLab pipeline authenticates to Vault using AppRole and retrieves
secrets securely during execution.

