---
layout: post
title: A pod IP is released after the pod is deleted
description: An experiment on Calico CNI and etcd, inspired by the source code of Calico
categories: kubernetes
tags:
- k8s
- calico
- CNI
- networking
---

# Prerequisite

Calico CNI is installed in your k8s cluster.

# IP assigned by k8s

In the experiment I have performed,
the pod IP is `100.112.209.159`.

# Go into an etcd pod

Try `kubectl exec` into an etcd pod.

```
kubectl exec -it -n kube-system <etcd_pod_name> sh
```

# List all CIDR blocks

```
etcdctl ls /calico/ipam/v2/assignment/ipv4/block/
```

Find out that the aforementioned IP falls under CIDR 100.112.209.128/26

# Get the block info/metadata

```
etcdctl get /calico/ipam/v2/assignment/ipv4/block/100.112.209.128-26
```

# Test it by Go

The code below is inspired by https://github.com/projectcalico/libcalico-go/blob/master/lib/backend/model/block.go

Copy the JSON you have pasted from the section above,
and paste into the first line of function `TestCalicoBlock` below.

{% highlight go linenos %}
func TestCalicoBlock(t *testing.T) {
	bytes := []byte(`{"cidr":"100.112.209.128/26","affinity":"host:ip-10-200-205-203","strictAffinity":false,"allocations":[0,null,1,null,null,null,null,null,null,null,null,null,null,null,null,null,3,null,null,null,null,null,7,null,null,null,null,null,null,null,null,null,null,5,null,null,null,null,null,null,null,2,null,null,null,null,8,null,null,null,null,null,null,null,null,4,null,null,null,null,null,null,6,null],"unallocated":[45,32,47,12,3,48,40,14,31,60,26,10,29,18,30,17,63,5,57,11,39,36,34,1,15,56,7,42,37,9,53,44,21,38,58,49,35,23,54,59,25,51,24,28,61,50,13,19,4,27,43,52,8,20,6],"attributes":[{"handle_id":null,"secondary":null},{"handle_id":"k8s-pod-network.884e11ced699fe17a493ffd59f18c97fd090fd8150411a11643a280a8250c510","secondary":null},{"handle_id":"k8s-pod-network.3abca72fb5d92129ce16d631fcff2a024b422a48389107600d3ab02e997b484a","secondary":null},{"handle_id":"k8s-pod-network.ba885aa5bb2e7f4124867d23f8975c5c3f42bd9d29632d535d90ebe516341c66","secondary":null},{"handle_id":"k8s-pod-network.2c02fa9db03c347bc005c225331bea5422c86fd70b4c194462dbb56ffe8b253a","secondary":null},{"handle_id":"k8s-pod-network.2caf1c8381761431c694771227eec339c5c0bb0ed3f11a6f2067883d5f405b69","secondary":null},{"handle_id":"k8s-pod-network.84a415903469eb4d965002b7ad8d377e36c93746e0a53944d915695903f7d416","secondary":null},{"handle_id":"k8s-pod-network.820a2747f282dc6cce92d5d544ea8e778a7fa787e219cc1c9757a58aa7a08455","secondary":null},{"handle_id":"k8s-pod-network.1ba612190e9a571ea1253e52834cba4d746ffd75b303ec0706c4ffe8845fe655","secondary":null}]}`)
	var allocations model.AllocationBlock
	_ = json.Unmarshal(bytes, &allocations)
	for ord, attrIdx := range allocations.Allocations {
		if attrIdx == nil {
			continue
		}
		fmt.Println(OrdinalToIP(ord, &allocations))
	}
}

type IP struct {
	net.IP
}

func OrdinalToIP(ord int, b *model.AllocationBlock) IP {
	sum := big.NewInt(0).Add(IPToBigInt(IP{IP: b.CIDR.IP}), big.NewInt(int64(ord)))
	return BigIntToIP(sum)
}

func IPToBigInt(ip IP) *big.Int {
	if ip.To4() != nil {
		return big.NewInt(0).SetBytes(ip.To4())
	} else {
		return big.NewInt(0).SetBytes(ip.To16())
	}
}

func BigIntToIP(ipInt *big.Int) IP {
	ip := IP{net.IP(ipInt.Bytes())}
	return ip
}
{% endhighlight %}

# Output

Before running `kubectl delete` command to delete the pod,
`100.112.209.159` was in the list.

However, _after_ the pod has been removed,
the output becomes as follows.

```
100.112.209.128
100.112.209.130
100.112.209.144
100.112.209.150
100.112.209.161
100.112.209.169
100.112.209.174
100.112.209.183
100.112.209.190
```

# Conclusion

`100.112.209.159` is gone, because it has been released.
