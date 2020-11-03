---
layout: post
title: Why a cluster role binding is needed in k8s-vault integration
description: An in-depth walkthrough in Kubernetes TokenReview API
categories: kubernetes
tags:
- k8s
- vault
---

# TL; DR
The `system:auth-delegator` ClusterRole allows a client to use a service account token to query the
[TokenReview API](https://docs.openshift.com/container-platform/4.4/rest_api/authorization_apis/tokenreview-authentication-k8s-io-v1.html).
If a token is not bound to this ClusterRole,
the token cannot be reviewed by the k8s.

# Background

In the
[official Kubernetes Auth Method](https://www.vaultproject.io/docs/auth/kubernetes.html)
of
[Vault](https://www.vaultproject.io/)
and other online sources,
it is found that a `ClusterRoleBinding` is always there.
You may wonder why a service account needs such a cluster-level permission
in order to authenticate with Vault.
This article attempts to dive into k8s API to explain the mechanism.

More can be found in
[this link](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication).

# What is ClusterRole system:auth-delegator all about

```
kubectl get clusterrole system:auth-delegator -o yaml
```

From the output yaml you will be able to see that it allows creating token reviews with k8s API.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2018-01-01T00:00:00Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:auth-delegator
  resourceVersion: "99"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterroles/system%3Aauth-delegator
  uid: da0c44d5-1111-1111-1111-063ccccda29e
rules:
- apiGroups:
  - authentication.k8s.io
  resources:
  - tokenreviews
  verbs:
  - create
- apiGroups:
  - authorization.k8s.io
  resources:
  - subjectaccessreviews
  verbs:
  - create
```

# Play with TokenReview API

Simply put,
this API tells whether a token is from a legit user of the cluster.

Switch to a k8s

```
kubectl config use-context <target_k8s_cluster>
```

Get the service account token of an application. Alternatively, get the service account secret and parse the JSON.

```
kubectl exec -it <pod_name> cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

Copy the token and go into a tmp directory.

```
cd /tmp
```

Write a file with the following content,
fill in the token and
name the file as `token-review.json`.

```json
{
  "apiVersion": "authentication.k8s.io/v1",
  "kind": "TokenReview",
  "spec": {
    "token": "<YOUR_SERVICE_ACCOUNT_TOKEN>"
  }
}
```

Save the file and run the commands below.

```bash
export TOKEN="<YOUR_SERVICE_ACCOUNT_TOKEN>"
curl -k -v \
 -X POST \
 -d @token-review.json \
 -H "Authorization: Bearer $TOKEN" \
 -H 'Accept: application/json' \
 -H 'Content-Type: application/json' \
 https://<kubernetes_api_host_and_port>/apis/authentication.k8s.io/v1/tokenreviews
```

# Discussion

If the service account has a ClusterRoleBinding with `system:auth-delegator`,
the API would return code 200,
with `system:serviceaccount:<service_account_namespace>:<service_account_name>`
in the user field of the response JSON as below.

```json
{
  "kind": "TokenReview",
  "apiVersion": "authentication.k8s.io/v1",
  "metadata": {
    "creationTimestamp": null
  },
  "spec": {
    "token": "<YOUR_SERVICE_ACCOUNT_TOKEN>"
  },
  "status": {
    "authenticated": true,
    "user": {
      "username": "system:serviceaccount:<service_account_namespace>:<service_account_name>",
      "uid": "3a366063-9999-9999-9999-0659048d2f3e",
      "groups": [
        "system:serviceaccounts",
        "system:authenticated"
      ]
    }
  }
}
```

From it we also know that when we configure the Vault role for authenticating a k8s service account,
`service_account_name` and `service_account_namespace` must be configured correctly.
Because the response from k8s API includes the service account name and its namespace.

If there is no such a binding, the response code would be 403 and the response JSON as below.

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "tokenreviews.authentication.k8s.io is forbidden: User \"system:serviceaccount:<service_account_namespace>:<service_account_name>\" cannot create tokenreviews.authentication.k8s.io at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "group": "authentication.k8s.io",
    "kind": "tokenreviews"
  },
  "code": 403
}
```
