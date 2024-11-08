---
reviewers:
- eparis
- pmorie
- ssycamore
title: Configuring Redis using a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->

ConfigMaps make it possible to inject configuration data into application pods.
This document provides a real-world example of how to configure Redis using a ConfigMap. 
It assumes that you have already [configured a pod to use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/).

## {{% heading "objectives" %}}

By the end of this tutorial, you will be able to:

* Create a ConfigMap with Redis configuration values
* Create a Redis pod that mounts and uses the created ConfigMap
* Verify that the configuration was correctly applied

## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

* The example shown on this page works with `kubectl` 1.14 and above.
* Understand [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/).

<!-- lessoncontent -->

## Walkthrough

Follow the steps below to configure a Redis cache using data stored in a ConfigMap.

### Create ConfigMap

First create a ConfigMap with an empty configuration block:

```shell
cat <<EOF >./example-redis-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-redis-config
data:
  redis-config: ""
EOF
```

Apply the ConfigMap created above, along with a Redis pod manifest:

```shell
kubectl apply -f example-redis-config.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```
Examine the contents of the Redis pod manifest and note the following:

First, a `config` volume is created by `spec.volumes[1]`.
The `key` and `path` under `spec.volumes[1].configMap.items[0]` expose the `redis-config` key from the `example-redis-config` ConfigMap as a file named `redis.conf` on the `config` volume.
Then the `config` volume is mounted at `/redis-master` by `spec.containers[0].volumeMounts[1]`.

This has the net effect of exposing the data in `data.redis-config` from the `example-redis-config`
ConfigMap above as `/redis-master/redis.conf` inside the pod.

{{% code_sample file="pods/config/redis-pod.yaml" %}}

### Verify configurations

Examine the created objects:

```shell
kubectl get pod/redis configmap/example-redis-config 
```

You should see the following output:

```
NAME        READY   STATUS    RESTARTS   AGE
pod/redis   1/1     Running   0          8s

NAME                             DATA   AGE
configmap/example-redis-config   1      14s
```

The `redis-config` key in the ConfigMap above was left blank.
You can run the following command to confirm this:

```shell
kubectl describe configmap/example-redis-config
```

You should see an empty `redis-config` key:

```shell
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
```

Use `kubectl exec` to enter the pod.
Then run the `redis-cli` tool to check the current configuration:

```shell
kubectl exec -it redis -- redis-cli
```

Check `maxmemory`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It should show the default value of 0:

```shell
1) "maxmemory"
2) "0"
```

Similarly, check `maxmemory-policy`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

This should also yield its default value of `noeviction`:

```shell
1) "maxmemory-policy"
2) "noeviction"
```

### Add configuration values

Now let's add some configuration values to the `example-redis-config` ConfigMap:

{{% code_sample file="pods/config/example-redis-config.yaml" %}}

Apply the updated ConfigMap:

```shell
kubectl apply -f example-redis-config.yaml
```

Confirm that the ConfigMap was updated:

```shell
kubectl describe configmap/example-redis-config
```

You should see the configuration values we just added:

```shell
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
----
maxmemory 2mb
maxmemory-policy allkeys-lru
```

Check the Redis pod again using `redis-cli` via `kubectl exec` to see if the configuration was applied:

```shell
kubectl exec -it redis -- redis-cli
```

Check `maxmemory`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It remains at the default value of 0:

```shell
1) "maxmemory"
2) "0"
```

Similarly, `maxmemory-policy` remains at the `noeviction` default setting:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

Returns:

```shell
1) "maxmemory-policy"
2) "noeviction"
```

The configuration values have not changed because the pod needs to be restarted to grab updated values from associated ConfigMaps. 
Let's delete and recreate the Pod:

```shell
kubectl delete pod redis
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

Now re-check the configuration values one last time:

```shell
kubectl exec -it redis -- redis-cli
```

Check `maxmemory`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

It should now return the updated value of 2097152:

```shell
1) "maxmemory"
2) "2097152"
```

Similarly, `maxmemory-policy` has also been updated:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

It now reflects the desired value of `allkeys-lru`:

```shell
1) "maxmemory-policy"
2) "allkeys-lru"
```

Clean up your work by deleting the created resources:

```shell
kubectl delete pod/redis configmap/example-redis-config
```

## {{% heading "whatsnext" %}}


* Learn more about [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Follow an example of [Updating configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/).
