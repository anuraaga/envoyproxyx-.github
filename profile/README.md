# EnvoyX: Envoy Proxy meets Dynamically Loadable Modules

This organization is dedicated to the development of the EnvoyX project, 
which aims to provide a way to extend the functionality of the [Envoy Proxy©](https://www.envoyproxy.io/) by dynamically loadable modules.
This can be thought as the NGINX style extension mechanism for the Envoy Proxy. In other words,
this allows the shared object files to be used as the HTTP filters for the Envoy Proxy.

## Goals / Non-Goals
Goals:
* The **fastest** and **most efficient** way to write HTTP filters for Envoy without having to recompile the whole Envoy binary.
  * EnvoyX provides the **zero-copy** based API for the HTTP filters, which makes it suitable for the high-performance sensitive use cases such as HTTP body modification, etc.
* Move fast and break things: the API is subject to change while maintaining minimal compatibility (See the [Versioning](#versioning) section).

Non-Goals:
* Security: by design, the plugins must be trusted.
* Long-term API stability: the API is subject to change while maintaining minimal compatibility (See the [Versioning](#versioning) section).
* Non-HTTP extensions: we focus on the HTTP filters only. period.

## How to use EnvoyX

See [Versioning](#versioning) for the detail of the versioning of the EnvoyX project to use.

### Deploying the custom build of the Envoy Proxy

EnvoyX project provides the custom drop-in replacement build of the release versions of the Envoy Proxy
with the support of the dynamic loading of the modules. We provide the docker containers,
and you can replace the default Envoy Proxy binary with the custom build like:

```diff
- envoyproxy/envoy:ENVOY_VERSION
+ ghcr.io/envoyproxy/envoy:ENVOY_VERSION-envoyx-ENVOYX_VERSION
```

where
* `ENVOY_VERSION` is the version of the Envoy Proxy (e.g., `v1.29-latest`, `v1.30-latest`, etc.)
* `ENVOYX_VERSION` is the version of the EnvoyX project (e.g., `v0.1.0`)

For example, `ghcr.io/envoyproxyx/envoy:v1.31-latest-envoyx-v1.0.0` can be deployed as a
drop-in replacement for `envoyproxy/envoy:v1.31-latest` which enables the use of the version `v1.0.0` of the EnvoyX project.

For the detail of the version of the EnvoyX project, please refer to the [Versioning](#versioning) section.

For all the available docker images, please refer to the [GitHub Container Registry](https://github.com/envoyproxyx/envoyx/pkgs/container/envoy).

### Developing the custom module

The custom dynamic modules can be developed with the EnvoyX SDK.
Currently, the SDK is available for the Go language, but we are planning to support more languages in the near future.

See [Go SDK](https://github.com/envoyproxyx/go-sdk) for details.

### Envoy configuration

The custom modules can be loaded by the Envoy Proxy by specifying the HTTP filter configuration 
like the following in the filter chain:

```yaml
- name: envoy.http.dynamic_modules
  typed_config:
    # Schema is defined at https://github.com/envoyproxyx/envoyx/blob/main/x/config.proto
    "@type": type.googleapis.com/envoy.extensions.filters.http.dynamic_modules.v3.DynamicModuleConfig
    # The name is *optional* and is only used for logging by Envoy, not for modules.
    name: test
    # The file_path is the path to the shared object file. We share the same file for both http filter chain.
    file_path: /path/to/libmain.so
    # This is passed to newHttpFilter in main.go
    filter_config: "helloworld"
    # Since c-shared modules by the Go compiler toolchain do not support dlclose, https://github.com/golang/go/issues/11100
    # we need to set do_not_dlclose to true to avoid the crash.
    do_not_dlclose: true
```

For the full example, see the example directory in the [Go SDK repository](https://github.com/envoyproxyx/go-sdk/blob/main/example/envoy.yaml).

## Versioning

First of all, in the EnvoyX project, we cut the versions in the form of 1.X.Y, where X is the major version of the Envoy Proxy. 
In other worse, major version is always 1. So, ignore the major version and focus on the minor version.

For minor versions, we cut release for **all repositories** at the same time including the docker container
as well as the SDKs.

In addition, we cut releases independently of the Envoy Proxy versions. That means,
there might be multiple versions of the EnvoyX project for a single version of the Envoy Proxy,
e.g. `ghcr.io/envoyproxyx/envoy:v1.31-latest-envoyx-v1.99.0`, `ghcr.io/envoyproxyx/envoy:v1.31-latest-envoyx-v1.100.0`, etc.

For patch versions, we cut the releases for the SDKs and the EnvoyX build independently of the other repositories.

### Compatibility Promise

We promise the following compatibility for the EnvoyX project:

Each minor version of the EnvoyX project is compatible with the deprecation introduced in the previous minor version of the EnvoyX project.
For example, the EnvoyX proxy `v1.50.0` should be able to run the dynamic modules compiled with the SDK version `v1.49.0` without issue,
but we do not promise the compatibility with `v1.48.0`.

That way, you can upgrade the EnvoyX proxy to the latest version without worrying about the compatibility with the existing modules while
allowing us to move fast and break things. 

## Why not upstream?

Of course, it would be great if the upstream Envoy Proxy project supports the dynamic loading of the modules.
That is always the best solution. 

However, until the various use cases are identified and the APIs become stable,
we would like to move fast and break things in the EnvoyX project. After the API is stabilized and the use cases are clear,
we will propose the changes to the upstream Envoy Proxy project. See [the upstream issue](https://github.com/envoyproxy/envoy/issues/2053).
