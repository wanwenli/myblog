---
layout: post
title: Delete a k8s namespace that is stuck in "Terminating" state
description: A simple bash script to clean it up
categories: kubernetes
tags:
- k8s
---

When you try to delete a namespace in k8s,
it might get stuck in Terminating state.
One of the most likely reasons is that
there are _finalizers_ that are unable to complete thus block the clean-up of a namespace.

Helm cannot release anything to a terminating namespace.
Therefore a terminating namespace is effectively unusable.
The idea of the solution is to forcibly remove its finalizers.
The solution attached below is able to work even
when command `kubectl edit namespace` does not.

```bash
NAMESPACE=$1
echo "force deleting namespace $NAMESPACE"
kubectl proxy &
kubectl get namespace $NAMESPACE -o json | jq '.spec = {"finalizers":[]}' > temp.json
curl -k -H "Content-Type: application/json" -X PUT --data-binary @temp.json 127.0.0.1:8001/api/v1/namespaces/$NAMESPACE/finalize
```

It uses `kubectl proxy` to proxy the kube-api,
so that it is reachable from localhost.
Last but not the least, check whether all the cloud resources,
such as storage associated with PersistentVolumeClaim and load balancers,
are really terminated after executing the command.
The command fastens the termination of a namespace and some cloud resources might not be cleaned up very properly.
