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
    ↓ HTTP request
user-server (HTTP proxy on hub)
    ↓ creates gRPC tunnel
ANP proxy-server (hub)
    ↓ gRPC tunnel
proxy-agent (spoke)
    ↓ forwards to
Managed Cluster API Server
```

### Key Discovery: user-server is an HTTP Proxy

Initial confusion: The konnectivity example shows TCP-level tunneling:
```go
cfg.Dial = tunnel.DialContext  // Override TCP dialer
```

This suggested cluster-proxy only handles TCP tunneling.

**However**, cluster-proxy also has a **user-server component** that:
- ✅ Is an HTTP reverse proxy (uses `httputil.ReverseProxy`)
- ✅ Receives HTTP requests from applications
- ✅ Can see and modify HTTP headers (including Authorization)
- ✅ Already creates tunnels to managed clusters
- ✅ Perfect for ServiceAccountMapping implementation

### user-server Code Evidence

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

### ✅ Implement in cluster-proxy user-server

The user-server component is the perfect location for ServiceAccountMapping logic because:

1. **HTTP-aware**: It's an HTTP proxy that can see and modify headers
2. **Extracts cluster**: Already parses target cluster from URL
3. **Receives auth**: Can extract caller ServiceAccount from request authentication
4. **Has connectivity**: Already creates tunnels to managed clusters
5. **Strategic location**: All HTTP requests to managed clusters flow through it

### Enhancement Approach

Add ServiceAccountMapping logic to `user-server.ServeHTTP()`:

```go
func (k *userServer) ServeHTTP(wr http.ResponseWriter, req *http.Request) {
    // Extract target cluster (existing code)
    tsc, _ := utils.GetTargetServiceConfig(req.RequestURI)

    // NEW: Extract caller ServiceAccount from authentication
    callerNamespace, callerSA, _ := extractServiceAccountFromAuth(req)

    // NEW: Look up ServiceAccountMapping and get token
    if token, err := k.tokenResolver.ResolveToken(req.Context(),
        callerNamespace, callerSA, tsc.Cluster); err == nil {
        // NEW: Inject spoke token
        req.Header.Set("Authorization", "Bearer "+token)
    }

    // Continue with existing proxy logic
    tunnel, _ := k.getTunnel(req.Context())
    proxy.ServeHTTP(wr, req)
}
```

## Why Not Other Options?

### ❌ OCM Registration Proxy Subresource Handler

Initially considered implementing in OCM registration (if it has a proxy handler).

**Problem**: Adds another layer of indirection. user-server already exists and serves this exact purpose.

### ❌ Spoke-side Implementation

Could enhance proxy-agent on spoke clusters to handle token mapping.

**Problems**:
- Need to sync ServiceAccountMapping info to all spokes
- Distributed token caching (less efficient)
- More complex implementation
- Higher latency (extra hops)

### ❌ Client Library Wrapper

Provide a library that wraps Kubernetes client and handles token acquisition.

**Problems**:
- Requires app code changes (defeats "zero changes" goal)
- Per-process token caching (less efficient)
- Every app needs to integrate the library

## Comparison Table

| Aspect | user-server Enhancement | OCM Registration Handler | Client Library |
|--------|------------------------|--------------------------|----------------|
| **Code Changes for Apps** | Zero | Zero | Minor (use wrapper) |
| **HTTP-aware** | Yes (already) | Maybe (depends) | Yes |
| **Implementation Complexity** | Low (add to existing) | Medium (new handler?) | Low |
| **Token Caching** | Centralized | Centralized | Per-process |
| **cluster-proxy Changes** | Minimal | None | None |
| **Architecture Impact** | Minimal | Depends | None |

## Conclusion

**Implement ServiceAccountMapping in cluster-proxy user-server.**

The user-server component is:
- Already an HTTP proxy
- Already receives all application requests to managed clusters
- Perfect strategic location for transparent token injection

No need to create new components or modify OCM registration. Just enhance the existing user-server.ServeHTTP() method.

## Files to Modify

In cluster-proxy repository (`open-cluster-management.io/cluster-proxy`):

1. **`pkg/userserver/token_resolver.go`** (new file)
   - ServiceAccountMapping lookup
   - Token resolution via managed cluster TokenRequest API
   - Token caching logic

2. **`pkg/userserver/user_server.go`** (enhance existing)
   - Add ServiceAccountMapping logic to `ServeHTTP()`
   - Extract caller ServiceAccount from request
   - Inject spoke token if mapping exists

3. **`pkg/userserver/auth_extractor.go`** (new file)
   - Extract ServiceAccount identity from request authentication
   - Handle client certificate and token authentication

## Next Steps

1. Implement ServiceAccountMapping controller in managed-serviceaccount repo
2. Enhance cluster-proxy user-server with token resolution logic
3. Add token caching to user-server
4. Test with ArgoCD and other hub applications
