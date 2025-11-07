# Use existing Kubernetes Secret in a Helm template

## Overview

The main objective is to create a `Secret` object with its data (e.g. a `password` field) being randomly
generated at the first deployment and once created its value should not change on the consecutive deployments.

## The `lookup` function

The [lookup](https://helm.sh/docs/chart_template_guide/functions_and_pipelines/#using-the-lookup-function) function in Helm is a powerful feature that allows you to query existing Kubernetes resources in a running cluster directly from the Helm templates.


Its syntax requires specifying the API version, resource kind, namespace, and name of the resource you want to look up. E.g:

```yaml
(lookup "v1" "Namespace" "" "namespace").metadata.annotations
```

will return `annotations` present for the `namespace` object.

The `lookup` function enables dynamic and conditional configuration in Helm charts based on the current state of the cluster. If the resource is not found, an empty value is returned, allowing you to implement existence checks. However, note that `lookup` only works when Helm connects to a live cluster, so in case of a dry run you should use `--dry-run=server`.

## Complete solution

```yaml
{{- $existingSecret := (lookup "v1" "Secret" .Release.Namespace "database-credentials") }}
{{- $password := randAlphaNum 64 | b64enc | quote }}
{{- if $existingSecret }}
{{- $password = index $existingSecret.data "password"}}
{{- end -}}
---
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
data:
  password: {{ $password }}
...
```

This code snippet will use the `lookup` function to check for an existing `Secret` and use its value for the purpose of creating a new version of the `Secret` object. In case this is a the first deployment or the `Secret` has been removed in any other way the `randAlphaNum` function will be used to generate a new random password.
