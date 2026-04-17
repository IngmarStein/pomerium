# TLS Client Auth SNI Filtering Design

## Context

When `downstream_mtls` is configured, Pomerium currently requests client certificates for all TLS connections. This happens via Envoy's TLS handshake (`TLSv1.3 (IN), TLS handshake, Request CERT (13)`) even when the certificate is optional for a route.

Some clients (e.g., UniFi iOS app) are confused by the client certificate request when it's not expected. This feature adds a configuration surface to restrict client certificate requests to connections where the server name (SNI) matches specified glob patterns.

## Configuration

### Proto Change

In `pkg/grpc/config/config.proto`, add field to `DownstreamMtlsSettings`:

```protobuf
message DownstreamMtlsSettings {
  // ... existing fields 1-6 ...
  
  // Glob patterns for SNI-based client certificate request filtering.
  // When specified, client certificates are only requested for connections
  // where the server name (SNI) matches one of these patterns.
  // When empty, client certificates are requested for all connections
  // (existing behavior).
  repeated string tls_client_auth_snis = 7;
}
```

### Go Struct Change

In `config/mtls.go`, add field to `DownstreamMTLSSettings`:

```go
type DownstreamMTLSSettings struct {
    // ... existing fields ...
    
    // TLSClientAuthSNIs contains glob patterns for SNI matching.
    // When non-empty, client certificates are only requested for connections
    // where the SNI matches at least one pattern.
    // When empty, certificates are requested for all connections.
    TLSClientAuthSNIs []string
}
```

### YAML Configuration

Users can configure via YAML:

```yaml
downstream_mtls:
  ca: /path/to/ca.pem
  enforcement: policy
  tls_client_auth_snis:
    - "*.example.com"
    - "auth.example.com"
```

## Behavior

| `tls_client_auth_snis` | Behavior |
|------------------------|----------|
| Empty / unset | Request client certificates for all connections (current behavior) |
| `["*.example.com"]` | Request client certificates only when SNI matches `*.example.com` |
| `["auth.example.com"]` | Request client certificates only when SNI is `auth.example.com` |

### SNI Matching

SNI matching uses glob patterns with the same logic as existing SANMatcher DNS patterns:
- `*.example.com` matches `auth.example.com`, `app.example.com`, etc.
- Exact matches work as expected
- Pattern matching is case-insensitive

### Interaction with Enforcement Modes

The SNI filter and enforcement modes operate independently:

- **SNI matches + `REJECT_CONNECTION`**: Request client cert AND require it (connection rejected if not presented)
- **SNI matches + `POLICY`**: Request client cert, policy decides access
- **SNI doesn't match**: No client cert request (regardless of enforcement mode)

This means the SNI list controls the "request" step; enforcement mode controls what happens when a cert is or isn't presented.

## Implementation

### Envoy Configuration

The main HTTPS listener in `config/envoyconfig/listeners_main.go` currently uses a single filter chain. This needs to be split into two filter chains when `tls_client_auth_snis` is configured:

```
Listener
├── FilterChain (SNI-matched)
│   ├── FilterChainMatch.ServerNames = ["*.example.com", ...]
│   ├── TransportSocket with TrustedCa (Envoy requests client cert)
│   └── HTTP Connection Manager
└── FilterChain (fallback)
    ├── No FilterChainMatch (catches all other SNIs)
    ├── TransportSocket without TrustedCa (no cert request)
    └── HTTP Connection Manager
```

### Key Implementation Points

1. **Filter chain ordering**: Envoy processes filter chains in order. The SNI-matched chain must come before the fallback chain.

2. **Multiple patterns**: When multiple glob patterns are specified, they are combined into a single `ServerNames` list in `FilterChainMatch`.

3. **Wildcard handling**: The existing glob pattern logic in `buildSubjectAltNameMatcher` (or similar) should be reused for SNI pattern compilation.

4. **Enforcement mode preservation**: The `RequireClientCertificate` field is still set based on enforcement mode within whichever filter chain matches.

5. **No SNI specified**: When `tls_client_auth_snis` is empty, generate the current single-filter-chain configuration (no behavior change).

### Files to Modify

1. `pkg/grpc/config/config.proto` - Add `tls_client_auth_snis` field
2. `config/mtls.go` - Add `TLSClientAuthSNIs` field to struct
3. `config/envoyconfig/listeners_main.go` - Split filter chain when SNIs specified
4. `config/envoyconfig/tls.go` - May need new helper for SNI glob matching
5. `config/options.yaml` - Document new configuration option
6. `config/options.go` - Add field binding for YAML parsing

### Backward Compatibility

- When `tls_client_auth_snis` is not specified or empty, behavior is identical to today
- No changes to existing `match_subject_alt_names` behavior
- No changes to enforcement mode behavior

## Testing

1. **Unit tests**: Test glob pattern matching for SNI filtering
2. **Integration tests**: Verify Envoy filter chains are generated correctly with/without SNI configuration
3. **Manual testing**: Test TLS handshake behavior with a client like `openssl s_client` or curl, observing whether `Request CERT` appears in handshake
