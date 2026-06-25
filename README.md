# dpm-examples

This repository contains Daml `dpm` examples, including a pair of projects that test consuming a DAR from an OCI registry:

- `examples/oci-dependency` builds and publishes `oci-dependency-interfaces`.
- `examples/oci-dep-consumer` consumes that DAR through an OCI dependency.

## Test The OCI DAR Dependency Example

These steps assume a Daml SDK / `dpm` build that supports OCI DAR dependencies. The example projects currently use SDK version `0.0.0` since, at the time of writing, it was required to build the SDK from source. Use the same SDK version throughout the test to avoid build errors.

```sh
export DPM_SDK_VERSION=0.0.0
dpm version
```

### 1. Build The Producer DAR

```sh
cd examples/oci-dependency
dpm build --all
```

The interface DAR should be created at:

```text
interfaces/.daml/dist/oci-dependency-interfaces-1.0.0.dar
```

### 2. Start A Local OCI Registry

```sh
docker run --rm -d -p 5001:5000 --name dpm-oci-registry registry:2
curl http://localhost:5001/v2/
```

### 3. Publish The Producer DAR

From `examples/oci-dependency`:

```sh
dpm publish dar \
  'oci://localhost:5001/dars/oci-dependency-interfaces:1.0.0' \
  -f interfaces/.daml/dist/oci-dependency-interfaces-1.0.0.dar \
  --exclude-license \
  --insecure
```

Optionally, expose the local registry through an HTTPS tunnel:

```sh
ngrok http 5001
```

If ngrok gives you `https://example-tunnel.ngrok-free.dev`, publish to that host instead and omit `--insecure`:

```sh
dpm publish dar \
  'oci://example-tunnel.ngrok-free.dev/dars/oci-dependency-interfaces:1.0.0' \
  -f interfaces/.daml/dist/oci-dependency-interfaces-1.0.0.dar \
  --exclude-license
```

Check that the tag exists:

```sh
curl http://localhost:5001/v2/dars/oci-dependency-interfaces/tags/list
```

You should see a response containing the `1.0.0` tag.

### 4. Install The OCI Dependency In The Consumer

If you are testing against the local HTTP registry, add the DAR from both consumer packages that use `Interfaces`:

```sh
cd ../oci-dep-consumer/main
dpm add dar \
  oci://localhost:5001/dars/oci-dependency-interfaces:1.0.0 \
  --dependencies \
  --insecure

cd ../test
dpm add dar \
  oci://localhost:5001/dars/oci-dependency-interfaces:1.0.0 \
  --dependencies \
  --insecure
```

If you are using the ngrok HTTPS tunnel instead, use the tunnel host and omit `--insecure`:

```sh
cd ../oci-dep-consumer/main
dpm add dar \
  oci://example-tunnel.ngrok-free.dev/dars/oci-dependency-interfaces:1.0.0 \
  --dependencies

cd ../test
dpm add dar \
  oci://example-tunnel.ngrok-free.dev/dars/oci-dependency-interfaces:1.0.0 \
  --dependencies
```

This should pin the dependency with an immutable `@sha256:...` digest in each package's `daml.yaml`.

### 5. Resolve And Build The Consumer

```sh
cd ..
dpm resolve
dpm build --all
```

The build succeeds when the OCI DAR has been installed into the DPM cache and the producer and consumer packages were built with the same SDK version.

### Optional: Test Through HTTPS

If you prefer to avoid `--insecure`, expose the local registry through HTTPS:

```sh
ngrok http 5001
```

Then use the ngrok host in the OCI URI:

```text
oci://<your-ngrok-host>/dars/oci-dependency-interfaces:1.0.0
```

When publishing and adding the DAR through HTTPS, omit `--insecure`.

### Common Clean-Up

Stop the local registry when you are done:

```sh
docker stop dpm-oci-registry
```
