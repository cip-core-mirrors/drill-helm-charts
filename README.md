# Helm Charts for Apache Drill

## Overview

This repository is a fork of [Agirish/drill-helm-charts](https://github.com/Agirish/drill-helm-charts), in order to make the chart usable for [helmfile](https://github.com/roboll/helmfile). It also acts as a helm repository for the helm chart.

The original [README](README-original.md) is also available. The below documentation assumes you're familiar with kubernetes, helm and helmfile.

## Chart Structure

Compared to the upstream project, the drill chart is now contained under the directory `charts/`. The zookeeper sub chart is removed, in favor of dealing with this on helmfile level.

### Templates

A `drill-configmap.yaml` is added which can be used to declaratively override default configuration, instead of depending on a shell script to manually configure specific files.

### Values

The values.yaml file has been adapted for the removal of the zookeeper sub chart, some unused values have been removed, and the init container of the drill statefulset (which checks for zookeeper availability) has been adapted so the zookeeper hostname and port can be configured (`.Values.zookeeper.hostname` and `.Values.zookeeper.port`). The contents for `drill-configmap.yaml` can be passed along as either inline content or use the templating functionality of helmfile to read the contents from files; they can be passed as a map under `.Values.drillConf.entries`, where the key is the filename for the configmap entry, and the value is the actual contents.

## Usage
### Install

Adapt your helmfile installation in your preferred way to add the helm repository, declare the release and override any values, if any.

#### Helm Repository

```
repositories:
- name: drill-helm-charts
  url: https://dimensie10.github.io/drill-helm-charts/
```

After adding the repository, run `helmfile repos` as usual.

#### Declare drill release without overridden values

```
releases:
  - name: drill1
    namespace: default
    missingFileHandler: Error
    version: 2.0.0
    chart: drill-helm-charts/drill
```

Adapt the release name and release namespace as you see fit.

#### Declare drill release with overridden values

```
releases:
  - name: drill1
    namespace: default
    missingFileHandler: Error
    version: 2.0.0
    chart: drill-helm-charts/drill
    values:
      - ./values.yaml
      - ./values.yaml.gotmpl
```

Use `values.yaml.gotmpl` if you want to use helm templating and/or go functions in your values file, as helmfile supports.

#### Overriding values

Default values defined in the helm chart's values.yaml file can be overridden in the usual way.

##### Overriding service type

Opposed to how the upstream chart defines the web-service (environment set to gke or on-prem), this chart sets the service using value `.Values.drill.service.type`, which by default is set to `LoadBalancer`. When working with Ingress objects, set this to `ClusterIP`. For the `on-prem` option of the upstream chart, set this to `NodePort`.

##### Overriding specific configuration files

Specifically two drill configuration files can be overridden in the chart, i.e. as follows (in the values.yaml.gotmpl file):

{% raw %}
```
drill:
  drillConf:
    # Flag to turn-on / turn-off this option
    override: true
    entries:
      "drill-override.conf": |
        drill.exec: {
        spill.directories: [ "/tmp/drill_spill" ],
        impersonation.enabled: true,
        impersonation.max_chained_user_hops: 3,
        }
      "drill-env.sh": |
{{ readFile "drill-env.sh" | indent 8 }}
```
{% endraw %}

`.Values.drill.drillConf.override` needs to be set to `true` in case you want to use overridden configuration files. As you can see, there are multiple ways of delivering the contents. With the latter, make sure helmfile can find the file specified as argument to readFile; also make sure your values filename ends in `.gotmpl`, so helmfile supports the called go/helm template function.

##### Overriding zookeeper client configuration

```
zookeeper:
  hostname: "zk-service"
  port: 2181
```

This is the default configuration in values.yaml for drill to use an existing zookeeper. Most probably at least the hostname needs to be changed.

#### Ingress support

This chart supports creating an ingress for the drill webservice, which is disabled by default. Hostnames and paths can be configured, the service port is taken from `.Values.drill.httpPort`. TLS can be enabled for the hostnames as well. Example:

```
ingress:
  enabled: true
  annotations:
    cert-manager.io/cluster-issuer: my-cluster-issuer
  hosts:
    - host: drill.mydomain.tld
      paths:
        - "/"
  tls:
    - secretName: drill-ingress-tls
      hosts:
      - drill.mydomain.tld
```
