# Kagent Authentication Chart

A comprehensive Helm chart that adds OAuth2-based authentication to Kagent using Keycloak as the OIDC identity provider. The chart uses Crossplane with the Keycloak provider to declaratively manage authentication infrastructure including realms, clients, roles, and protocol mappers.

## Features

- **OAuth2 Proxy Integration** - Secure authentication gateway for Kagent
- **Keycloak OIDC Provider** - Centralized identity management
- **Role-Based Access Control (RBAC)** - Users must have the `kagent-user` role to access Kagent
- **Infrastructure as Code** - All Keycloak resources (realm, client, roles, mappers) defined in Kubernetes via Crossplane
- **Gateway API Support** - Automatic routing configuration with TLS termination
- **Automatic Secret Generation** - Cookie password generated and managed by External Secrets
- **Protocol Mappers** - JWT token enriched with role claims for authorization

## Prerequisites

### Required Dependencies

- **[Crossplane](https://docs.crossplane.io/latest/get-started/install/)** - Infrastructure as Code platform
- **[Crossplane Keycloak Provider](https://github.com/crossplane-contrib/provider-keycloak)** - Manage Keycloak resources via Kubernetes (v2.12.1+)
- **[External Secrets Operator](https://external-secrets.io/latest/introduction/getting-started/)** - Manage sensitive data and generate passwords
- **[Keycloak Instance](https://www.keycloak.org/getting-started)** - Running and accessible (e.g., `https://auth.lab1.kubekub.com`)
- **[Gateway API](https://gateway-api.sigs.k8s.io/guides/getting-started/)** (optional) - For automatic routing and ingress

### Cluster Requirements

- Kubernetes 1.24+
- Istio or other Gateway API implementation (for routing)
- [Cert-manager](https://cert-manager.io/docs/installation/) (for TLS certificate management)

## Configuration

### OAuth2 Proxy Settings

| Parameter | Default | Description |
|-----------|---------|-------------|
| `oauth2-proxy.enabled` | `true` | Enable OAuth2 Proxy deployment |
| `oauth2-proxy.config.clientID` | `oauth2-proxy` | Keycloak client ID |
| `oauth2-proxy.extraArgs.provider` | `keycloak-oidc` | OIDC provider type |
| `oauth2-proxy.extraArgs.allowed-role` | `user` | Required role for access |
| `oauth2-proxy.resources.limits.cpu` | `100m` | CPU limit |
| `oauth2-proxy.resources.limits.memory` | `128Mi` | Memory limit |

### Keycloak Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `keycloak.enabled` | `true` | Enable Keycloak resource provisioning |
| `keycloak.realm.name` | `kagent` | Realm name |
| `keycloak.realm.displayName` | `KAgent Realm` | Realm display name |
| `keycloak.client.name` | `oauth2-proxy` | OAuth2 client name |
| `keycloak.client.accessType` | `CONFIDENTIAL` | Client type (CONFIDENTIAL/PUBLIC) |
| `keycloak.roles[].name` | `kagent-user` | Roles to create |

### Gateway API Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `gatewayApi.enabled` | `true` | Enable Gateway API resources |
| `gatewayApi.gateway.className` | `istio` | Gateway class |
| `gatewayApi.gateway.listeners.https.tls.version` | `1.3` | TLS version |

## Architecture

### Component Overview

```
┌─────────────┐
│   User      │
└──────┬──────┘
       │ HTTPS
       ▼
┌─────────────────────────────────┐
│  Gateway                        │ Gateway API
│  - TLS Termination              │
│  - HTTP → HTTPS Redirect        │
└──────┬──────────────────────────┘
       │
       ▼
┌─────────────────────────────────┐
│  OAuth2 Proxy                   │
│  - OIDC Authentication          │
│  - Session Management           │
│  - Role Validation (RBAC)       │
└──────┬──────────────────────────┘
       │ (authenticated)
       ▼
┌─────────────────────────────────┐
│  Kagent Application             │ Protected by OAuth2 Proxy
│  - Kubernetes Agent             │
│  - API Endpoints                │
└─────────────────────────────────┘

┌──────────────────────────────────┐
│  Keycloak (External)             │
│  - OAuth2/OIDC Provider          │ Managed by Crossplane
│  - Realm: kagent                 │
│  - Client: oauth2-proxy          │
│  - Roles: kagent-user            │
│  - Protocol Mappers: role claims │
└──────────────────────────────────┘
```

### Authentication Flow

1. **User Access** - User requests `https://kagent.lab1.kubekub.com`
2. **Gateway Processing** - Gateway API routes to OAuth2 Proxy via HTTPRoute
3. **OIDC Authentication** - OAuth2 Proxy redirects to Keycloak
4. **User Login** - User authenticates in Keycloak realm `kagent`
5. **Role Check** - Keycloak issues access token with `realm_access.roles` claim
6. **Authorization** - OAuth2 Proxy validates `kagent-user` role
7. **Session Cookie** - OAuth2 Proxy sets secure session cookie
8. **Application Access** - User can access Kagent with valid session

### Crossplane Resources

The chart creates the following Keycloak resources via Crossplane:

- **Realm** (`realm.yaml`) - Keycloak realm for Kagent
- **Client** (`client.yaml`) - OAuth2 OIDC client with confidential access
- **Role** (`role.yaml`) - `kagent-user` role for authorization
- **Protocol Mapper** (`role-protocol-mappers.yaml`) - Adds roles to JWT claims
- **Client Default Scopes** (`clientdefaultscopes.yaml`) - OAuth2 scopes

## Usage

### Adding Users to Kagent

Users must be assigned the `kagent-user` role in Keycloak to access Kagent:

```bash
# Via Keycloak Admin Console
1. Navigate to Realm: kagent
2. Select Users
3. Click user account
4. Go to Role Mapping tab
5. Assign "kagent-user" role

# Via Crossplane (Kubernetes)
apiVersion: user.keycloak.m.crossplane.io/v1alpha1
kind: User
metadata:
  name: john-doe
spec:
  forProvider:
    realmId: kagent
    username: john.doe@example.com
    enabled: true
    email: john.doe@example.com
---
apiVersion: user.keycloak.m.crossplane.io/v1alpha1
kind: UserRole
metadata:
  name: john-doe-kagent-user
spec:
  forProvider:
    realmId: kagent
    userId: john-doe
    roleName: kagent-user
```

### Verify Authentication

```bash
# 1. Check OAuth2 Proxy pod
kubectl get pods -n kagent -l app=oauth2-proxy

# 2. View OAuth2 Proxy logs
kubectl logs -n kagent -l app=oauth2-proxy -f

# 3. Verify Keycloak client
kubectl get client -n kagent

# 4. Check gateway routing
kubectl get gateway,httproute -n kagent
```


## Troubleshooting

### Authentication Fails

**Symptom**: "Invalid client credentials" error

**Solution**:
1. Verify Keycloak client secret is created
   ```bash
   kubectl get secret oauth2-proxy-keycloak-secret -n kagent
   ```
2. Check secret contains `attribute.client_secret`
   ```bash
   kubectl get secret oauth2-proxy-keycloak-secret -n kagent -o jsonpath='{.data}' | base64 -d
   ```

### Roles Not in Token

**Symptom**: User authenticated but role not present in token

**Solution**:
1. Verify protocol mapper is created
   ```bash
   kubectl get protocolmapper -n kagent
   ```
2. Check mapper logs
   ```bash
   kubectl describe protocolmapper oauth2-proxy-roles -n kagent
   ```

### Access Denied Despite Role

**Symptom**: User has role but still gets "access denied"

**Solution**:
1. Verify OAuth2 Proxy configuration
   ```bash
   kubectl get deployment oauth2-proxy -n kagent -o yaml | grep -A 5 "allowed-role"
   ```
2. Decode access token to verify role claim
   ```bash
   # From OAuth2 Proxy logs, extract token and decode at jwt.io
   ```

### Gateway Not Routing

**Symptom**: HTTPRoute not working

**Solution**:
1. Verify gateway exists
   ```bash
   kubectl get gateway -n kagent
   ```
2. Check HTTPRoute status
   ```bash
   kubectl describe httproute kagent -n kagent
   ```
3. Verify certificate exists
   ```bash
   kubectl get secret kagent-server-cert -n kagent
   ```

## Dependencies

- **external-secrets** - Generates and manages cookie secret
- **crossplane** - Infrastructure as Code platform
- **crossplane-keycloak** - Keycloak provider for Crossplane
- **cert-manager** - TLS certificate management
- **gateway-api** - Service routing and ingress

## Values Reference

See `values.yaml` for complete configuration options.

Key sections:
- `global` - Global chart settings
- `oauth2-proxy` - OAuth2 Proxy Helm chart values
- `keycloak` - Keycloak resource configuration
- `gatewayApi` - Gateway API configuration
- `cookieSecret` - External secret password generation
