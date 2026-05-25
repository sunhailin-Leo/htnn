# HTNN

**Builds**

[![test](https://github.com/mosn/htnn/actions/workflows/test.yml/badge.svg)](https://github.com/mosn/htnn/actions/workflows/test.yml)

**Code quality**

[![coverage](https://codecov.io/gh/mosn/htnn/branch/main/graph/badge.svg)](https://codecov.io/gh/mosn/htnn)
[![go report card](https://goreportcard.com/badge/github.com/mosn/htnn)](https://goreportcard.com/report/github.com/mosn/htnn)

---

HTNN (Hyper Trust-Native Network) is Ant Group's internally developed cloud-native cross-layer networking solution, based on Envoy and Istio, and support extending via Go runtime. HTNN architecturally embraces cloud-native standards, supports multi-cluster management and flexible scalability. As a result, it has an ecosystem that supports high development efficiency. By open sourcing HTNN, Ant Group hopes that the community can share its product capabilities and work together to build advanced network products.

## Documentation

* [Introduction](https://github.com/mosn/htnn/blob/main/site/content/en/docs/getting-started/introduction.md)
* [Quick Start](https://github.com/mosn/htnn/blob/main/site/content/en/docs/getting-started/quick_start.md)
* [Get Involved](https://github.com/mosn/htnn/blob/main/site/content/en/docs/developer-guide/get_involved.md)

If you want to extend Envoy via Go, you can consider using the dataplane of HTNN only. Please read the documentation:

* [Dataplane Support](https://github.com/mosn/htnn/blob/main/site/content/en/docs/developer-guide/dataplane_support.md)

## Multi-Version Envoy & Go Support

HTNN supports multiple Envoy versions through a build tag mechanism. The data plane Go code can be compiled into a shared library targeting different Envoy versions.

### Supported Versions

| Envoy Version | Build Tag | Min Go Version | Envoy SDK (replace in go.mod) |
|---------------|-----------|----------------|-------------------------------|
| dev (latest)  | `envoydev` | 1.24.6 | `v1.38.0` (or latest) |
| 1.38          | `envoy1.38` | 1.24.6 | `v1.38.0` |
| 1.37          | `envoy1.37` | 1.24.6 | `v1.37.2` |
| 1.36          | `envoy1.36` | 1.22   | `v1.36.6` |
| 1.35          | `envoy1.35` | 1.22   | `v1.35.3` |
| 1.32 (default)| _(none)_   | 1.22   | `v1.32.0` (already in go.mod) |
| 1.31          | `envoy1.31` | 1.22   | `v1.31.x` |
| 1.29          | `envoy1.29` | 1.22   | `v1.29.x` |

### For External Users (importing HTNN as a module)

If you are importing `mosn.io/htnn/api` or `mosn.io/htnn/plugins` in your own project:

#### Step 1: Add replace directive in YOUR project's `go.mod`

```go.mod
module your-project

go 1.22

require (
    mosn.io/htnn/api v0.5.0
    mosn.io/htnn/plugins v0.5.0
)

// Target Envoy 1.36 - add this line:
replace github.com/envoyproxy/envoy => github.com/envoyproxy/envoy v1.36.6
```

#### Step 2: Compile with the corresponding build tag

```bash
# For Envoy 1.35~1.38:
CGO_ENABLED=1 go build -tags so,envoy1.36 --buildmode=c-shared -o libgolang.so .

# For default version (1.32), no extra tag needed:
CGO_ENABLED=1 go build -tags so --buildmode=c-shared -o libgolang.so .

# For older versions:
CGO_ENABLED=1 go build -tags so,envoy1.29 --buildmode=c-shared -o libgolang.so .
```

### For HTNN Developers (working within this repo)

If you are developing HTNN itself, use the convenience script:

```bash
# Switch SDK version (adds replace to api/go.mod and plugins/go.mod)
./patch/switch-envoy-go-version.sh 1.36.6

# Build shared library
cd plugins && ENVOY_API_VERSION=1.36 make build-so-local

# Run tests
cd api && ENVOY_API_VERSION=1.36 make unit-test
```

> **Note**: `switch-envoy-go-version.sh` is only for use within the HTNN repo. External users should directly edit their own `go.mod`.

### Go Version Requirements

- **Envoy 1.29 ~ 1.36**: Go 1.22+
- **Envoy 1.37 ~ 1.38**: Go 1.24.6+ (enforced by Envoy's `go.mod`)

### CI Test Matrix

The CI runs tests across multiple Go and Envoy version combinations:

- **Go versions**: 1.22, 1.23, 1.24, 1.25
- **Envoy versions**: 1.29, 1.31, 1.32, 1.35, 1.36, 1.37, 1.38, dev
- Incompatible combinations (e.g., Go 1.22 + Envoy 1.37) are automatically excluded.

### Adding Support for Future Envoy Versions

When a new Envoy version (e.g., 1.39) is released:

1. **Check API compatibility**: Compare `contrib/golang/common/go/api/filter.go` between the new version and the current latest supported version. If the Go SDK API is unchanged, the existing build tag code path can be reused.

2. **Add the build tag** (if API is unchanged, add to existing tag list):
   - `api/pkg/filtermanager/api/api_latest.go`: Add `|| envoy1.39`
   - `api/pkg/filtermanager/api_impl_latest.go`: Add `|| envoy1.39`
   - `api/pkg/filtermanager/api/api_131_132.go`: Add `&& !envoy1.39`
   - `api/pkg/filtermanager/api_impl_131_132.go`: Add `&& !envoy1.39`

3. **Update scripts and CI**:
   - `patch/switch-envoy-go-version.sh`: Add `1.39` to the supported version regex
   - `.github/workflows/test.yml`: Add `1.39` to the envoy_version matrix (with Go version exclusions if needed)

4. **Update documentation**:
   - `site/content/en/docs/developer-guide/dataplane_support.md`
   - `site/content/zh-hans/docs/developer-guide/dataplane_support.md`
   - This `README.md`

5. **If there ARE breaking API changes**: Create new version-specific files (e.g., `api_impl_135_138.go` with a dedicated build tag range) following the pattern of existing compatibility layers.

### Compatibility Notes

- The **default version** in `go.mod` remains **1.32**. Users targeting other versions must use `replace` + build tags.
- Envoy 1.35 ~ 1.38 share the **same Go SDK API** (confirmed by file-level comparison). They use a common code path.
- The **control plane** (Istio integration) is independent of the Envoy data plane version.
- Running a newer-Envoy-only interface on an older Envoy will trigger the compatibility layer, which logs an error and returns a null value.


## Thanks

**HTNN** would not be possible without the valuable open-source work of projects in the community. We would like to extend a special thank-you to:

* [Envoy](https://www.envoyproxy.io).
* [Istio](https://istio.io).
