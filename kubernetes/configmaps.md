# ConfigMaps
## What are they?
[ConfigMaps](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) let you specify application config as a Kubernetes resource. Each config map has keys and values.

## Defining a config map
Kubernetes has commands to dynamically generate config maps and post them via API. You can also define them in files alongside deployment config and `kubectl apply -f` them.

### Example: YAML file
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dnsmasq-conf
  labels:
    name: dnsmasq-conf
  namespace: pihole
data:
  02-lan.conf: |-
    addn-hosts=/etc/pihole-local/lan.list
```

## Using a config map
Config maps can be used to populate environment variables or files. By recommended use, when config maps are consumed as files, a config map is mounted as a volume where the files take the names of the keys. The file contents are the corresponding config values.

### Example: volume mount
The volume
```yaml
volumes:
- name: dnsmasq-conf
  configMap:
    name: dnsmasq-conf
```

The mount
```yaml
volumeMounts:
- name: dnsmasq-conf
  mountPath: /etc/dnsmasq.d
```

When in the container:
```bash
$ cat /etc/dnsmasq.d/02-lan.conf
addn-hosts=/etc/pihole-local/lan.list
```

### Quirks
#### Config map mounts replace directory contents
When mounting a config map at e.g. `/etc/dnsmasq.d`, the directory will be replaced with a directory _only_ containing the config map contents. If there were any files in there previously, they will be deleted. Sometimes you want to place files in existing directories, or combine multiple config maps into one directory.

There are two ways around this:
1. Subpaths
```yaml
volumeMounts:
- name: dnsmasq
  mountPath: /etc/dnsmasq.d/
- name: dnsmasq-conf
  mountPath: /etc/dnsmasq.d/02-lan.conf
  subPath: 02-lan.conf
```
This will put `02-lan.conf` in `/etc/dnsmasq.d`, but preserve the contents already there. As a consequence, it appears updating the config map via API does not live-update the config map in the container. You need to recreate the pod.

2. Projected volume
```yaml
volumeMounts:
- name: grafana-dashboards
  mountPath: /etc/grafana/provisioning/dashboards/
```
```yaml
volumes:
- name: grafana-dashboards
  projected:
    sources:
    - configMap:
        name: grafana-dashboards-conf
```
While this allows adding multiple config maps (or secrets) to a single directory, any contents already in that directory will still be deleted.
