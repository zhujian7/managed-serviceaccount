# cluster-proxy Architecture Review

## Overview

After reviewing the cluster-proxy codebase, I've identified important architectural considerations for the ServiceAccountMapping design.

## cluster-proxy Architecture

### What cluster-proxy Actually Does

cluster-proxy is built on [apiserver-network-proxy (konnectivity)](https://github.com/kubernetes-sigs/apiserver-network-proxy) and provides:

1. **Network Tunneling**: Establishes reverse gRPC tunnels from managed clusters to hub
2. **TCP-level Proxying**: Forwards TCP connections through the tunnel
3. **Not HTTP-aware**: Operates at the connection/transport layer, not HTTP request layer

### How Clients Use cluster-proxy

From `/Users/jiazhu/go/src/open-cluster-management.io/cluster-proxy/examples/test-client.md`:

```go
// 1. Create a gRPC tunnel using konnectivity
tunnel, err := konnectivity.CreateSingleUseGrpcTunnel(
    context.TODO(),
    proxyServerAddress,
    grpc.WithTransportCredentials(...),
)

// 2. Build Kubernetes client config
cfg, err := clientcmd.BuildConfigFromFlags("", managedClusterKubeconfig)

// 3. Set cluster name as host
cfg.Host = managedClusterName

// 4. Override the TCP dialer to use the tunnel
cfg.Dial = tunnel.DialContext

// 5. Create client - authentication comes from cfg (kubeconfig)
client := kubernetes.NewForConfigOrDie(cfg)
```

**Key Observations:**
- cluster-proxy doesn't see HTTP headers (including Authorization)
- cluster-proxy doesn't modify authentication
- Authentication is handled by the Kubernetes client using the kubeconfig
- cluster-proxy just tunnels TCP connections

## Implications for ServiceAccountMapping Design

### Problem: cluster-proxy Cannot Intercept HTTP Headers

Our ServiceAccountMapping design assumes cluster-proxy can:
1. Intercept HTTP requests
2. Extract hub ServiceAccount identity
3. Look up ServiceAccountMapping
4. Request token from managed cluster
5. Inject token into Authorization header

**Reality**: cluster-proxy operates at TCP level and cannot do any of this.

### Where Token Injection Could Actually Happen

#### Option 1: OCM Registration's Proxy Subresource Handler

The `/apis/cluster.open-cluster-management.io/v1beta1/managedclusters/{cluster}/proxy` path is likely served by an **aggregated API server in OCM registration**, not by cluster-proxy.

Flow:
```
Client → Hub API Server → Aggregated API (OCM Registration Proxy Handler)
                                ↓ (uses cluster-proxy for connectivity)
                         Managed Cluster API Server
```

The **Proxy Handler** could:
1. Extract client identity from request (ServiceAccount name/namespace)
2. Look up ServiceAccountMapping in that namespace
3. Use cluster-proxy to connect to managed cluster
4. Call managed cluster's `/api/v1/namespaces/{ns}/serviceaccounts/{sa}/token` API
5. Forward original request with spoke token in Authorization header

**This is the right place for ServiceAccountMapping logic.**

#### Option 2: Client-side Library/Wrapper

Provide a Kubernetes client library that:
1. Wraps `kubernetes.Interface`
2. Intercepts requests before they go through cluster-proxy
3. Looks up ServiceAccountMapping
4. Requests token from managed cluster
5. Injects token into request

**Cons**: Requires app code changes (defeats "zero changes" goal)

#### Option 3: HTTP Proxy Mode for cluster-proxy

Enhance cluster-proxy to add an HTTP proxy mode (in addition to current gRPC mode):
1. Accept HTTP/HTTPS requests (not just TCP tunneling)
2. Parse HTTP headers
3. Implement ServiceAccountMapping logic
4. Forward with modified Authorization header

**Cons**: Significant cluster-proxy architecture change

### Recommended Approach

**Implement ServiceAccountMapping logic in OCM Registration's proxy subresource handler**, not in cluster-proxy.

## Revised Architecture

### Component Responsibilities

```
┌────────────────────────────────────────────────────────────────┐
│                         Hub Cluster                            │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ App (using argocd-hub-sa)                                │ │
│  │ Makes request via:                                       │ │
│  │ /apis/cluster...io/v1beta1/managedclusters/cluster1/proxy│ │
│  └────────────────────────┬─────────────────────────────────┘ │
│                           │                                    │
│                           ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Hub API Server                                           │ │
│  │ - Authenticates request (argocd-hub-sa)                  │ │
│  │ - Routes to aggregated API                               │ │
│  └────────────────────────┬─────────────────────────────────┘ │
│                           │                                    │
│                           ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ OCM Registration: Proxy Subresource Handler              │ │
│  │                                                           │ │
│  │ NEW LOGIC:                                                │ │
│  │ 1. Extract caller SA: argocd-hub-sa in ns argocd        │ │
│  │ 2. Look up ServiceAccountMapping in argocd              │ │
│  │    Found: argocd-hub-sa → argocd-spoke-sa              │ │
│  │ 3. Check token cache (cache miss)                        │ │
│  │ 4. Use cluster-proxy to connect to cluster1             │ │
│  │ 5. Call cluster1's TokenRequest API:                     │ │
│  │    POST /api/v1/namespaces/argocd/serviceaccounts/      │ │
│  │         argocd-spoke-sa/token                           │ │
│  │ 6. Cache token (55 min)                                  │ │
│  │ 7. Forward original request with spoke token             │ │
│  └────────────────────────┬─────────────────────────────────┘ │
│                           │ (via cluster-proxy)                │
└───────────────────────────┼────────────────────────────────────┘
                            │ (konnectivity gRPC tunnel)
                            ▼
┌────────────────────────────────────────────────────────────────┐
│                    Managed Cluster (cluster1)                  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ Request arrives with:                                    │ │
│  │ Authorization: Bearer <argocd-spoke-sa token>            │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

### Implementation Location

**NOT in cluster-proxy**. Instead, implement in:
- OCM registration component's proxy subresource handler, OR
- A new HTTP proxy component that sits in front of cluster-proxy

cluster-proxy continues to do what it does today: provide TCP tunnels.

## Questions for User

1. **Does OCM registration have a proxy subresource handler that serves `/managedclusters/{name}/proxy`?**
   - If yes, that's where ServiceAccountMapping logic should go
   - If no, where is this path handled?

2. **Is there an HTTP-layer component that uses cluster-proxy for connectivity?**
   - That's the right place for token injection logic

3. **Are we OK with modifying OCM registration instead of cluster-proxy?**
   - ServiceAccountMapping would be implemented there

4. **Alternative: Should we provide a client library instead?**
   - Apps would use a wrapper library that handles token acquisition
   - Requires minor code changes but simpler architecture

## Comparison: Library vs Proxy Handler

| Aspect | Client Library | Proxy Handler Enhancement |
|--------|---------------|---------------------------|
| **Code Changes** | Minor (use wrapper lib) | Zero |
| **Where Logic Lives** | Client-side | Hub server-side |
| **Token Cache** | Per-client process | Centralized |
| **Implementation Complexity** | Low | Medium |
| **Requires OCM Changes** | No | Yes |
| **Performance** | More token requests | Fewer (shared cache) |

## Recommendation

1. **Find where `/managedclusters/{name}/proxy` is handled** (likely OCM registration)
2. **Implement ServiceAccountMapping logic there**, not in cluster-proxy
3. **Keep cluster-proxy unchanged** - it just provides network tunnels
4. **Update our design document** to clarify this architecture

The ServiceAccountMapping concept is still valid, but the implementation location needs to be corrected.
