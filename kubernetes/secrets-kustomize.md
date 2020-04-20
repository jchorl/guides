# Secrets
## Why?
You need to access secrets in Kubernetes.

## The Problem
You want to define secrets alongside your application config. You could make a `secrets.yaml` file and gitignore it, but then a fresh pull of the repo would apply just fine and your application would break when it first ran. You could just `kubectl create secret`, but then again, you may deploy your application and the secret may not be present.

## The solution
Kubernetes has a `kustomize` config mechanism, that generates configs from files. If the files are not present, the `kubectl apply` will fail, alerting you that the deploy failed.

**NOTE: unfortunately you can't mix kustomize and non-kustomize configs, so these likely aren't worth using :(**

### Example
First, configure kustomize using e.g. `secrets.yaml`:
```yaml
secretGenerator:
- name: db-user-pass
  files:
  - username.txt
  - password.txt
```

Then you can use the secret:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: db-user-pass-g6tg97fft9
```
