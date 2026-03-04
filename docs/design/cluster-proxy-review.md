# cluster-proxy Architecture Review

## Overview

This document summarizes the cluster-proxy architecture analysis to determine the correct implementation location for ServiceAccountMapping.

## cluster-proxy Architecture

### Components

cluster-proxy is built on [apiserver-network-proxy (konnectivity)](https://github.com/kubernetes-sigs/apiserver-network-proxy) and has the following components:

**Hub Cluster:**
1. **user-server**: HTTP reverse proxy server that receives requests from applications
2. **ANP proxy-server**: apiserver-network-proxy gRPC tunnel server

**Managed Cluster:**
3. **proxy-agent**: apiserver-network-proxy agent that establishes gRPC tunnel to proxy-server

### Request Flow

```
Application (ArgoCD)
    ↓ HTTP request with hub SA auth
user-server (HTTP proxy on hub)
    ↓ creates gRPC tunnel
ANP proxy-server (hub)
    ↓ gRPC tunnel (hub SA auth passed through)
proxy-agent (spoke)
    ↓ [ServiceAccountMapping: hub SA → spoke token injection]
    ↓ forwards with spoke token
Managed Cluster API Server
```

### Key Discovery: cluster-proxy is HTTP-Aware

Initial confusion: The konnectivity example shows TCP-level tunneling:
```go
cfg.Dial = tunnel.DialContext  // Override TCP dialer
```

This suggested cluster-proxy only handles TCP tunneling.

**However**, cluster-proxy has HTTP-aware components that can intercept and modify requests:

**Hub-side:**

- **user-server**: HTTP reverse proxy (uses `httputil.ReverseProxy`)
- Receives HTTP requests from applications
- Can see and modify HTTP headers (including Authorization)
- Already creates tunnels to managed clusters

**Spoke-side:**

- **proxy-agent**: Receives requests from gRPC tunnel
- Can intercept HTTP requests before forwarding to local API server
- Can modify headers (including Authorization)
- **✅ Chosen location for ServiceAccountMapping implementation** (better performance/scalability)

### Architecture Deep Dive

#### user-server (Hub-side HTTP Proxy)

From `/pkg/userserver/user_server.go`:

```go
func (k *userServer) ServeHTTP(wr http.ResponseWriter, req *http.Request) {
    // user-server receives HTTP requests
    dump, _ := httputil.DumpRequest(req, true)

    // Extracts target cluster from URL
    tsc, _ := utils.GetTargetServiceConfig(req.RequestURI)

    // Creates tunnel to managed cluster
    tunnel, _ := k.getTunnel(req.Context())

    // Forwards HTTP request through tunnel
    proxy := httputil.NewSingleHostReverseProxy(targetURL)
    proxy.Transport = &http.Transport{
        DialContext: tunnel.DialContext,
        // ... TLS config
    }
    proxy.ServeHTTP(wr, req)
}
```

## Implementation Decision

### ✅ Implement in cluster-proxy proxy-agent (spoke-side)

The proxy-agent component (on managed clusters) is chosen for ServiceAccountMapping logic because:

1. **Hub access**: proxy-agent already has hub kubeconfig for OCM operations
2. **Can watch hub**: Can watch ServiceAccountMapping CRs on hub
3. **Local token generation**: Generates tokens locally on spoke (lower latency)
4. **Distributed caching**: Each spoke caches its own tokens (better scalability)
5. **Less hub load**: No token requests from hub to spoke
6. **Simpler hub**: Hub user-server remains unchanged

### Enhancement Approach

Add ServiceAccountMapping logic to **proxy-agent** (spoke-side):

```go
// In proxy-agent on managed cluster
type ProxyAgent struct {
    saMappingWatcher *SAMappingWatcher
    hubClient        client.Client  // Already exists for OCM operations
    spokeClient      kubernetes.Interface
    clusterName      string
}

func (a *ProxyAgent) Start(ctx context.Context) error {
    // Initialize SAMappingWatcher with hub client
    a.saMappingWatcher = NewSAMappingWatcher(
        a.hubClient,      // Uses existing hub kubeconfig
        a.spokeClient,    // Local kube client
        a.clusterName,
    )

    // Start watching ServiceAccountMapping on hub
    if err := a.saMappingWatcher.Start(ctx); err != nil {
        return err
    }

    // Continue with existing proxy-agent logic
    // ...
}

// When processing requests from tunnel
func (a *ProxyAgent) HandleRequest(req *http.Request) error {
    // Extract hub SA from request authentication
    hubNamespace, hubSA, err := extractHubSAFromRequest(req)
    if err == nil && hubSA != "" {
        // Try to resolve spoke token via local cache + local TokenRequest
        if token, err := a.saMappingWatcher.ResolveToken(req.Context(), hubNamespace, hubSA); err == nil {
            // Inject spoke token
            req.Header.Set("Authorization", "Bearer "+token)
        }
    }

    // Forward to local API server
    // ...
}
```

## Why Not Other Options?

### ❌ Hub-side (user-server) Implementation

Initially considered implementing in user-server on hub.

**Problems**:
- Hub must call spoke TokenRequest API (extra network hop)
- Centralized token caching (hub becomes bottleneck)
- More hub load as it handles all token requests
- Requires user-server changes

### ❌ OCM Registration Proxy Subresource Handler

Could implement in OCM registration (if it has a proxy handler).

**Problem**: Adds another layer of indirection when cluster-proxy already exists.

### ❌ Client Library Wrapper

Provide a library that wraps Kubernetes client and handles token acquisition.

**Problems**:
- Requires app code changes (defeats "zero changes" goal)
- Per-process token caching (less efficient)
- Every app needs to integrate the library

## Comparison Table

| Aspect | proxy-agent Enhancement (Spoke-side) | user-server Enhancement (Hub-side) | OCM Registration Handler | Client Library |
|--------|-------------------------------------|-----------------------------------|--------------------------|----------------|
| **Code Changes for Apps** | Zero | Zero | Zero | Minor (use wrapper) |
| **Token Generation** | Local (spoke) | Remote (hub to spoke API) | Depends | Local |
| **Token Caching** | Distributed | Centralized | Centralized | Per-process |
| **Network Latency** | Low (local) | High (extra hop) | Medium | Low |
| **Hub Load** | Low | High | Medium | None |
| **Scalability** | High | Low | Medium | High |
| **Implementation Complexity** | Low | Low | Medium | Low |
| **cluster-proxy Changes** | Minimal (proxy-agent) | Minimal (user-server) | None | None |

## Conclusion

**Implement ServiceAccountMapping in cluster-proxy proxy-agent (spoke-side).**

The proxy-agent component is the optimal location because it:

- Already has hub kubeconfig for OCM operations (can watch hub resources)
- Can generate tokens locally on spoke (lower latency)
- Enables distributed token caching (better scalability)
- Reduces hub load (no token requests from hub to spoke)
- Keeps hub user-server unchanged (simpler architecture)

## Files to Modify

In cluster-proxy repository (`open-cluster-management.io/cluster-proxy`):

1. **`pkg/proxyagent/agent/sa_mapping_watcher.go`** (new file)
   - Watch ServiceAccountMapping CRs on hub cluster
   - Build and maintain local mapping cache
   - Token resolution via local TokenRequest API
   - Token caching logic

2. **`pkg/proxyagent/agent/agent.go`** (enhance existing)
   - Initialize SAMappingWatcher with hub client
   - Extract hub ServiceAccount from incoming requests
   - Inject spoke token if mapping exists
   - Forward request to local API server

3. **`pkg/proxyagent/agent/auth_extractor.go`** (new file)
   - Extract ServiceAccount identity from request authentication
   - Handle client certificate and token authentication from hub

## Next Steps

1. Implement ServiceAccountMapping CRD and controller in managed-serviceaccount repo
2. Enhance cluster-proxy proxy-agent with ServiceAccountMapping watcher
3. Add token caching to proxy-agent
4. Test with ArgoCD and other hub applications
