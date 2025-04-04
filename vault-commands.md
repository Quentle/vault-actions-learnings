# If needed, enable the KV engine at the path "kv":
curl --header "X-Vault-Token: $VAULT_TOKEN" \
     --request POST \
     --data '{"type":"kv","options":{"version":"2"}}' \
     http://127.0.0.1:8200/v1/sys/mounts/kv

# Store secrets (REST API):
curl --header "X-Vault-Token: $VAULT_TOKEN" \
     --request POST \
     --data '{"data":{"SreSecret1":"value1","SreSecret2":"value2"}}' \
     http://127.0.0.1:8200/v1/kv/data/sre-secrets

# Alternatively, use the Vault CLI:
vault kv put kv/sre-secrets SreSecret1="value1" SreSecret2="value2"

# Enable JWT authentication and configure GitHub OIDC
vault auth enable jwt

vault write auth/jwt/config \
  oidc_discovery_url="https://token.actions.githubusercontent.com" \
  bound_issuer="https://token.actions.githubusercontent.com" \
  default_role="my-github-role" \
  allowed_redirect_uris='["https://localhost:8250/oidc/callback"]'

# Create a Vault policy named sre-secrets-read-policy with read access to kv/data/sre-secrets
vault write sys/policies/acl/sre-secrets-read-policy policy="
path \"kv/data/sre-secrets*\" {
  capabilities = [\"read\"]
}
"

# Create a role with access to kv secrets
vault write auth/jwt/role/my-github-role \
  bound_claims_type="glob" \
  bound_claims={"sub":"repo:Quentle/*"} \ \
  user_claim="repository" \
  policies=["default","sre-secrets-read-policy"] \
  ttl="1h" \
  allowed_redirect_uris='["https://localhost:8250/oidc/callback"]'

# Additional REST API example calls:

# Enable JWT auth:
curl --header "X-Vault-Token: $VAULT_TOKEN" \
     --request POST \
     --data '{"type":"jwt"}' \
     "$VAULT_ADDR/v1/sys/auth/jwt"

# Configure JWT with GitHub OIDC:
curl --header "X-Vault-Token: $VAULT_TOKEN" \
     --request POST \
     --data '{"oidc_discovery_url":"https://token.actions.githubusercontent.com","bound_issuer":"https://token.actions.githubusercontent.com","default_role":"my-github-role"}' \
     "$VAULT_ADDR/v1/auth/jwt/config"

# Create the sre-secrets-read-policy:
curl --header "X-Vault-Token: $VAULT_TOKEN" \
     --request PUT \
     --data '{"policy":"path \"kv/data/sre-secrets*\" { capabilities = [\"read\"] }"}' \
     "$VAULT_ADDR/v1/sys/policies/acl/sre-secrets-read-policy"

# Create the my-github-role:
curl --header "X-Vault-Token: $VAULT_TOKEN" \
     --request POST \
     --data '{"bound_claims_type":"glob","bound_claims":{"sub":"repo:Quentle/*"},"user_claim":"repository","policies":["default","sre-secrets-read-policy"],"ttl":"1h","allowed_redirect_uris":["https://localhost:8250/oidc/callback"]}' \
     "$VAULT_ADDR/v1/auth/jwt/role/my-github-role"
