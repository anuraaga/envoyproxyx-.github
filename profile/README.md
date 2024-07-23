# EnvoyX: Envoy Proxy meets Dynamically Loadable Modules

This organization is dedicated to the development of the EnvoyX project, 
which aims to provide a way to extend the functionality of the [Envoy Proxy©](https://www.envoyproxy.io/) by dynamically loadable modules.
This can be thought as the NGINX style extension mechanism for the Envoy Proxy. In other words,
this allows the shared object files to be used as the HTTP filters for the Envoy Proxy.

### Table of Contents

1. [Goals / Non-Goals](#goals--non-goals)
2. [How to use EnvoyX](#how-to-use-envoyx)
3. [Versioning](#versioning)
4. [Why not upstream?](#why-not-upstream)
5. [How do you compare with other extension mechanisms?](#how-do-you-compare-with-other-extension-mechanisms)

## Goals / Non-Goals
Goals:
* The **fastest** way to write HTTP filters for Envoy without having to recompile the whole Envoy binary.
  * EnvoyX provides the **zero-copy** based API for the HTTP filters, which makes it suitable for the high-performance sensitive use cases such as HTTP body modification, etc.
* The **most developer-friendly** way to write HTTP filters for Envoy without having to recompile the whole Envoy binary.
  * Being able to deploy shared object files means almost the same development experience as the usual software development.
* Move fast and break things: the API is subject to change while maintaining minimal compatibility (See the [Versioning](#versioning) section).

Non-Goals:
* Security: by design, the plugins must be trusted.
* Long-term API stability: the API is subject to change while maintaining minimal compatibility (See the [Versioning](#versioning) section).
* Non-HTTP extensions: we focus on the HTTP filters only. period.

## How to use EnvoyX

See [Versioning](#versioning) for the detail of the versioning of the EnvoyX project to use.

Note that we are still in the early stage of the development, and looking for the real-world feedbacks.

### Deploying the custom build of the Envoy Proxy

WARNING: Currently, the container only supports amd64 architecture.

EnvoyX project provides the custom drop-in replacement build of the release versions of the Envoy Proxy
with the support of the dynamic loading of the modules. We provide the docker containers,
and you can replace the default Envoy Proxy binary with the custom build like:

```diff
- envoyproxy/envoy:ENVOY_VERSION
+ ghcr.io/envoyproxy/envoy:ENVOY_VERSION-envoyx-ENVOYX_VERSION
```

where
* `ENVOY_VERSION` is the version of the Envoy Proxy (e.g., `v1.29-latest`, `v1.30-latest`, etc.)
* `ENVOYX_VERSION` is the version of the EnvoyX project (e.g., `v1.0.0`)

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
    # The file_path is the path to the shared object file. You can share the same file for all http filter chains.
    file_path: /path/to/libmain.so
    # This is passed to the shared object file as the configuration.
    filter_config: "helloworld"
    # Since c-shared modules by the Go compiler toolchain do not support dlclose, https://github.com/golang/go/issues/11100
    # we need to set do_not_dlclose to true to avoid the crash.
    do_not_dlclose: true
```

For the full example, see the example directory in the [Go SDK repository](https://github.com/envoyproxyx/go-sdk/blob/main/example/envoy.yaml).

## Versioning

First of all, in the EnvoyX project, we cut the versions in the form of 1.X.Y, where X and Y are the minor and patch versions, respectively.
In other worse, major version is always 1. So, ignore the major version and focus on the minor version.

For minor versions, we cut release for **all repositories** at the same time including the docker container
as well as the SDKs.

For patch versions, we cut the releases for the SDKs and the EnvoyX build independently of the other repositories.

### Alignments with Envoy Proxy versions

First, we cut releases independently of the Envoy Proxy versions. That means,
there might be multiple versions of the EnvoyX project for a single version of the Envoy Proxy,
e.g. `ghcr.io/envoyproxyx/envoy:v1.31-latest-envoyx-v1.99.0`, `ghcr.io/envoyproxyx/envoy:v1.31-latest-envoyx-v1.100.0`, etc.

We support the latest three versions of the Envoy Proxy whichever available at the time of the release of the EnvoyX project.
For example, at the time of writing, the latest three versions of the Envoy Proxy are `v1.29-latest`, `v1.30-latest`, and `v1.31-latest`,
hence the EnvoyX container image tags will be `ghcr.io/envoyproxyx/envoy:v1.29-latest-envoyx-v1.X.Y`, 
`ghcr.io/envoyproxyx/envoy:v1.30-latest-envoyx-v1.X.Y`, and `ghcr.io/envoyproxyx/envoy:v1.31-latest-envoyx-v1.X.Y`.

### Compatibility Promise

We promise the following compatibility for the EnvoyX project:

Each minor version of the EnvoyX project is compatible with the deprecation introduced in the previous minor version of the EnvoyX project.
For example, the EnvoyX proxy `v1.50.0` should be able to run the dynamic modules compiled with the SDK version `v1.49.0` without issue,
but we do not promise the compatibility with `v1.48.0`.

That way, you can upgrade the EnvoyX proxy to the latest version without worrying about the compatibility with the existing modules while
allowing us to move fast and break things. 

## Why not upstream?

Of course, it would be great if the upstream Envoy Proxy project supports the dynamic loading of the modules.
That is always the ultimate goal of the EnvoyX project.

However, until the various use cases are identified and the APIs become stable,
we would like to move fast and break things in the EnvoyX project. After the API is stabilized and the use cases are clear,
we will propose the changes to the upstream Envoy Proxy project. See [the upstream issue](https://github.com/envoyproxy/envoy/issues/2053).

## How do you compare with other extension mechanisms?

There are four main extension mechanisms for the Envoy Proxy:
* Lua
* WebAssembly
* External Process
* Statically-compiled native extensions

If you can do the statically compiled native extensions, then EnvoyX serves no purpose for you.
This is only for those who cannot or do not want to build and maintain the custom Envoy builds by themselves.

The other four mechanisms have their own pros and cons,
but the main difference is the performance and the development experience.

For the performance, the other extensions mechanisms, by design, have to copy the data between the Envoy Proxy and the extension.
That is not the case for the EnvoyX project.
The EnvoyX project provides the zero-copy based API for the HTTP filters, which makes it suitable for the high-performance sensitive use cases such as HTTP body modification, etc.

For developer experience, Lua and WebAssembly are much less attractive than dynamic modules and external process.
That is because we can use the normal developer tools for the development of the plugins.

As for the security, Lua and WebAssembly are much more secure than the dynamic modules and external process.
That is because of its sandbox nature and the isolation provided by the Envoy implementation. external process and
dynamic modules are not sandboxed at all and should be trusted. However, if you can control the deployment of Envoy plugins,
then the security is not a big concern. If you are still concerned about the security even if you can control the deployment,
the question you should ask yourself is "what's the difference between the deployment of dynamic modules and your applications?".

Forgot to mention, there's contrib [Golang filter](https://www.envoyproxy.io/docs/envoy/latest/start/sandboxes/golang-http),
which allows the dynamically loaded Go HTTP filters. However, that should not be limited to Golang at all, plus
the APIs is almost the same as the Wasm and Lua, which comes the unnecessary overheads. In any case, that is still in a contrib.

As you can see, this is not one-size-fits-all, but we believe that the EnvoyX project fills
the gap between the performance and the developer experience provided by the other extension mechanisms.
