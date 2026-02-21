# GitLab CE + HashiCorp Vault Integration Guide

This guide explains how to integrate GitLab Community Edition (CE) with HashiCorp Vault (Open Source) using AppRole authentication to securely inject secrets into GitLab CI/CD pipelines.

---

## Overview

This guide covers:

- Enabling KV v2 secrets engine
- Creating and applying a Vault policy
- Configuring AppRole authentication
- Connecting GitLab CI/CD to Vault
- Retrieving secrets securely in a pipeline

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

## 2. Create a Vault Policy

Create a policy file:

```bash
vi gitlab-policy.hcl
```

Contents of `gitlab-policy.hcl`:

```hcl
path "kv-v2/myapp" {
  capabilities = ["read"]
}

path "kv-v2/data/myapp" {
  capabilities = ["read"]
}
```

Apply the policy:

```bash
vault policy write gitlab-policy gitlab-policy.hcl
vault policy read gitlab-policy
```

---

## 3. Enable and Configure AppRole

Enable AppRole authentication:

```bash
vault auth enable approle
```

Create a role for GitLab:

```bash
vault write auth/approle/role/gitlab-role \
    token_policies="gitlab-policy" \
    secret_id_ttl=0 \
    token_ttl=10m \
    token_max_ttl=20m
```

---

## 4. Retrieve ROLE_ID and SECRET_ID

Get the Role ID:

```bash
vault read -field=role_id auth/approle/role/gitlab-role/role-id
```

Generate a Secret ID:

```bash
vault write -f -field=secret_id auth/approle/role/gitlab-role/secret-id
```

Store both values securely. They will be used as GitLab CI/CD variables.

---

## 5. Configure GitLab CI/CD Variables

In your GitLab project:

**Settings → CI/CD → Variables**

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

## 6. Create `.gitlab-ci.yml`

Create .gitlab-ci.yml in the project root:

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

Your GitLab CI/CD pipelines now securely fetch secrets from Vault without storing them in the repository.

