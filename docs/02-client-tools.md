# Installing the Client Tools

In this lab you will install the command line utilities required to complete this tutorial: [cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl), and [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl).


## Install Go

Following the [Official Documentation](https://go.dev/doc/install)

## Install CFSSL

The `cfssl` and `cfssljson` command line utilities will be used to provision a [PKI Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) and generate TLS certificates.

Download and install `cfssl` and `cfssljson`:

```sh
go install github.com/cloudflare/cfssl/cmd/cfssl@latest
go install github.com/cloudflare/cfssl/cmd/cfssljson@latest
```

### Verification

Verify `cfssl` and `cfssljson` version 1.6.3 or higher is installed:

```sh
cfssl version
```

> output

```
Version: dev
Runtime: go1.12.12
```

```sh
cfssljson --version
```
```
Version: dev
Runtime: go1.12.12
```

## Install kubectl

The `kubectl` command line utility is used to interact with the Kubernetes API Server. Download and install `kubectl` using the instructions specific to your platform [here](https://kubernetes.io/docs/tasks/tools/)

### Verification

Verify `kubectl` version 1.21.0 or higher is installed:

```sh
kubectl version --output=yaml
```

> output

```yaml
clientVersion:
  buildDate: "2023-02-22T13:39:03Z"
  compiler: gc
  gitCommit: fc04e732bb3e7198d2fa44efa5457c7c6f8c0f5b
  gitTreeState: clean
  gitVersion: v1.26.2
  goVersion: go1.19.6
  major: "1"
  minor: "26"
  platform: linux/amd64
kustomizeVersion: v4.5.7

The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

Next: [Provisioning Compute Resources](03-compute-resources.md)
