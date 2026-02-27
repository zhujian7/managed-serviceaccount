# Enhancement: Transparent Multi-cluster Authentication via ServiceAccountMapping

## Release Signoff Checklist

- [ ] Enhancement issue in release milestone, which links to pull request in [enhancements repository]
- [ ] Core members have approved the issue with the label `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place
- [ ] User-facing documentation has been created

## Summary

This enhancement proposes a **ServiceAccountMapping** API that provides transparent authentication for hub components accessing managed clusters. By mapping hub service accounts to managed cluster service accounts, applications can access managed clusters via cluster-proxy without managing tokens or requiring extensive RBAC permissions.

This is an **alternative approach** to the [ManagedServiceAccountTokenRequest API](token-request-api.md).

## Table of Contents

- [Motivation](#motivation)
- [Goals and Non-Goals](#goals-and-non-goals)
- [Proposal](#proposal)
- [API Design](#api-design)
- [Implementation](#implementation)
- [User Experience](#user-experience)
- [Comparison with TokenRequest API](#comparison-with-tokenrequest-api)
- [Test Plan](#test-plan)
- [Risks and Mitigations](#risks-and-mitigations)
- [Security Considerations](#security-considerations)
- [Migration Path](#migration-path)
- [Alternatives Considered](#alternatives-considered)

## Motivation

### Problems with Current Architecture

The existing ManagedServiceAccount implementation has two main problems:

1. **Token Storage Problem**: Synchronizing service account tokens from managed clusters to the hub as Secrets creates:
   - Centralized attack surface (all tokens stored on hub)
   - Long-lived credentials (360-day tokens)
   - Secret sprawl (one Secret per ManagedServiceAccount)

2. **RBAC Complexity Problem**: When a hub component (e.g., ArgoCD, ACM controllers) needs to access managed clusters:
   - Must grant Secret read permissions for each cluster namespace
   - Example: ArgoCD accessing 100 clusters needs Secret read in 100 namespaces
   - Creates overly broad permissions or complex RBAC management

### Problems with TokenRequest API Alternative

The [ManagedServiceAccountTokenRequest API](token-request-api.md) solves problem #1 (token storage) but partially addresses problem #2:
- Apps still need permission to CREATE TokenRequest resources
- Apps must implement token management logic (request, cache, refresh)
- Requires code changes in all consuming applications
- Not transparent to existing applications

### What This Enhancement Addresses

This enhancement provides **transparent authentication** that solves both problems:

1. ✅ **No token storage**: Tokens generated on-demand, never persisted on hub
2. ✅ **Simple RBAC**: Apps only need their own ServiceAccount, no additional permissions
3. ✅ **Zero code changes**: Existing apps work without modification
4. ✅ **Automatic provisioning**: ManagedServiceAccounts created automatically for bound clusters
5. ✅ **Centralized management**: One mapping resource per app, not per cluster

### Use Cases

1. **ArgoCD Multi-cluster Deployment**: ArgoCD hub component accessing 100 managed clusters
   - Current: Needs Secret read in 100 cluster namespaces
   - With Mapping: Just uses its own ServiceAccount, zero permission changes

2. **Multi-cluster Operators**: Operators running on hub managing workloads on spokes
   - Current: Complex RBAC + token management code
   - With Mapping: Transparent authentication via cluster-proxy

3. **CI/CD Pipelines**: Automation systems accessing multiple clusters
   - Current: Manage secrets or implement TokenRequest client
   - With Mapping: Use standard ServiceAccount, access via cluster-proxy

4. **Observability Systems**: Prometheus/monitoring accessing multiple clusters
   - Current: Store tokens or manage token requests
   - With Mapping: Transparent authentication

## Goals and Non-Goals

### Goals

- Provide transparent authentication for hub components accessing managed clusters
- Eliminate need for hub components to manage tokens
- Simplify RBAC by using only hub ServiceAccount permissions
- Automatically provision ManagedServiceAccounts on bound clusters
- Leverage cluster-proxy for token acquisition and caching
- Support ManagedClusterSetBinding for multi-tenant isolation
- Maintain backward compatibility with existing ManagedServiceAccount resources

### Non-Goals

- Implement token caching outside of cluster-proxy (cluster-proxy owns caching)
- Support offline authentication (requires cluster-proxy connectivity)
- Manage spoke cluster RBAC automatically (separate concern)
- Replace cluster-proxy component
- Support user authentication (only ServiceAccount authentication)
- Modify existing ManagedServiceAccount CRD

## Proposal

### High-Level Design

Introduce a **ServiceAccountMapping** CRD that maps hub ServiceAccounts to managed cluster service account names. When created:

1. A controller discovers bound clusters via ManagedClusterSetBinding
2. Creates ManagedServiceAccount resources for each bound cluster
3. cluster-proxy intercepts requests from mapped hub ServiceAccounts
4. cluster-proxy directly requests tokens from managed clusters
5. cluster-proxy caches tokens and forwards requests transparently

```
┌─────────────────────────────────────────────────────────────────┐
│                         Hub Cluster                             │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 1. Admin Creates ServiceAccountMapping                   │  │
│  │    namespace: argocd                                     │  │
│  │    hubServiceAccount: argocd-hub-sa                      │  │
│  │    managedServiceAccount: argocd-spoke-sa                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 2. Controller watches ServiceAccountMapping              │  │
│  │    - Finds ManagedClusterSetBinding in argocd namespace  │  │
│  │    - Discovers bound clusters (cluster1...cluster100)    │  │
│  │    - Creates ManagedServiceAccount for each cluster:     │  │
│  │      * argocd/cluster1 → argocd-spoke-sa                │  │
│  │      * argocd/cluster2 → argocd-spoke-sa                │  │
│  │      * ...                                                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 3. ArgoCD App (using argocd-hub-sa)                      │  │
│  │    Makes request via cluster-proxy to cluster1           │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                            │
│                     ▼                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 4. cluster-proxy Hub Agent                               │  │
│  │    - Detects caller: argocd-hub-sa                       │  │
│  │    - Looks up ServiceAccountMapping in argocd namespace  │  │
│  │    - Finds mapping: argocd-hub-sa → argocd-spoke-sa     │  │
│  │    - Checks token cache (miss)                           │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                            │
└─────────────────────┼────────────────────────────────────────────┘
                      │ cluster-proxy tunnel
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Managed Cluster (cluster1)                   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 5. cluster-proxy calls TokenRequest API                  │  │
│  │    POST /api/v1/namespaces/argocd/serviceaccounts/       │  │
│  │         argocd-spoke-sa/token                            │  │
│  │    Body: {expirationSeconds: 3600}                       │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                            │
│                     ▼                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 6. Returns short-lived token (1 hour)                    │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                            │
└─────────────────────┼────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Hub Cluster                             │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 7. cluster-proxy                                         │  │
│  │    - Caches token for 55 minutes                         │  │
│  │    - Forwards original request with spoke token          │  │
│  └──────────────────┬───────────────────────────────────────┘  │
│                     │                                            │
│                     ▼                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 8. ArgoCD App                                            │  │
│  │    - Receives response from cluster1                     │  │
│  │    - No token management code needed!                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Principles

1. **Transparency**: Applications don't know tokens are being managed
2. **Automatic Provisioning**: ManagedServiceAccounts created automatically for bound clusters
3. **Cluster-proxy Enhancement**: cluster-proxy handles token lifecycle (request, cache, refresh)
4. **Direct Token Requests**: cluster-proxy directly calls managed cluster TokenRequest API (no aggregated API)
5. **ManagedClusterSetBinding Integration**: Leverage existing multi-cluster grouping

---

## API Design

### ServiceAccountMapping CRD

```go
// ServiceAccountMapping maps a hub ServiceAccount to a managed cluster
// service account name. When created, ManagedServiceAccounts are automatically
// provisioned on all clusters bound to the namespace via ManagedClusterSetBinding.
//
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:scope=Namespaced
type ServiceAccountMapping struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   ServiceAccountMappingSpec   `json:"spec,omitempty"`
    Status ServiceAccountMappingStatus `json:"status,omitempty"`
}

// ServiceAccountMappingSpec defines the mapping configuration
type ServiceAccountMappingSpec struct {
    // HubServiceAccount is the name of the ServiceAccount on the hub cluster
    // that will use this mapping for authentication to managed clusters.
    // The ServiceAccount must exist in the same namespace as this mapping resource.
    // +required
    // +kubebuilder:validation:MinLength=1
    HubServiceAccount string `json:"hubServiceAccount"`

    // ManagedServiceAccount is the name of the service account to be created
    // on managed clusters. ManagedServiceAccount resources will be automatically
    // created for each cluster bound to this namespace.
    // +required
    // +kubebuilder:validation:MinLength=1
    ManagedServiceAccount string `json:"managedServiceAccount"`

    // TokenExpirationSeconds is the requested duration of validity of tokens
    // requested from managed clusters. Defaults to 3600 seconds (1 hour).
    // cluster-proxy will cache tokens and refresh them automatically.
    // +optional
    // +kubebuilder:default=3600
    // +kubebuilder:validation:Minimum=600
    // +kubebuilder:validation:Maximum=86400
    TokenExpirationSeconds *int64 `json:"tokenExpirationSeconds,omitempty"`
}

// ServiceAccountMappingStatus reflects the current state of the mapping
type ServiceAccountMappingStatus struct {
    // Conditions represent the latest available observations of the mapping's state
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`

    // ManagedServiceAccounts lists the ManagedServiceAccount resources created
    // for this mapping, organized by cluster.
    // +optional
    ManagedServiceAccounts []ManagedServiceAccountReference `json:"managedServiceAccounts,omitempty"`
}

// ManagedServiceAccountReference references a created ManagedServiceAccount
type ManagedServiceAccountReference struct {
    // Cluster is the name of the managed cluster
    // +required
    Cluster string `json:"cluster"`

    // Name is the name of the ManagedServiceAccount resource
    // +required
    Name string `json:"name"`

    // Namespace is the namespace of the ManagedServiceAccount resource (cluster namespace)
    // +required
    Namespace string `json:"namespace"`

    // Ready indicates if the ManagedServiceAccount is ready for use
    // +required
    Ready bool `json:"ready"`
}

// Condition types
const (
    // ServiceAccountMappingReady indicates all ManagedServiceAccounts are created and ready
    ServiceAccountMappingReady = "Ready"

    // ServiceAccountMappingProgressing indicates ManagedServiceAccounts are being created
    ServiceAccountMappingProgressing = "Progressing"

    // ServiceAccountMappingDegraded indicates some ManagedServiceAccounts failed to create
    ServiceAccountMappingDegraded = "Degraded"
)
```

### Example ServiceAccountMapping Resource

```yaml
apiVersion: authentication.open-cluster-management.io/v1beta1
kind: ServiceAccountMapping
metadata:
  name: argocd-mapping
  namespace: argocd
spec:
  hubServiceAccount: argocd-hub-sa
  managedServiceAccount: argocd-spoke-sa
  tokenExpirationSeconds: 3600
status:
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2026-02-27T10:00:00Z"
    reason: AllManagedServiceAccountsReady
    message: "All ManagedServiceAccounts created and ready"
  managedServiceAccounts:
  - cluster: cluster1
    name: argocd-spoke-sa
    namespace: cluster1
    ready: true
  - cluster: cluster2
    name: argocd-spoke-sa
    namespace: cluster2
    ready: true
  # ... cluster3-cluster100
```

---

## Implementation

### Component 1: ServiceAccountMapping Controller

The controller watches ServiceAccountMapping resources and manages ManagedServiceAccount lifecycle.

```go
// pkg/controllers/serviceaccountmapping/controller.go
package serviceaccountmapping

import (
    "context"
    "fmt"

    authv1beta1 "open-cluster-management.io/managed-serviceaccount/apis/authentication/v1beta1"
    clusterv1beta2 "open-cluster-management.io/api/cluster/v1beta2"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "sigs.k8s.io/controller-runtime/pkg/client"
)

type Reconciler struct {
    client.Client
}

// Reconcile handles ServiceAccountMapping resources
func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Get ServiceAccountMapping
    mapping := &authv1beta1.ServiceAccountMapping{}
    if err := r.Get(ctx, req.NamespacedName, mapping); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. Find bound clusters via ManagedClusterSetBinding
    boundClusters, err := r.discoverBoundClusters(ctx, mapping.Namespace)
    if err != nil {
        return ctrl.Result{}, err
    }

    // 3. Ensure ManagedServiceAccount exists for each bound cluster
    var managedSAList []authv1beta1.ManagedServiceAccountReference
    allReady := true

    for _, clusterName := range boundClusters {
        msa, ready, err := r.ensureManagedServiceAccount(ctx, mapping, clusterName)
        if err != nil {
            allReady = false
            continue
        }

        managedSAList = append(managedSAList, authv1beta1.ManagedServiceAccountReference{
            Cluster:   clusterName,
            Name:      msa.Name,
            Namespace: msa.Namespace,
            Ready:     ready,
        })

        if !ready {
            allReady = false
        }
    }

    // 4. Update status
    mapping.Status.ManagedServiceAccounts = managedSAList
    if allReady {
        setCondition(mapping, authv1beta1.ServiceAccountMappingReady, metav1.ConditionTrue,
            "AllManagedServiceAccountsReady", "All ManagedServiceAccounts created and ready")
    } else {
        setCondition(mapping, authv1beta1.ServiceAccountMappingProgressing, metav1.ConditionTrue,
            "CreatingManagedServiceAccounts", "Creating ManagedServiceAccounts for bound clusters")
    }

    if err := r.Status().Update(ctx, mapping); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}

// discoverBoundClusters finds all clusters bound to the namespace via ManagedClusterSetBinding
func (r *Reconciler) discoverBoundClusters(ctx context.Context, namespace string) ([]string, error) {
    // 1. List ManagedClusterSetBindings in the namespace
    bindings := &clusterv1beta2.ManagedClusterSetBindingList{}
    if err := r.List(ctx, bindings, client.InNamespace(namespace)); err != nil {
        return nil, err
    }

    if len(bindings.Items) == 0 {
        return nil, fmt.Errorf("no ManagedClusterSetBinding found in namespace %s", namespace)
    }

    // 2. For each binding, get the ManagedClusterSet and find member clusters
    var allClusters []string
    for _, binding := range bindings.Items {
        // Check if binding is Bound
        if !isBindingBound(&binding) {
            continue
        }

        clusterSetName := binding.Spec.ClusterSet

        // Get all ManagedClusters with the clusterset label
        clusters, err := r.getClustersInSet(ctx, clusterSetName)
        if err != nil {
            return nil, err
        }

        allClusters = append(allClusters, clusters...)
    }

    return allClusters, nil
}

// getClustersInSet finds all clusters in a ManagedClusterSet
func (r *Reconciler) getClustersInSet(ctx context.Context, clusterSetName string) ([]string, error) {
    clusters := &clusterv1.ManagedClusterList{}
    if err := r.List(ctx, clusters, client.MatchingLabels{
        clusterv1beta2.ClusterSetLabel: clusterSetName,
    }); err != nil {
        return nil, err
    }

    var clusterNames []string
    for _, cluster := range clusters.Items {
        clusterNames = append(clusterNames, cluster.Name)
    }

    return clusterNames, nil
}

// ensureManagedServiceAccount creates or updates ManagedServiceAccount for a cluster
func (r *Reconciler) ensureManagedServiceAccount(
    ctx context.Context,
    mapping *authv1beta1.ServiceAccountMapping,
    clusterName string,
) (*authv1beta1.ManagedServiceAccount, bool, error) {
    msa := &authv1beta1.ManagedServiceAccount{
        ObjectMeta: metav1.ObjectMeta{
            Name:      mapping.Spec.ManagedServiceAccount,
            Namespace: clusterName, // Cluster namespace
            Labels: map[string]string{
                "app.kubernetes.io/managed-by": "serviceaccount-mapping",
                "serviceaccount-mapping":       mapping.Namespace + "/" + mapping.Name,
            },
            OwnerReferences: []metav1.OwnerReference{
                {
                    APIVersion: mapping.APIVersion,
                    Kind:       mapping.Kind,
                    Name:       mapping.Name,
                    UID:        mapping.UID,
                    Controller: ptr.To(true),
                },
            },
        },
        Spec: authv1beta1.ManagedServiceAccountSpec{
            Rotation: authv1beta1.ManagedServiceAccountRotation{
                Enabled: false, // No rotation needed for ephemeral tokens
            },
        },
    }

    // Create or update
    existing := &authv1beta1.ManagedServiceAccount{}
    err := r.Get(ctx, client.ObjectKey{Namespace: clusterName, Name: msa.Name}, existing)
    if err != nil {
        if client.IgnoreNotFound(err) == nil {
            // Create
            if err := r.Create(ctx, msa); err != nil {
                return nil, false, err
            }
            return msa, false, nil
        }
        return nil, false, err
    }

    // Check if ready
    ready := isManagedServiceAccountReady(existing)
    return existing, ready, nil
}

func isManagedServiceAccountReady(msa *authv1beta1.ManagedServiceAccount) bool {
    for _, cond := range msa.Status.Conditions {
        if cond.Type == "ServiceAccountCreated" && cond.Status == metav1.ConditionTrue {
            return true
        }
    }
    return false
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

### Component 2: cluster-proxy user-server Enhancement

The cluster-proxy **user-server** component (hub-side HTTP proxy) needs to be enhanced to support transparent token acquisition.

#### Architecture Note

cluster-proxy has these components:
- **Hub**: user-server (HTTP proxy) + ANP proxy-server (gRPC tunnel server)
- **Spoke**: proxy-agent (gRPC tunnel client)

The ServiceAccountMapping logic goes in **user-server.ServeHTTP()** because:
- It already receives HTTP requests from applications
- It can see and modify HTTP headers (including Authorization)
- It can extract caller ServiceAccount identity from authenticated requests
- It already creates tunnels to managed clusters

#### user-server Token Resolution Logic

```go
// pkg/userserver/token_resolver.go
package userserver

import (
    "context"
    "fmt"
    "net/http"
    "sync"
    "time"

    authv1 "k8s.io/api/authentication/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
    "sigs.k8s.io/controller-runtime/pkg/client"

    authv1beta1 "open-cluster-management.io/managed-serviceaccount/apis/authentication/v1beta1"
)

// TokenResolver resolves hub ServiceAccount to spoke tokens via ServiceAccountMapping
type TokenResolver struct {
    hubClient     client.Client
    tokenCache    *TokenCache
    clusterClients map[string]kubernetes.Interface // Cluster name -> client
    mu            sync.RWMutex
}

// TokenCache caches spoke tokens
type TokenCache struct {
    tokens map[string]*CachedToken
    mu     sync.RWMutex
}

type CachedToken struct {
    Token      string
    Expiration time.Time
}

// ResolveToken resolves a hub ServiceAccount to a spoke token for the target cluster
func (r *TokenResolver) ResolveToken(
    ctx context.Context,
    hubNamespace string,
    hubServiceAccount string,
    targetCluster string,
) (string, error) {
    // 1. Check cache
    cacheKey := fmt.Sprintf("%s/%s/%s", hubNamespace, hubServiceAccount, targetCluster)
    if token := r.tokenCache.Get(cacheKey); token != "" {
        return token, nil
    }

    // 2. Find ServiceAccountMapping in hub namespace
    mappings := &authv1beta1.ServiceAccountMappingList{}
    if err := r.hubClient.List(ctx, mappings, client.InNamespace(hubNamespace)); err != nil {
        return "", fmt.Errorf("failed to list ServiceAccountMappings: %w", err)
    }

    var mapping *authv1beta1.ServiceAccountMapping
    for i := range mappings.Items {
        if mappings.Items[i].Spec.HubServiceAccount == hubServiceAccount {
            mapping = &mappings.Items[i]
            break
        }
    }

    if mapping == nil {
        return "", fmt.Errorf("no ServiceAccountMapping found for hub SA %s/%s",
            hubNamespace, hubServiceAccount)
    }

    // 3. Get spoke service account name from mapping
    spokeSAName := mapping.Spec.ManagedServiceAccount
    spokeNamespace := hubNamespace // Spoke SA in same namespace as hub

    // 4. Get cluster client
    clusterClient, err := r.getClusterClient(targetCluster)
    if err != nil {
        return "", fmt.Errorf("failed to get cluster client for %s: %w", targetCluster, err)
    }

    // 5. Request token from managed cluster directly
    expirationSeconds := int64(3600)
    if mapping.Spec.TokenExpirationSeconds != nil {
        expirationSeconds = *mapping.Spec.TokenExpirationSeconds
    }

    tokenRequest := &authv1.TokenRequest{
        Spec: authv1.TokenRequestSpec{
            ExpirationSeconds: &expirationSeconds,
        },
    }

    result, err := clusterClient.CoreV1().
        ServiceAccounts(spokeNamespace).
        CreateToken(ctx, spokeSAName, tokenRequest, metav1.CreateOptions{})
    if err != nil {
        return "", fmt.Errorf("failed to create token on cluster %s: %w", targetCluster, err)
    }

    // 6. Cache token (expire 5 minutes before actual expiration)
    r.tokenCache.Set(cacheKey, result.Status.Token, result.Status.ExpirationTimestamp.Time)

    return result.Status.Token, nil
}

// getClusterClient returns a client for the target managed cluster
func (r *TokenResolver) getClusterClient(clusterName string) (kubernetes.Interface, error) {
    r.mu.RLock()
    client, exists := r.clusterClients[clusterName]
    r.mu.RUnlock()

    if exists {
        return client, nil
    }

    // Create new client via cluster-proxy mechanism
    // This uses existing cluster-proxy infrastructure
    client, err := r.createClusterClient(clusterName)
    if err != nil {
        return nil, err
    }

    r.mu.Lock()
    r.clusterClients[clusterName] = client
    r.mu.Unlock()

    return client, nil
}

// Enhanced ServeHTTP for user-server with ServiceAccountMapping support
func (k *userServer) ServeHTTP(wr http.ResponseWriter, req *http.Request) {
    if klog.V(4).Enabled() {
        dump, err := httputil.DumpRequest(req, true)
        if err != nil {
            http.Error(wr, err.Error(), http.StatusBadRequest)
            return
        }
        klog.V(4).Infof("request:\n%s", string(dump))
    }

    var tsc utils.TargetServiceConfig
    var err error

    switch utils.GetProxyType(req.RequestURI) {
    case utils.ProxyTypeService:
        tsc, err = utils.GetTargetServiceConfig(req.RequestURI)
    case utils.ProxyTypeKubeAPIServer:
        tsc, err = utils.GetTargetServiceConfigForKubeAPIServer(req.RequestURI)
    }
    if err != nil {
        http.Error(wr, err.Error(), http.StatusBadRequest)
        return
    }

    // NEW: Extract caller ServiceAccount from authentication
    callerNamespace, callerSA, err := extractServiceAccountFromAuth(req)
    if err == nil && callerSA != "" {
        // Try to resolve token via ServiceAccountMapping
        if token, err := k.tokenResolver.ResolveToken(req.Context(), callerNamespace, callerSA, tsc.Cluster); err == nil {
            // Inject spoke token
            req.Header.Set("Authorization", "Bearer "+token)
            klog.V(4).Infof("Injected spoke token for hub SA %s/%s -> cluster %s", callerNamespace, callerSA, tsc.Cluster)
        } else {
            klog.V(4).Infof("No ServiceAccountMapping found for %s/%s, using original auth", callerNamespace, callerSA)
        }
    }

    // Continue with existing proxy logic
    targetURL, err := url.Parse(serviceProxyURL(tsc.Cluster))
    if err != nil {
        http.Error(wr, err.Error(), http.StatusBadRequest)
        return
    }

    tunnel, err := k.getTunnel(req.Context())
    if err != nil {
        http.Error(wr, err.Error(), http.StatusBadRequest)
        return
    }

    proxy := httputil.NewSingleHostReverseProxy(targetURL)
    proxy.Transport = &http.Transport{
        DialContext:       tunnel.DialContext,
        // ... existing config
    }
    proxy.ServeHTTP(wr, req)
}

// extractServiceAccountFromAuth extracts ServiceAccount namespace and name from request
func extractServiceAccountFromAuth(req *http.Request) (namespace, name string, err error) {
    // Check for client certificate (common in-cluster auth)
    if req.TLS != nil && len(req.TLS.PeerCertificates) > 0 {
        cert := req.TLS.PeerCertificates[0]
        // Parse from cert subject: system:serviceaccount:<namespace>:<name>
        for _, name := range cert.Subject.Names {
            if nameStr, ok := name.Value.(string); ok {
                if strings.HasPrefix(nameStr, "system:serviceaccount:") {
                    parts := strings.Split(nameStr, ":")
                    if len(parts) == 4 {
                        return parts[2], parts[3], nil
                    }
                }
            }
        }
    }

    // Check Authorization header for existing token (less common)
    authHeader := req.Header.Get("Authorization")
    if strings.HasPrefix(authHeader, "Bearer ") {
        // Could validate token to extract SA info, but complex
        // For now, rely on certificate authentication
    }

    return "", "", fmt.Errorf("no ServiceAccount found in request")
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

---

## User Experience

### Setup (One-time, by Admin)

```bash
# 1. Create ManagedClusterSet (if not exists)
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
  namespace: argocd
spec:
  clusterSet: production
EOF

# 4. Create ArgoCD ServiceAccount (if not exists)
kubectl create serviceaccount argocd-hub-sa -n argocd

# 5. Create ServiceAccountMapping
kubectl apply -f - <<EOF
apiVersion: authentication.open-cluster-management.io/v1beta1
kind: ServiceAccountMapping
metadata:
  name: argocd-mapping
  namespace: argocd
spec:
  hubServiceAccount: argocd-hub-sa
  managedServiceAccount: argocd-spoke-sa
  tokenExpirationSeconds: 3600
EOF

# 6. Verify ManagedServiceAccounts are created
kubectl get managedserviceaccounts -A | grep argocd-spoke-sa
```

### Application Usage (ArgoCD Example)

**No code changes required!** ArgoCD just uses its existing ServiceAccount:

```yaml
# ArgoCD Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-server
  namespace: argocd
spec:
  template:
    spec:
      serviceAccountName: argocd-hub-sa  # Just use the hub SA
      containers:
      - name: argocd-server
        image: argoproj/argocd:latest
        # ArgoCD code unchanged - accesses clusters via cluster-proxy
        # cluster-proxy transparently handles token acquisition
```

**Application code remains unchanged:**

```go
// ArgoCD accessing cluster1 (pseudo-code)
kubeconfig := &rest.Config{
    Host: "https://hub-api/apis/cluster.open-cluster-management.io/v1beta1/managedclusters/cluster1/proxy",
    // No token needed! Uses in-cluster ServiceAccount authentication
    // cluster-proxy intercepts and handles token mapping
}

client, _ := kubernetes.NewForConfig(kubeconfig)
pods, _ := client.CoreV1().Pods("default").List(ctx, metav1.ListOptions{})
// Works transparently!
```

### Checking Status

```bash
# Check ServiceAccountMapping status
kubectl get serviceaccountmapping -n argocd argocd-mapping -o yaml

# Output shows all created ManagedServiceAccounts
status:
  conditions:
  - type: Ready
    status: "True"
    message: "All ManagedServiceAccounts created and ready"
  managedServiceAccounts:
  - cluster: cluster1
    name: argocd-spoke-sa
    namespace: cluster1
    ready: true
  - cluster: cluster2
    name: argocd-spoke-sa
    namespace: cluster2
    ready: true
  # ... cluster3-cluster100
```

---

## Comparison with TokenRequest API

| Aspect | TokenRequest API | ServiceAccountMapping |
|--------|------------------|----------------------|
| **App Code Changes** | Required (call TokenRequest API) | Zero changes |
| **Token Management** | App manages (request, cache, refresh) | cluster-proxy manages |
| **RBAC Complexity** | Need TokenRequest CREATE permission | Just use hub ServiceAccount |
| **Transparency** | Explicit token requests | Completely transparent |
| **Performance** | App controls caching | cluster-proxy caches |
| **Token Lifetime Control** | App specifies per request | Configured in mapping |
| **ManagedServiceAccount Creation** | Manual | Automatic |
| **Best For** | New apps, explicit control, fine-grained token management | Existing apps, zero changes, simplified operations |
| **cluster-proxy Dependency** | Low (just for cluster access) | High (manages token lifecycle) |
| **Architecture Complexity** | Aggregated API server needed | Controller + cluster-proxy enhancement |

### When to Use Each

**Use TokenRequest API when:**
- Building new applications from scratch
- Need fine-grained control over token lifecycle
- Want explicit token management in application code
- Need tokens outside of cluster-proxy context
- Security policy requires explicit token requests

**Use ServiceAccountMapping when:**
- Migrating existing applications (ArgoCD, operators, etc.)
- Want zero code changes
- Prefer transparent authentication
- Trust cluster-proxy for token management
- Want simplified operations and RBAC

---

## Test Plan

### Unit Tests

1. **ServiceAccountMapping Controller Tests**
   - Test discovery of bound clusters via ManagedClusterSetBinding
   - Test ManagedServiceAccount creation for multiple clusters
   - Test status update with correct cluster references
   - Test cleanup when ServiceAccountMapping deleted
   - Test handling of binding changes (clusters added/removed)

2. **cluster-proxy Token Resolver Tests**
   - Test token resolution for valid ServiceAccountMapping
   - Test cache hit/miss scenarios
   - Test token refresh before expiration
   - Test error handling when mapping not found
   - Test error handling when cluster unreachable

### Integration Tests

1. **E2E ServiceAccountMapping Flow**
   - Create ManagedClusterSet and binding
   - Create ServiceAccountMapping
   - Verify ManagedServiceAccounts created on all bound clusters
   - Verify ServiceAccounts created on managed clusters
   - Make request via cluster-proxy using hub SA
   - Verify request succeeds with spoke token

2. **Multi-cluster Tests**
   - Create mapping for 100 clusters
   - Verify all ManagedServiceAccounts created
   - Test concurrent requests to different clusters
   - Verify cluster-proxy token caching works correctly

3. **RBAC Tests**
   - Verify hub SA without mapping cannot access clusters
   - Verify different namespaces have isolated mappings
   - Verify ManagedClusterSetBinding controls cluster access

4. **Lifecycle Tests**
   - Test adding clusters to ManagedClusterSet (new ManagedServiceAccounts created)
   - Test removing clusters from set (ManagedServiceAccounts remain but unused)
   - Test deleting ServiceAccountMapping (ManagedServiceAccounts cleaned up)
   - Test updating mapping spec (ManagedServiceAccounts updated)

### Performance Tests

1. **Token Resolution Latency**
   - Measure first request latency (cache miss)
   - Measure subsequent request latency (cache hit)
   - Test concurrent token requests from multiple apps

2. **Scale Tests**
   - Test 100 clusters with single mapping
   - Test 10 mappings in same namespace
   - Test token cache memory usage with many clusters
   - Test cluster-proxy resource usage under load

---

## Risks and Mitigations

### Risk: cluster-proxy Complexity

**Risk**: Significant cluster-proxy enhancement increases complexity and maintenance burden.

**Mitigation**:
- Design cluster-proxy changes as modular middleware
- Comprehensive testing of cluster-proxy token resolution
- Document cluster-proxy configuration and troubleshooting
- Consider feature flag to disable transparent auth if needed

### Risk: Token Cache Memory Usage

**Risk**: cluster-proxy caching many tokens could consume significant memory.

**Mitigation**:
- Implement LRU cache with configurable size limits
- Monitor cluster-proxy memory usage
- Document expected memory usage per cluster/mapping
- Tokens expire and are evicted automatically

### Risk: Token Request Failures

**Risk**: If managed cluster token request fails, application request fails.

**Mitigation**:
- Implement retry logic in cluster-proxy
- Return clear error messages to applications
- Monitor token request failure rates with metrics
- Consider fallback to cached expired token with warning (short grace period)

### Risk: Breaking Existing cluster-proxy Users

**Risk**: cluster-proxy enhancement might affect existing users.

**Mitigation**:
- Design as optional middleware (only active when ServiceAccountMapping exists)
- Existing authentication methods continue to work
- Feature flag to disable transparent auth
- Comprehensive regression testing

### Risk: Debugging Difficulty

**Risk**: Transparent authentication makes debugging harder (where did token come from?).

**Mitigation**:
- Add detailed logging in cluster-proxy token resolution
- Include mapping info in cluster-proxy metrics
- Document troubleshooting steps
- Provide CLI tool to test token resolution

---

## Security Considerations

### Token Lifetime

- Default 1-hour expiration balances security and performance
- Minimum 10 minutes to prevent excessive request load
- Maximum 24 hours to limit exposure window
- Configured per mapping, not per request

### Token Scope

- Tokens inherit spoke ServiceAccount permissions
- No privilege escalation beyond spoke RBAC
- Hub ServiceAccount has no special privileges on hub
- Spoke RBAC managed separately (out of scope for this design)

### Token Storage

- Tokens cached in cluster-proxy memory only
- Never persisted to disk or etcd
- Cache eviction on expiration or memory pressure
- No token storage on hub cluster

### Authentication Flow Security

- Hub authentication still required (ServiceAccount must exist)
- cluster-proxy validates ServiceAccountMapping exists
- ManagedClusterSetBinding enforces cluster access control
- Audit logs capture both hub user and spoke token usage

### Multi-tenancy

- ServiceAccountMapping is namespace-scoped
- Different namespaces have isolated mappings
- ManagedClusterSetBinding provides namespace-level cluster access control
- No cross-namespace token access

---

## Migration Path

### Phase 1: Introduction (v0.11.0)

1. Introduce ServiceAccountMapping CRD
2. Implement ServiceAccountMapping controller
3. Enhance cluster-proxy with token resolution (feature flag disabled by default)
4. Keep existing ManagedServiceAccount Secret synchronization
5. Document new approach with examples

### Phase 2: Adoption (v0.12.0)

1. Enable cluster-proxy token resolution by default
2. Provide migration guide for existing applications
3. Add metrics and monitoring
4. Support both Secret-based and ServiceAccountMapping patterns

### Phase 3: Deprecation (v0.13.0)

1. Recommend ServiceAccountMapping pattern in documentation
2. Mark Secret synchronization as deprecated
3. Add deprecation warnings
4. Migrate existing OCM components (ArgoCD plugin, etc.)

### Phase 4: Removal (v1.0.0)

1. Remove Secret synchronization from addon agent
2. Only support ServiceAccountMapping pattern
3. Clean up deprecated code

---

## Alternatives Considered

### Alternative 1: ManagedServiceAccountTokenRequest API

**Approach**: Aggregated API server for explicit token requests (documented in [token-request-api.md](token-request-api.md)).

**Pros**:
- Explicit control over token lifecycle
- No cluster-proxy changes needed
- Simpler architecture (just API server)

**Cons**:
- Requires app code changes
- Apps must implement token management
- More complex RBAC (need TokenRequest permission)
- Not transparent to existing apps

**Decision**: ServiceAccountMapping is better for existing apps; TokenRequest API better for new apps. Could support both.

### Alternative 2: cluster-proxy with Built-in Token Store

**Approach**: cluster-proxy stores long-lived tokens and uses them for authentication.

**Pros**:
- Simple implementation
- No token requests to managed clusters

**Cons**:
- Back to storing long-lived tokens (defeats purpose)
- cluster-proxy becomes credential store (security risk)
- No improvement over Secret-based approach

**Decision**: Rejected - doesn't solve security problem.

### Alternative 3: Webhook-based Token Injection

**Approach**: Mutating webhook intercepts pod creation and injects tokens as environment variables.

**Pros**:
- Works with any application
- No cluster-proxy changes

**Cons**:
- Tokens in environment variables (security risk)
- Still need to manage token lifecycle
- Only works for pods, not controllers
- Doesn't solve Secret read permission problem

**Decision**: Rejected - inferior to ServiceAccountMapping.

### Alternative 4: Keep Current Secret-based Approach

**Approach**: Continue storing long-lived tokens as Secrets.

**Pros**:
- No implementation work
- Proven to work

**Cons**:
- Doesn't solve either problem (token storage or RBAC complexity)
- Security risks remain
- Not acceptable for security-conscious environments

**Decision**: Rejected - must improve security.

---

## Conclusion

The **ServiceAccountMapping** approach provides transparent multi-cluster authentication that solves both the token storage problem and the RBAC complexity problem:

✅ **No Token Storage**: Tokens generated on-demand, cached in cluster-proxy memory only
✅ **Simple RBAC**: Applications use their own ServiceAccount, no additional permissions
✅ **Zero Code Changes**: Existing applications work without modification
✅ **Automatic Provisioning**: ManagedServiceAccounts created automatically for bound clusters
✅ **Multi-tenant Support**: Namespace-scoped mappings with ManagedClusterSetBinding isolation
✅ **Scalable**: One mapping per app, not per cluster

This enhancement provides the best user experience for migrating existing hub components (ArgoCD, operators, monitoring) to secure, ephemeral authentication while maintaining backward compatibility with existing ManagedServiceAccount resources.

### Recommendation

**Implement ServiceAccountMapping** as the primary approach for transparent multi-cluster authentication. The [ManagedServiceAccountTokenRequest API](token-request-api.md) can be offered as an alternative for applications that need explicit token management control.
