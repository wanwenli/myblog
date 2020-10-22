---
layout: post
title: Too many redirects when forcing SSL redirection on an ingress
categories: kubernetes
description: Why does my web page keep redirecting when forcing SSL redirection on an ingress with nginx-ingress-controller?
tags:
- kubernetes
- k8s
- nginx-ingress-controller
---

In summary, these are the steps for SSL redirection to work properly and
this method has been mentioned by
[this answer on stackoverflow](https://stackoverflow.com/a/64224737/1397473).

* Add `use-forwarded-headers: "true"` into the ConfigMap of ingress controller
* Do a rolling upgrade on the deployment of Nginx ingress controller,
so that the new value in the ConfigMap would be picked up.
* Add `nginx.ingress.kubernetes.io/force-ssl-redirect: "true"` into the annotations of an ingress

I found this by reading the source code of
[ingress-nginx](https://github.com/kubernetes/ingress-nginx).
Here is a detailed explanation on how it works.

The `TOO_MANY_REDIRECTS` error happens because no matter HTTP or HTTPS is used,
Nginx controller always return
[308 REDIRECT](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/308)
to clients.
Next, the browser goes into an infinite loop of redirections.
Why? In the following section it will be explained.

When `force-ssl-redirect` annotation is set to true on an ingress level,
Nginx ingress controller adds/updates the following snippet into `nginx.conf`.

{% highlight lua %}
rewrite_by_lua_block {
    lua_ingress.rewrite({
        force_ssl_redirect = true,
        use_port_in_redirects = false,
    })
    balancer.rewrite()
    plugins.run()
}
{% endhighlight %}

Note: when SSL redirect is false in an ingress, the value of force_ssl_redirect is also false.

These are the a few code snippets you will need to look into to understand how this fix works.

* In the source code of ingress-nginx,
[here](https://github.com/kubernetes/ingress-nginx/blob/bf11e2ef636cd535b66ebb2ab638a941662da699/rootfs/etc/nginx/lua/lua_ingress.lua#L122)
defines how the Lua block in nginx.conf works.
* When force_ssl_redirect is true, function `redirect_to_https` alone determines whether the request should be redirected.
* According to the implementation of
[redirect_to_https](https://github.com/kubernetes/ingress-nginx/blob/bf11e2ef636cd535b66ebb2ab638a941662da699/rootfs/etc/nginx/lua/lua_ingress.lua#L57),
its return value solely depends on `ngx.var.pass_access_scheme`,
because `ngx.var.scheme` is either http or https. Later I will argue that it has always been http.
* According
[this chunk](https://github.com/kubernetes/ingress-nginx/blob/bf11e2ef636cd535b66ebb2ab638a941662da699/rootfs/etc/nginx/lua/lua_ingress.lua#L97),
it takes the value of `http_x_forwarded_proto` if `use-forwarded-headers` is true and
that's why it is enabled in ingress controller.

Since the following snippet works as a hacky way,
`http_x_forwarded_proto` must reflect the true protocol of the request.

```nginx
if ($http_x_forwarded_proto != 'https') {
  return 301 https://$host$request_uri;
}
```

Q.E.D.
