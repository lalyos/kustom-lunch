## Exercise 1 - hardcoded env vars

Lets start to play with a [12factorized](https://12factor.net) app by
configuring env variables:

```
k apply -k base
k apply -k with-env
k rollout history deployment lunch
```

Now try to switch between the 2 revision (repeatedly)
```
k rollout undo deployment lunch
k rollout history deployment lunch
```

Try to hand-edit (edit change-cause metadata as well)
```
k edit deployments lunch --output-patch 
```

Or do a patch:
```
patch='{"metadata":{"annotations":{"kubernetes.io/change-cause":"patched"}},"spec":{"template":{"spec":{"containers":[{"env":[{"name":"TITLE","value":"Lets have a lunch v1.1 - patched"}],"name":"lunch"}]}}}}'

k patch deployments lunch --patch="$patch"
```

check rollout history and play with:
```
k rollout undo --to-revision=X deployment lunch

# helper function
undo() { rev=$1; : ${rev:? required};  k rollout undo --to-revision=${rev} deployment lunch; }
```

## Exercise 2 - env vars from ConfigMap

Why? so you can store the deployment.yaml in the main version control,
and have separate ConfigMaps for each environment (dev/qa/prod).

ConfigMaps are namespaced, so you can deploy the same manifest into different
namespaces (dev/qa/prod). Or even better a different person/role prepares the
namespace + configmap, and one can use the same manifest.

Cleanup
```
k delete -k base/
k apply -k base/
k rollout history deployment lunch
```

Lets deploy the ConfigMap variant:
```
k apply -k with-configmap
k rollout history deployment lunch
```

So fas so good, you can try to switch back and forth between revisions.

What happens when you cange the ConfigMap directly:
```
k edit cm lunch-config
## add to end of title: "v1.1"
```
Hmmm nothing hapens (yet...). Think about container lifecycle, and env vars.

## Exercise 3 - Generated ConfigMaps

Kustomize provides a nice solution: generated ConfigMaps.
I can help you to generate a new configmap for each value change,
with the content hash code in the name (lunch-config-d6545dhfch) and
change the deplyment.yaml to use the newy generated cm.

You can see this is quite similar to how ReplicaSet refers to pods
by using pod-template-hash labels.

Cleanup
```
k delete -k base/
k apply -k base/
k rollout history deployment lunch
```

Now lets use kustomize's `configMapGenerator`:
```
k apply -k with-configmap-gen/
k rollout history deployment lunch
```

now instead of directly editing ConfigMaps, change values in `kustomization.yaml`
```
# add "v1.2" to the end of title in kustomization.yaml
k apply -k with-configmap-gen/
k rollout history deployment lunch

REVISION  CHANGE-CAUSE
1         simple deployment
2         use configmap generator
3         use configmap generator
```

Your change is instantly shows up, a new revision is created, but change cause is the same. Lets fix it:

```
k patch deployments. lunch --patch='{"metadata":{"annotations":{"kubernetes.io/change-cause":"generated configmap v1.1"}}}'
k rollout history deployment lunch

REVISION  CHANGE-CAUSE
1         simple deployment
2         use configmap generator
3         generated configmap v1.1
```
It doesn't messes up the revisions, as changing only annotation, doesn't changes the template hash.

So but we are back again by modifying a yaml file (configurator roles, shoudnt care about k8s formats)

## Exercise 4 - Generated ConfigMaps with env files

The same way you can create configmaps `k create cm delme --from-env-file=config.env` you can extract the related env vars from `kustomization.yaml` into a file:

```
k apply -k with-configmap-gen-envfile/
k rollout history deployment lunch
```

Now we don't touch any yaml file, just a plain ordinary env file:
```
## change the env file directly
vi with-configmap-gen-envfile/lunch.env

## apply the changes
k apply -k with-configmap-gen-envfile/
```

So this solution has the benefits:

- deployment.yaml is unchanged between environments (dev/qa/prod)
- the person with "configurator" role doesn't have to deal with k8s yaml
- env var change takes immediate effect: no manual redeploy/scale needed
- config change is rollbackable