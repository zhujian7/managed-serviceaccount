# Enhancement: ManagedServiceAccountTokenRequest API

## Release Signoff Checklist

- [ ] Enhancement issue in release milestone, which links to pull request in [enhancements repository]
- [ ] Core members have approved the issue with the label `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place
- [ ] User-facing documentation has been created

## Summary

This enhancement proposes a new `ManagedServiceAccountTokenRequest` API that generates short-lived, on-demand service account tokens from managed clusters via the cluster-proxy, replacing the current pattern of synchronizing long-lived tokens as Secrets on the hub cluster.

The design includes:
- **Core TokenRequest API**: Aggregated API server for ephemeral token generation
- **ManagedClusterSet Integration**: Simplified multi-cluster RBAC using OCM's ManagedClusterSet/Binding
- **Migration Guide**: Migration path for existing ClusterProfile credentials plugin

## Table of Contents

- [Motivation](#motivation)
- [Goals and Non-Goals](#goals-and-non-goals)
- [Core API Design](#core-api-design)
- [ManagedClusterSet Integration](#managedclusterset-integration)
- [ClusterProfile Plugin Migration](#clusterprofile-plugin-migration)
- [Test Plan](#test-plan)
- [Risks and Mitigations](#risks-and-mitigations)
- [Security Considerations](#security-considerations)
- [Observability](#observability)
- [Future Enhancements](#future-enhancements)
- [Alternatives Considered](#alternatives-considered)
- [Conclusion](#conclusion)

## Motivation

### Current Architecture and Security Concerns

The existing ManagedServiceAccount implementation synchronizes service account tokens from managed clusters to the hub cluster as Secret resources. While functional, this approach has significant security concerns:

1. **Centralized Attack Surface**: All managed cluster tokens are stored on the hub cluster. A hub cluster compromise exposes credentials for all managed clusters.

2. **Long-lived Credentials**: Tokens have a default validity of 8640 hours (360 days). Even with rotation, these are persistent credentials that can be extracted and reused.

3. **Privilege Escalation Path**: Compromised hub cluster secrets can be used to pivot to managed clusters with the permissions granted to the service accounts.

4. **Secret Sprawl**: Each ManagedServiceAccount creates a Secret on the hub, increasing the attack surface and management overhead.

5. **Storage Security**: Kubernetes Secrets are only base64-encoded by default, requiring additional encryption-at-rest configuration for proper security.

### Problems This Enhancement Addresses

This enhancement addresses the security concerns while maintaining the convenience of centralized access:

- **Eliminate persistent token storage** on the hub cluster
- **Reduce token lifetime** from days to hours (or minutes)
- **Minimize attack surface** by generating tokens only when needed
- **Align with security best practices** (least privilege, just-in-time access)
- **Leverage existing infrastructure** (cluster-proxy) without introducing new dependencies
- **Simplify multi-cluster RBAC** using ManagedClusterSet/ManagedClusterSetBinding

### Use Cases

1. **Automated Operations**: Controllers on the hub that need to perform operations on managed clusters can request short-lived tokens instead of using stored credentials.

2. **CLI Tools**: Hub administrators can use `kubectl create` to generate tokens for temporary access to managed clusters.

3. **CI/CD Pipelines**: Automation systems can request tokens with specific audiences and expiration times bound to job lifecycles.

4. **Multi-cluster Operators**: Operators that need to access multiple managed clusters can request tokens on-demand with appropriate scoping.

5. **Multi-tenant Environments**: Applications in different namespaces can access different sets of clusters via ManagedClusterSetBindings.

## Goals and Non-Goals

### Goals

- Provide an API for generating short-lived service account tokens from managed clusters
- Eliminate the need to store persistent tokens as Secrets on the hub cluster
- Support Kubernetes TokenRequest API features (audiences, expiration, bound object references)
- Simplify multi-cluster RBAC using ManagedClusterSet/ManagedClusterSetBinding
- Maintain backward compatibility with existing ManagedServiceAccount Secret-based workflow
- Leverage existing cluster-proxy infrastructure for managed cluster communication
- Provide clear migration path from v1beta1 Secret-based pattern to TokenRequest API

### Non-Goals

- Replace the cluster-proxy component (we depend on it)
- Modify the core ManagedServiceAccount CRD (separate API resource)
- Support offline token generation (requires cluster-proxy connectivity)
- Implement token caching or storage (consumers can cache client-side if needed)
- Change the addon agent deployment mechanism

---

## Core API Design

### Design Overview

Introduce a new `ManagedServiceAccountTokenRequest` API resource that acts as an ephemeral token generator. When a client creates this resource, the hub's aggregated API server:

1. Validates the request and client permissions
2. Connects to the target managed cluster via cluster-proxy
3. Calls the Kubernetes TokenRequest API on the managed cluster
4. Returns the short-lived token in the response (not persisted)

```
┌─────────────────────────────────────────────────────────────┐
│                        Hub Cluster                          │
│                                                              │
│  ┌──────────┐    CREATE TokenRequest                       │
│  │  Client  │──────────────────────────┐                   │
│  │ /kubectl │                           │                   │
│  └──────────┘                           ▼                   │
│                          ┌───────────────────────────┐      │
│                          │   Aggregated API Server   │      │
│                          │  (TokenRequest Handler)   │      │
│                          └───────────┬───────────────┘      │
│                                      │                       │
│                                      │ via cluster-proxy     │
└──────────────────────────────────────┼───────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    Managed Cluster                          │
│                                                              │
│                          ┌───────────────────────────┐      │
│                          │  ServiceAccount           │      │
│                          │  TokenRequest API         │      │
│                          └───────────┬───────────────┘      │
│                                      │                       │
│                                      │ return token          │
│                                      ▼                       │
│                          ┌───────────────────────────┐      │
│                          │  Short-lived Token        │      │
│                          │  (not persisted)          │      │
│                          └───────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### API Definition

#### ManagedServiceAccountTokenRequest

```go
// ManagedServiceAccountTokenRequest is used to request short-lived tokens
// for ManagedServiceAccounts. This is a CREATE-only API resource that
// returns an ephemeral token without persisting it.
//
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +genclient
// +genclient:onlyVerbs=create
type ManagedServiceAccountTokenRequest struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   ManagedServiceAccountTokenRequestSpec   `json:"spec,omitempty"`
    Status ManagedServiceAccountTokenRequestStatus `json:"status,omitempty"`
}

// ManagedServiceAccountTokenRequestSpec defines the desired token parameters
type ManagedServiceAccountTokenRequestSpec struct {
    // TargetCluster is the name of the managed cluster where the
    // ManagedServiceAccount exists. The ManagedServiceAccount resource
    // should exist in the namespace matching this cluster name on the hub.
    // +required
    TargetCluster string `json:"targetCluster"`

    // ManagedServiceAccount is the name of the ManagedServiceAccount
    // resource in the target cluster's namespace.
    // +required
    ManagedServiceAccount string `json:"managedServiceAccount"`

    // ExpirationSeconds is the requested duration of validity of the token.
    // The token issuer may return a token with a different validity duration.
    // Defaults to 3600 seconds (1 hour) if not specified.
    // +optional
    // +kubebuilder:default=3600
    // +kubebuilder:validation:Minimum=600
    // +kubebuilder:validation:Maximum=86400
    ExpirationSeconds *int64 `json:"expirationSeconds,omitempty"`

    // Audiences are the intended audiences of the token. A recipient of
    // a token must identify itself with an identifier in the list of
    // audiences of the token, and otherwise should reject the token.
    // Defaults to the API server's default audiences if not specified.
    // +optional
    Audiences []string `json:"audiences,omitempty"`

    // BoundObjectRef is a reference to an object that the token will be bound to.
    // The token will only be valid for as long as the bound object exists.
    // This provides additional security by binding the token to a specific
    // workload (e.g., a Pod).
    // +optional
    BoundObjectRef *BoundObjectReference `json:"boundObjectRef,omitempty"`
}

// BoundObjectReference is a reference to an object that a token is bound to
type BoundObjectReference struct {
    // Kind of the referent
    // +required
    Kind string `json:"kind"`
    // API version of the referent
    // +optional
    APIVersion string `json:"apiVersion,omitempty"`
    // Name of the referent
    // +required
    Name string `json:"name"`
    // UID of the referent
    // +optional
    UID types.UID `json:"uid,omitempty"`
}

// ManagedServiceAccountTokenRequestStatus contains the token response
type ManagedServiceAccountTokenRequestStatus struct {
    // Token is the service account bearer token. This token can be used
    // to authenticate to the managed cluster's API server.
    // This field is only populated in the API response and is never persisted.
    // +required
    Token string `json:"token"`

    // ExpirationTimestamp is the time of expiration of the returned token.
    // +required
    ExpirationTimestamp metav1.Time `json:"expirationTimestamp"`
}
```

### Implementation Components

#### 1. Aggregated API Server

Create a new aggregated API server using `k8s.io/apiserver`:

```go
// pkg/apiserver/tokenrequest/apiserver.go
type TokenRequestAPIServer struct {
    GenericAPIServer *genericapiserver.GenericAPIServer
    ClusterProxyClientFactory ClusterProxyClientFactory
}

func (s *TokenRequestAPIServer) InstallAPIs() error {
    apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(
        authv1beta1.GroupVersion.Group,
        Scheme,
        metav1.ParameterCodec,
        Codecs,
    )

    storage := map[string]rest.Storage{
        "managedserviceaccounttokenrequests": &TokenRequestREST{
            clusterProxyFactory: s.ClusterProxyClientFactory,
        },
    }

    apiGroupInfo.VersionedResourcesStorageMap["v1beta1"] = storage
    return s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo)
}
```

#### 2. TokenRequest REST Handler

```go
// pkg/apiserver/tokenrequest/storage.go
type TokenRequestREST struct {
    clusterProxyFactory ClusterProxyClientFactory
}

// Create handles the token request
func (r *TokenRequestREST) Create(
    ctx context.Context,
    obj runtime.Object,
    createValidation rest.ValidateObjectFunc,
    options *metav1.CreateOptions,
) (runtime.Object, error) {
    tokenReq := obj.(*authv1beta1.ManagedServiceAccountTokenRequest)

    // Validate the request
    if err := r.validateTokenRequest(tokenReq); err != nil {
        return nil, err
    }

    // Extract target cluster and ManagedServiceAccount from spec
    targetCluster := tokenReq.Spec.TargetCluster
    msaName := tokenReq.Spec.ManagedServiceAccount

    // Get cluster-proxy client for the managed cluster
    managedClient, err := r.clusterProxyFactory.GetClient(targetCluster)
    if err != nil {
        return nil, fmt.Errorf("failed to connect to managed cluster %s: %w",
            targetCluster, err)
    }

    // Get the ManagedServiceAccount to find the spoke namespace
    msa := &authv1beta1.ManagedServiceAccount{}
    if err := r.clusterProxyFactory.GetHubClient().Get(ctx,
        client.ObjectKey{Namespace: targetCluster, Name: msaName}, msa); err != nil {
        return nil, err
    }

    // Determine the spoke namespace (could be from ManagedServiceAccount status)
    spokeNamespace := r.getSpokeNamespace(msa)

    // Create TokenRequest on the managed cluster
    tokenRequest := &authv1.TokenRequest{
        Spec: authv1.TokenRequestSpec{
            Audiences:         tokenReq.Spec.Audiences,
            ExpirationSeconds: tokenReq.Spec.ExpirationSeconds,
            BoundObjectRef:    convertBoundObjectRef(tokenReq.Spec.BoundObjectRef),
        },
    }

    result, err := managedClient.CoreV1().
        ServiceAccounts(spokeNamespace).
        CreateToken(ctx, msaName, tokenRequest, metav1.CreateOptions{})
    if err != nil {
        return nil, fmt.Errorf("failed to create token on managed cluster: %w", err)
    }

    // Populate the response status (ephemeral, not persisted)
    tokenReq.Status = authv1beta1.ManagedServiceAccountTokenRequestStatus{
        Token:               result.Status.Token,
        ExpirationTimestamp: result.Status.ExpirationTimestamp,
    }

    return tokenReq, nil
}

// New returns this as a create-only resource
func (r *TokenRequestREST) New() runtime.Object {
    return &authv1beta1.ManagedServiceAccountTokenRequest{}
}

func (r *TokenRequestREST) NamespaceScoped() bool {
    return true
}
```

#### 3. Cluster-Proxy Client Factory

```go
// pkg/apiserver/tokenrequest/proxy_client.go
type ClusterProxyClientFactory interface {
    GetClient(clusterName string) (kubernetes.Interface, error)
    GetHubClient() client.Client
}

type clusterProxyClientFactory struct {
    hubClient     client.Client
    proxyConfig   *rest.Config
}

func (f *clusterProxyClientFactory) GetClient(clusterName string) (kubernetes.Interface, error) {
    // Use cluster-proxy to get a client for the managed cluster
    // This leverages OCM's existing cluster-proxy infrastructure
    proxyPath := fmt.Sprintf("/apis/cluster.open-cluster-management.io/v1beta1/managedclusters/%s/proxy",
        clusterName)

    config := rest.CopyConfig(f.proxyConfig)
    config.Host = config.Host + proxyPath

    return kubernetes.NewForConfig(config)
}
```

#### 4. APIService Registration

```yaml
# deploy/apiservice.yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.authentication.open-cluster-management.io
spec:
  group: authentication.open-cluster-management.io
  version: v1beta1
  service:
    name: managed-serviceaccount-apiserver
    namespace: open-cluster-management-addon
  groupPriorityMinimum: 1000
  versionPriority: 15
  insecureSkipTLSVerify: false
```

### RBAC Configuration

#### Simple: ClusterRole for All Clusters

```yaml
# Grant permission to create token requests for any cluster
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: managedserviceaccount-token-requester
rules:
- apiGroups: ["authentication.open-cluster-management.io"]
  resources: ["managedserviceaccounttokenrequests"]
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: my-app-token-requester
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: managedserviceaccount-token-requester
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: my-app
```

This allows the app to create TokenRequests in its own namespace for any target cluster.

#### Namespace-Scoped: Role for Specific Namespace

```yaml
# Grant permission only in user's namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: managedserviceaccount-token-requester
  namespace: my-app  # User's namespace
rules:
- apiGroups: ["authentication.open-cluster-management.io"]
  resources: ["managedserviceaccounttokenrequests"]
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-token-requester
  namespace: my-app
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: managedserviceaccount-token-requester
subjects:
- kind: ServiceAccount
  name: my-app
  namespace: my-app
```

This restricts TokenRequest creation to the specific namespace. Combined with ManagedClusterSetBinding (see [ManagedClusterSet Integration](#managedclusterset-integration)), this provides multi-tenant cluster access control.

#### Aggregated API Server Permissions

```yaml
# Aggregated API server needs permission to read ManagedServiceAccounts and access cluster-proxy
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: managedserviceaccount-apiserver
rules:
- apiGroups: ["authentication.open-cluster-management.io"]
  resources: ["managedserviceaccounts"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["cluster.open-cluster-management.io"]
  resources: ["managedclusters"]
  verbs: ["get"]
- apiGroups: ["cluster.open-cluster-management.io"]
  resources: ["managedclusters/proxy"]
  verbs: ["get", "create"]
- apiGroups: ["cluster.open-cluster-management.io"]
  resources: ["managedclustersetbindings"]
  verbs: ["get", "list"]
```

### User Experience

#### Creating a Token Request

```bash
# Create a token request for a ManagedServiceAccount
kubectl create -f - <<EOF
apiVersion: authentication.open-cluster-management.io/v1beta1
kind: ManagedServiceAccountTokenRequest
metadata:
  name: get-cluster1-token
  namespace: my-app  # User's namespace (not cluster1!)
spec:
  targetCluster: cluster1
  managedServiceAccount: my-sample
  expirationSeconds: 3600
  audiences:
  - "https://kubernetes.default.svc"
EOF
```

#### Response

```yaml
apiVersion: authentication.open-cluster-management.io/v1beta1
kind: ManagedServiceAccountTokenRequest
metadata:
  name: get-cluster1-token
  namespace: my-app
  creationTimestamp: "2026-02-25T14:30:00Z"
spec:
  targetCluster: cluster1
  managedServiceAccount: my-sample
  expirationSeconds: 3600
  audiences:
  - "https://kubernetes.default.svc"
status:
  token: "eyJhbGciOiJSUzI1NiIsImtpZCI6IjEyMzQ1Njc4OTAifQ..."
  expirationTimestamp: "2026-02-25T15:30:00Z"
```

#### Using the Token

```bash
# Extract the token
TOKEN=$(kubectl create -f tokenrequest.yaml -o jsonpath='{.status.token}')

# Use the token to access the managed cluster via cluster-proxy
kubectl --token="${TOKEN}" \
  --server="https://hub-api/apis/cluster.open-cluster-management.io/v1beta1/managedclusters/cluster1/proxy" \
  get pods
```

#### Client-side Token Caching (Optional)

Clients can implement caching to avoid repeated token requests:

```go
type TokenCache struct {
    tokens map[string]*CachedToken
    mu     sync.RWMutex
}

type CachedToken struct {
    Token      string
    Expiration time.Time
}

func (c *TokenCache) GetOrRequestToken(
    ctx context.Context,
    client kubernetes.Interface,
    namespace, targetCluster, msaName string,
) (string, error) {
    c.mu.RLock()
    cached, exists := c.tokens[targetCluster+"/"+msaName]
    c.mu.RUnlock()

    // Return cached token if valid for at least 5 more minutes
    if exists && time.Until(cached.Expiration) > 5*time.Minute {
        return cached.Token, nil
    }

    // Request new token
    tokenReq := &authv1beta1.ManagedServiceAccountTokenRequest{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "get-token",
            Namespace: namespace,
        },
        Spec: authv1beta1.ManagedServiceAccountTokenRequestSpec{
            TargetCluster:         targetCluster,
            ManagedServiceAccount: msaName,
            ExpirationSeconds:     ptr.To(int64(3600)),
        },
    }

    result, err := client.AuthenticationV1beta1().
        ManagedServiceAccountTokenRequests(namespace).
        Create(ctx, tokenReq, metav1.CreateOptions{})
    if err != nil {
        return "", err
    }

    c.mu.Lock()
    c.tokens[targetCluster+"/"+msaName] = &CachedToken{
        Token:      result.Status.Token,
        Expiration: result.Status.ExpirationTimestamp.Time,
    }
    c.mu.Unlock()

    return result.Status.Token, nil
}
```

---

## ManagedClusterSet Integration

### Problem Statement

When an application needs to access multiple managed clusters (e.g., 100 clusters), we face RBAC complexity:

**Without ManagedClusterSet integration:**
- Option A: ClusterRole - grants access to ALL clusters (overly broad)
- Option B: 100 namespace-scoped Roles - management overhead

**Goal:** Provide namespace-scoped access to a **subset of clusters** using OCM's existing ManagedClusterSet/ManagedClusterSetBinding mechanism.

### OCM ManagedClusterSet Concepts

#### ManagedClusterSet

A **ManagedClusterSet** is a cluster-scoped resource that defines a group of ManagedClusters:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSet
metadata:
  name: production
spec:
  clusterSelector:
    selectorType: LabelSelector
    labelSelector:
      matchLabels:
        environment: production
```

ManagedClusters are assigned to a ManagedClusterSet via label:
```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: cluster1
  labels:
    cluster.open-cluster-management.io/clusterset: production
    environment: production
```

#### ManagedClusterSetBinding

A **ManagedClusterSetBinding** projects a ManagedClusterSet into a namespace:

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSetBinding
metadata:
  name: production
  namespace: my-app  # App namespace
spec:
  clusterSet: production
```

**Key Insight:** Applications in namespace `my-app` can now work with clusters from the `production` ManagedClusterSet.

### Proposed Integration

#### Design Principle

**Leverage ManagedClusterSetBinding for authorization:**
- If a ManagedClusterSetBinding exists in namespace N binding ManagedClusterSet S
- And user U has permission to CREATE TokenRequest in namespace N
- Then user U can create ManagedServiceAccountTokenRequests for clusters in set S from namespace N

#### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Hub Cluster                                │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ ManagedClusterSet: production                            │  │
│  │ - cluster1, cluster2, ..., cluster100                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              │ bound to namespace                │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ ManagedClusterSetBinding                                 │  │
│  │   namespace: my-app                                      │  │
│  │   clusterSet: production                                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              │ grants access                     │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ App (ServiceAccount: my-app/my-app-sa)                   │  │
│  │                                                           │  │
│  │ RBAC: Can CREATE ManagedServiceAccountTokenRequest       │  │
│  │       in namespace "my-app"                              │  │
│  │                                                           │  │
│  │ Creates TokenRequest in namespace "my-app"               │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                            │
│                     ▼                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ TokenRequest API Server                                  │  │
│  │                                                           │  │
│  │ Authorization Checks:                                    │  │
│  │ 1. ✅ Standard RBAC: Can user CREATE TokenRequest       │  │
│  │    in namespace "my-app"? (checked by k8s API server)   │  │
│  │ 2. ✅ Is cluster1 in ManagedClusterSet "production"?    │  │
│  │ 3. ✅ Is "production" bound to namespace "my-app"?      │  │
│  │ 4. ✅ Is the binding status "Bound"?                    │  │
│  │                                                           │  │
│  │ ✅ All checks pass → Generate token                      │  │
│  │ ❌ Any check fails → Deny request                        │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation

#### 1. Enhanced TokenRequest API Server

The ManagedServiceAccountTokenRequest aggregated API server validates access based on ManagedClusterSetBindings:

```go
// pkg/apiserver/tokenrequest/authorization.go
package tokenrequest

import (
    "context"
    "fmt"

    "k8s.io/apimachinery/pkg/labels"
    clusterv1 "open-cluster-management.io/api/cluster/v1"
    clusterv1beta2 "open-cluster-management.io/api/cluster/v1beta2"
    clusterclient "open-cluster-management.io/api/client/cluster/clientset/versioned"
    authv1beta1 "open-cluster-management.io/managed-serviceaccount/apis/authentication/v1beta1"
)

type ClusterSetAuthorizer struct {
    clusterClient clusterclient.Interface
}

// AuthorizeTokenRequest checks if the user can create a TokenRequest for the target cluster
// based on ManagedClusterSetBindings
func (a *ClusterSetAuthorizer) AuthorizeTokenRequest(
    ctx context.Context,
    userNamespace string, // Namespace where user is operating from
    targetCluster string, // Cluster for which token is requested
) error {
    // 1. Get the target ManagedCluster
    managedCluster, err := a.clusterClient.ClusterV1().
        ManagedClusters().
        Get(ctx, targetCluster, metav1.GetOptions{})
    if err != nil {
        return fmt.Errorf("failed to get managed cluster %s: %w", targetCluster, err)
    }

    // 2. Determine which ManagedClusterSet this cluster belongs to
    clusterSetName, ok := managedCluster.Labels[clusterv1beta2.ClusterSetLabel]
    if !ok {
        // Cluster not in any ManagedClusterSet - fall back to standard RBAC
        return nil
    }

    // 3. Check if there's a ManagedClusterSetBinding in the user's namespace
    binding, err := a.clusterClient.ClusterV1beta2().
        ManagedClusterSetBindings(userNamespace).
        Get(ctx, clusterSetName, metav1.GetOptions{})
    if err != nil {
        if apierrors.IsNotFound(err) {
            return fmt.Errorf("no ManagedClusterSetBinding for clusterset %s in namespace %s",
                clusterSetName, userNamespace)
        }
        return fmt.Errorf("failed to get clusterset binding: %w", err)
    }

    // 4. Verify the binding is valid
    if binding.Spec.ClusterSet != clusterSetName {
        return fmt.Errorf("clusterset binding mismatch")
    }

    // 5. Check binding status is Bound
    if !isBindingBound(binding) {
        return fmt.Errorf("clusterset binding %s is not bound", binding.Name)
    }

    // Authorization successful
    return nil
}

func isBindingBound(binding *clusterv1beta2.ManagedClusterSetBinding) bool {
    for _, cond := range binding.Status.Conditions {
        if cond.Type == clusterv1beta2.ClusterSetBindingBoundType && cond.Status == metav1.ConditionTrue {
            return true
        }
    }
    return false
}
```

#### 2. TokenRequest REST Handler with ClusterSet Authorization

```go
// pkg/apiserver/tokenrequest/storage.go
type TokenRequestREST struct {
    clusterProxyFactory ClusterProxyClientFactory
    clusterSetAuthorizer *ClusterSetAuthorizer
}

func (r *TokenRequestREST) Create(
    ctx context.Context,
    obj runtime.Object,
    createValidation rest.ValidateObjectFunc,
    options *metav1.CreateOptions,
) (runtime.Object, error) {
    tokenReq := obj.(*authv1beta1.ManagedServiceAccountTokenRequest)

    // Get the namespace from which the request originated
    requestNamespace := tokenReq.Namespace

    // Target cluster from spec
    targetCluster := tokenReq.Spec.TargetCluster

    // Authorize based on ManagedClusterSetBinding
    if err := r.clusterSetAuthorizer.AuthorizeTokenRequest(ctx, requestNamespace, targetCluster); err != nil {
        return nil, errors.NewForbidden(
            authv1beta1.Resource("managedserviceaccounttokenrequests"),
            tokenReq.Name,
            fmt.Errorf("not authorized to request token for cluster %s: %w", targetCluster, err),
        )
    }

    // Continue with token generation (existing code)
    // ...
}
```

#### 3. RBAC Configuration

**Simple RBAC for applications:**

```yaml
# Grant permission to CREATE ManagedServiceAccountTokenRequests in the app namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tokenrequest-creator
  namespace: my-app  # App namespace
rules:
- apiGroups: ["authentication.open-cluster-management.io"]
  resources: ["managedserviceaccounttokenrequests"]
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-tokenrequest
  namespace: my-app
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: tokenrequest-creator
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: my-app
```

**ManagedClusterSetBinding RBAC (for binding creator):**

```yaml
# Grant permission to create ManagedClusterSetBindings
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: clusterset-binder
rules:
- apiGroups: ["cluster.open-cluster-management.io"]
  resources: ["managedclustersets/bind"]
  resourceNames: ["production"]  # Specific clustersets
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-clusterset-binder
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: clusterset-binder
subjects:
- kind: User
  name: admin
  apiGroup: rbac.authorization.k8s.io
```

### User Workflow

#### Setup (One-time, by Admin)

```bash
# 1. Create ManagedClusterSet
kubectl apply -f - <<EOF
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSet
metadata:
  name: production
spec:
  clusterSelector:
    selectorType: LabelSelector
    labelSelector:
      matchLabels:
        environment: production
EOF

# 2. Label managed clusters to join the set
kubectl label managedcluster cluster1 cluster.open-cluster-management.io/clusterset=production
kubectl label managedcluster cluster2 cluster.open-cluster-management.io/clusterset=production
# ... cluster100

# 3. Create ManagedClusterSetBinding in app namespace
kubectl apply -f - <<EOF
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSetBinding
metadata:
  name: production
  namespace: my-app
spec:
  clusterSet: production
EOF

# 4. Grant app permission to create TokenRequests in its namespace
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tokenrequest-creator
  namespace: my-app
rules:
- apiGroups: ["authentication.open-cluster-management.io"]
  resources: ["managedserviceaccounttokenrequests"]
  verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-tokenrequest
  namespace: my-app
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: tokenrequest-creator
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: my-app
EOF
```

#### Application Usage

```go
// Application code - requests token for any cluster in the bound ManagedClusterSet
func (app *MultiClusterApp) GetToken(ctx context.Context, clusterName string) (string, error) {
    // Create TokenRequest in app's own namespace
    tokenReq := &authv1beta1.ManagedServiceAccountTokenRequest{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "get-token-" + clusterName,
            Namespace: "my-app",  // App's namespace
        },
        Spec: authv1beta1.ManagedServiceAccountTokenRequestSpec{
            TargetCluster:         clusterName,
            ManagedServiceAccount: "my-serviceaccount",
            ExpirationSeconds:     ptr.To(int64(3600)),
        },
    }

    // Standard Kubernetes API call
    result, err := app.authClient.AuthenticationV1beta1().
        ManagedServiceAccountTokenRequests("my-app").
        Create(ctx, tokenReq, metav1.CreateOptions{})
    if err != nil {
        return "", err
    }

    return result.Status.Token, nil
}
```

### Benefits of ManagedClusterSet Integration

| Aspect | Without ManagedClusterSet | With ManagedClusterSet |
|--------|---------------------------|------------------------|
| **RBAC Resources** | 100 Roles + 100 RoleBindings OR 1 overly-broad ClusterRole | 1 Role + 1 RoleBinding in app namespace |
| **Cluster Access Control** | Per-cluster RBAC or all-or-nothing | Group-based via ManagedClusterSet |
| **Namespace Scoping** | N/A | Natural namespace isolation via binding |
| **Onboarding New Cluster** | Update RBAC | Just label the cluster to join set |
| **Removing Cluster Access** | Update RBAC | Remove label from cluster |
| **Multi-tenant Isolation** | Complex RBAC | ManagedClusterSetBinding per namespace |
| **Auditing** | RBAC audit | Clusterset binding + token request audit |

### Example: Multi-tenant Scenario

**Scenario:** Two teams, each accessing different clusters:

```yaml
# Production team
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSet
metadata:
  name: production
---
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSetBinding
metadata:
  name: production
  namespace: team-prod
spec:
  clusterSet: production
---
# Development team
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSet
metadata:
  name: development
---
apiVersion: cluster.open-cluster-management.io/v1beta2
kind: ManagedClusterSetBinding
metadata:
  name: development
  namespace: team-dev
spec:
  clusterSet: development
```

**Result:**
- Team-prod apps can only request tokens for clusters in "production" set
- Team-dev apps can only request tokens for clusters in "development" set
- Each team operates independently in their own namespace
- No cross-team access, enforced by ManagedClusterSetBinding

---

## ClusterProfile Plugin Migration

### Overview

This section describes the necessary changes to the **clusterprofile-credentials-plugin** ([cmd/clusterprofile-credentials-plugin/main.go](../../cmd/clusterprofile-credentials-plugin/main.go)) to support the new ManagedServiceAccountTokenRequest API.

This plugin is the primary consumer of the ClusterProfile credential sync mechanism and demonstrates the migration pattern for moving from Secret-based token access to on-demand TokenRequest API.

### Current Implementation

#### What the Plugin Does

The clusterprofile-credentials-plugin is a Kubernetes client-go credential exec plugin that:
- Implements the `client.authentication.k8s.io/v1` ExecCredential protocol
- Integrates with ClusterProfile credential flow
- Reads synced token secrets from the ClusterProfile namespace
- Returns service account tokens to authenticate to spoke clusters via cluster-proxy
- Is invoked by kubectl/controllers when accessing spoke clusters through ClusterProfile

#### Current Code

```go
// Retrieve the synced token secret from clusterprofile namespace
namespace := inferNamespace()
tokenSecretName := fmt.Sprintf("%s-%s", cfg.ClusterName, p.ManagedServiceAccount)
secret, err := p.KubeClient.CoreV1().Secrets(namespace).Get(ctx, tokenSecretName, metav1.GetOptions{})
if err != nil {
    return clientauthenticationv1.ExecCredentialStatus{}, fmt.Errorf("failed to get synced credential secret %s/%s: %w", namespace, tokenSecretName, err)
}

tokenData, ok := secret.Data[corev1.ServiceAccountTokenKey]
if !ok || len(tokenData) == 0 {
    return clientauthenticationv1.ExecCredentialStatus{}, fmt.Errorf("secret %s/%s missing or empty %q key", namespace, tokenSecretName, corev1.ServiceAccountTokenKey)
}

return clientauthenticationv1.ExecCredentialStatus{Token: string(tokenData)}, nil
```

#### Problems with Current Approach

1. **Depends on persistent secrets**: Requires ClusterProfileCredSyncer to sync long-lived tokens as Secrets
2. **Long-lived credentials**: Returns 360-day tokens, increasing security risk
3. **No expiration handling**: Plugin has no awareness of token expiration
4. **Defeats TokenRequest benefits**: Even if we switch to TokenRequest API elsewhere, this plugin still uses persistent secrets

### Proposed Changes

Replace secret reading with on-demand token generation using the ManagedServiceAccountTokenRequest API.

#### Updated Code

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "os"
    "sync"
    "time"

    "github.com/spf13/pflag"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    kubernetes "k8s.io/client-go/kubernetes"
    clientauthenticationv1 "k8s.io/client-go/pkg/apis/clientauthentication/v1"
    "k8s.io/client-go/rest"
    "k8s.io/client-go/tools/clientcmd"
    "k8s.io/utils/ptr"

    authv1beta1 "open-cluster-management.io/managed-serviceaccount/apis/authentication/v1beta1"
    authclient "open-cluster-management.io/managed-serviceaccount/pkg/generated/clientset/versioned"
    "open-cluster-management.io/managed-serviceaccount/pkg/addon/manager/controller"
    "sigs.k8s.io/cluster-inventory-api/pkg/credentialplugin"
)

type Provider struct {
    KubeClient kubernetes.Interface
    AuthClient authclient.Interface
    ManagedServiceAccount string
    TokenCache *TokenCache
}

type TokenCache struct {
    tokens map[string]*CachedToken
    mu     sync.RWMutex
}

type CachedToken struct {
    Token      string
    Expiration time.Time
}

func NewTokenCache() *TokenCache {
    return &TokenCache{
        tokens: make(map[string]*CachedToken),
    }
}

func (p Provider) GetToken(ctx context.Context, info clientauthenticationv1.ExecCredential) (clientauthenticationv1.ExecCredentialStatus, error) {
    // Extract clusterName from ExecCredential
    type execClusterConfig struct {
        ClusterName string `json:"clusterName"`
    }

    if info.Spec.Cluster == nil || len(info.Spec.Cluster.Config.Raw) == 0 {
        return clientauthenticationv1.ExecCredentialStatus{}, fmt.Errorf("missing ExecCredential.Spec.Cluster.Config")
    }

    var cfg execClusterConfig
    if err := json.Unmarshal(info.Spec.Cluster.Config.Raw, &cfg); err != nil {
        return clientauthenticationv1.ExecCredentialStatus{}, fmt.Errorf("invalid ExecCredential.Spec.Cluster.Config: %w", err)
    }

    // Check cache for valid token
    cacheKey := fmt.Sprintf("%s/%s", cfg.ClusterName, p.ManagedServiceAccount)
    if token := p.TokenCache.Get(cacheKey); token != "" {
        return clientauthenticationv1.ExecCredentialStatus{Token: token}, nil
    }

    // Determine the namespace for TokenRequest creation
    namespace := inferNamespace()

    // Request a new token using ManagedServiceAccountTokenRequest API
    tokenReq := &authv1beta1.ManagedServiceAccountTokenRequest{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("token-%s-%s", cfg.ClusterName, p.ManagedServiceAccount),
            Namespace: namespace,
        },
        Spec: authv1beta1.ManagedServiceAccountTokenRequestSpec{
            TargetCluster:         cfg.ClusterName,
            ManagedServiceAccount: p.ManagedServiceAccount,
            ExpirationSeconds:     ptr.To(int64(3600)),
        },
    }

    result, err := p.AuthClient.AuthenticationV1beta1().
        ManagedServiceAccountTokenRequests(namespace).
        Create(ctx, tokenReq, metav1.CreateOptions{})
    if err != nil {
        return clientauthenticationv1.ExecCredentialStatus{}, fmt.Errorf("failed to request token for cluster %s: %w", cfg.ClusterName, err)
    }

    // Cache the token
    p.TokenCache.Set(cacheKey, result.Status.Token, result.Status.ExpirationTimestamp.Time)

    // Return the token with expiration timestamp
    return clientauthenticationv1.ExecCredentialStatus{
        Token:               result.Status.Token,
        ExpirationTimestamp: &result.Status.ExpirationTimestamp,
    }, nil
}

func (c *TokenCache) Get(key string) string {
    c.mu.RLock()
    defer c.mu.RUnlock()

    cached, exists := c.tokens[key]
    if !exists {
        return ""
    }

    // Return token if valid for at least 5 more minutes
    if time.Until(cached.Expiration) > 5*time.Minute {
        return cached.Token
    }

    return ""
}

func (c *TokenCache) Set(key, token string, expiration time.Time) {
    c.mu.Lock()
    defer c.mu.Unlock()

    c.tokens[key] = &CachedToken{
        Token:      token,
        Expiration: expiration,
    }
}
```

### Key Changes

1. **Add AuthClient**: Client for ManagedServiceAccount APIs
2. **Token Caching**: Cache tokens for their validity period minus 5 minutes
3. **TokenRequest API Call**: Replace secret reading with API call
4. **ExpirationTimestamp Support**: Return expiration timestamp for automatic refresh

### Benefits of New Implementation

#### Security Improvements

✅ **Short-lived tokens**: 1-hour validity instead of 360 days
✅ **No persistent storage**: Tokens never stored as Secrets
✅ **Automatic expiration**: Client-side cache respects token lifetime
✅ **Reduced attack surface**: Compromised plugin binary doesn't expose long-lived credentials

#### Operational Improvements

✅ **Client-side caching**: Reduces API call overhead (1 call per hour)
✅ **Expiration awareness**: Plugin knows when token expires and refreshes automatically
✅ **Better error handling**: Clear errors if TokenRequest API is unavailable
✅ **Alignment with TokenRequest API**: Uses same API as other consumers

### RBAC Requirements

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: clusterprofile-plugin-tokenrequest
  namespace: <clusterprofile-namespace>
rules:
- apiGroups: ["authentication.open-cluster-management.io"]
  resources: ["managedserviceaccounttokenrequests"]
  verbs: ["create"]
```

---

## Test Plan

### Unit Tests

1. **API Validation Tests**
   - Validate token request spec constraints (expiration bounds, required fields)
   - Test bound object reference validation
   - Test audience validation

2. **REST Handler Tests**
   - Mock cluster-proxy client and test token generation
   - Test error handling (cluster unreachable, ServiceAccount not found)
   - Test targetCluster/managedServiceAccount field processing

3. **ClusterSet Authorization Tests**
   - Test authorization with valid ManagedClusterSetBinding
   - Test denial when binding doesn't exist
   - Test denial when binding is not "Bound"
   - Test fallback to standard RBAC when cluster not in any set

4. **Client Factory Tests**
   - Test cluster-proxy client creation
   - Test connection pooling and reuse
   - Test error scenarios

### Integration Tests

1. **E2E Token Request Flow**
   - Create ManagedServiceAccount on hub
   - Wait for ServiceAccount creation on managed cluster
   - Request token via TokenRequest API
   - Verify token is valid and can authenticate to managed cluster
   - Verify token expires after specified duration

2. **RBAC Tests**
   - Verify users without permission cannot create token requests
   - Verify token requests are scoped to namespace
   - Verify aggregated API server has required permissions

3. **ManagedClusterSet Integration Tests**
   - Create ManagedClusterSet and bind to namespace
   - Verify TokenRequest succeeds for clusters in the set
   - Verify TokenRequest fails for clusters outside the set
   - Verify TokenRequest fails when binding is removed

4. **Cluster-Proxy Integration**
   - Test token request when cluster-proxy is healthy
   - Test error handling when cluster-proxy is unavailable
   - Test token request to multiple clusters in parallel

5. **Token Features**
   - Test custom audiences
   - Test custom expiration times (min/max bounds)
   - Test bound object references (if managed cluster supports it)

6. **ClusterProfile Plugin Tests**
   - Test plugin with TokenRequest API
   - Test client-side token caching
   - Test automatic token refresh

### Performance Tests

1. **Latency Tests**
   - Measure token request latency (hub → cluster-proxy → managed cluster)
   - Test concurrent token requests
   - Identify bottlenecks in aggregated API server

2. **Scale Tests**
   - Test token requests across 100+ managed clusters
   - Test multiple clients requesting tokens concurrently
   - Measure API server resource usage under load

---

## Risks and Mitigations

### Risk: Cluster-Proxy Dependency

**Risk**: Token requests fail if cluster-proxy is unavailable.

**Mitigation**:
- Cluster-proxy is already core OCM infrastructure
- Document cluster-proxy as prerequisite
- Provide clear error messages when cluster-proxy is unreachable
- Consider fallback to Secret-based approach during migration period

### Risk: Increased Network Latency

**Risk**: On-demand token generation requires network calls, adding latency.

**Mitigation**:
- Tokens are short-lived but long enough (default 1 hour) for client-side caching
- Provide client library with built-in caching
- Document caching best practices
- Acceptable tradeoff for improved security

### Risk: Token Request Rate Limiting

**Risk**: Managed cluster API server rate limits could throttle token requests.

**Mitigation**:
- Default 1-hour expiration allows aggressive client-side caching
- Monitor token request rates with metrics
- Implement hub-side rate limiting if needed
- Use cluster-proxy connection pooling

### Risk: Breaking Changes for Existing Users

**Risk**: Users relying on Secret-based tokens may be disrupted.

**Mitigation**:
- Phased rollout with feature gates
- Maintain backward compatibility during migration period
- Clear migration documentation with examples
- Deprecation warnings before removal

---

## Security Considerations

### Token Lifetime

- Default 1-hour expiration balances security and usability
- Minimum 10 minutes to prevent excessive request load
- Maximum 24 hours to limit exposure window
- Users can configure based on their security requirements

### Token Scope

- Tokens inherit ServiceAccount permissions on managed cluster
- Audience validation ensures tokens used for intended purpose
- Bound object references provide workload-specific tokens
- No elevation of privileges beyond ServiceAccount RBAC

### Hub Cluster Security

- No persistent token storage on hub (ephemeral only)
- RBAC controls who can request tokens
- Audit logging captures all token requests
- Token requests are namespace-scoped

### Network Security

- All communication over cluster-proxy (encrypted)
- Relies on cluster-proxy's mTLS authentication
- No tokens transmitted in clear text
- Token only appears in API response (not persisted)

### ManagedClusterSet Security

- ManagedClusterSetBinding creation requires special `managedclustersets/bind` permission
- Binding grants namespace-level access, not per-user access
- Standard RBAC still applies for TokenRequest creation
- Multi-tenant isolation via namespace boundaries

---

## Observability

### Metrics

```go
// Token request metrics
tokenRequestTotal = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "managedserviceaccount_token_requests_total",
        Help: "Total number of token requests",
    },
    []string{"cluster", "serviceaccount", "result"},
)

tokenRequestDuration = prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name: "managedserviceaccount_token_request_duration_seconds",
        Help: "Token request duration in seconds",
    },
    []string{"cluster"},
)

tokenRequestErrors = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "managedserviceaccount_token_request_errors_total",
        Help: "Total number of token request errors",
    },
    []string{"cluster", "error_type"},
)
```

### Logging

```go
// Structured logging for token requests
logger.Info("token request received",
    "cluster", clusterName,
    "serviceaccount", saName,
    "audiences", audiences,
    "expiration", expirationSeconds,
)

logger.Info("token generated successfully",
    "cluster", clusterName,
    "serviceaccount", saName,
    "expiration_time", expirationTimestamp,
    "duration_ms", duration.Milliseconds(),
)

logger.Error(err, "token request failed",
    "cluster", clusterName,
    "serviceaccount", saName,
    "error_type", errorType,
)
```

---

## Migration Path

### Phase 1: Introduction (v0.11.0)

1. Introduce `ManagedServiceAccountTokenRequest` API as v1beta1
2. Deploy aggregated API server alongside existing addon manager
3. Implement ManagedClusterSet integration
4. Keep existing Secret-based synchronization as default behavior
5. Document new API and provide examples
6. Update ClusterProfile plugin to support both modes

### Phase 2: Adoption (v0.12.0)

1. Add feature gate `EphemeralTokenRequest` (default: false)
2. When enabled, skip Secret creation in addon agent
3. Provide migration guide for users
4. Add metrics for token request usage
5. Enable TokenRequest API by default for ClusterProfile plugin

### Phase 3: Deprecation (v0.13.0)

1. Enable `EphemeralTokenRequest` feature gate by default
2. Mark Secret-based synchronization as deprecated
3. Add deprecation warnings in logs
4. Update documentation to recommend TokenRequest API

### Phase 4: Removal (v1.0.0)

1. Remove Secret synchronization code from addon agent
2. Remove `EphemeralTokenRequest` feature gate
3. Remove secret-based code path from ClusterProfile plugin
4. Clean up deprecated code paths

---

## Future Enhancements

1. **Token Refresh API**: Extend existing token before expiration
2. **Batch Token Requests**: Request tokens for multiple ServiceAccounts in one call
3. **Token Revocation**: Explicit revocation mechanism for compromised tokens
4. **Token Usage Tracking**: Track which tokens are actively used vs requested but unused
5. **Automatic Audience Detection**: Infer audiences based on managed cluster configuration
6. **Federation with External IDPs**: Allow OIDC/SAML integration for token issuance

---

## Alternatives Considered

### Alternative 1: Hub-side Token Generation with Private Keys

**Approach**: Store managed cluster ServiceAccount signing keys on hub and generate tokens locally.

**Pros**:
- No network latency for token generation
- Works offline without cluster-proxy

**Cons**:
- Requires storing private keys on hub (even worse security risk)
- Key rotation complexity
- Violates security principle of key isolation
- Not viable for production

### Alternative 2: Service Mesh with mTLS

**Approach**: Use service mesh for managed cluster authentication.

**Pros**:
- Strong cryptographic authentication
- Automatic certificate rotation
- No token storage

**Cons**:
- Requires service mesh installation (significant infrastructure)
- Not all managed clusters may support service mesh
- Adds operational complexity
- Outside scope of OCM addon

### Alternative 3: OIDC Workload Identity Federation

**Approach**: Federate authentication to external OIDC provider.

**Pros**:
- Industry-standard approach
- Centralized identity management
- No token storage on hub

**Cons**:
- Requires external OIDC provider
- Complex setup for managed clusters
- Not all environments have OIDC infrastructure
- May not work in air-gapped environments

### Alternative 4: Keep Current Secret-based Approach

**Approach**: Continue storing long-lived tokens as Secrets.

**Pros**:
- No implementation work
- Proven to work
- No performance impact

**Cons**:
- Significant security risks (as outlined in Motivation)
- Doesn't align with security best practices
- Accumulates technical debt
- Not acceptable for security-conscious environments

---

## Conclusion

The `ManagedServiceAccountTokenRequest` API provides a secure, scalable, and backward-compatible enhancement to the ManagedServiceAccount addon. By generating short-lived tokens on-demand via cluster-proxy, it eliminates the security risks of persistent token storage while leveraging existing OCM infrastructure.

**Key Benefits:**

✅ **Enhanced Security**: Short-lived tokens (1 hour) instead of 360-day tokens
✅ **No Persistent Storage**: Tokens never stored as Secrets on hub
✅ **Simplified RBAC**: ManagedClusterSet integration reduces RBAC complexity from 100 Roles to 1
✅ **Multi-tenant Support**: Namespace-scoped access via ManagedClusterSetBinding
✅ **Backward Compatible**: Phased migration path with feature gates
✅ **OCM Aligned**: Leverages existing cluster-proxy and ManagedClusterSet concepts
✅ **Client-side Caching**: Reduces API overhead for high-frequency access

This enhancement aligns with Kubernetes security best practices, improves the security posture of multi-cluster environments, and provides a clear migration path for existing users including the ClusterProfile credentials plugin.
