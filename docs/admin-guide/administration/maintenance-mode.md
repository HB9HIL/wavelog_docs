# Maintenance Mode

Wavelog brings with version 1.2 a maintenance mode.

The maintenance mode allows to block normal user logins (User Level: 'Operator'). Only Administrators can login normally if the maintenance mode is enabled.

There are three ways to enable it. Any one of them is sufficient.

## Option 1: File in the root directory

Just create a file in the root directory called `.maintenance`

Example for Debian Servers:

    cd /var/www/wavelog
    touch .maintenance

To disable the maintenance mode just remove the file:

    cd /var/www/wavelog
    rm .maintenance

## Option 2: Environment variable

Set the environment variable `MAINTENANCE_MODE` to `true` (also accepted: `1`, `yes`, `on`). The variable is read on every request, so it needs to reach the PHP process, e.g. via the Docker `environment` section or the Kubernetes `env` list.

Example for Docker Compose:

```yaml
services:
  wavelog:
    environment:
      MAINTENANCE_MODE: "true"
```

Example for Kubernetes (triggers a rolling restart of all pods):

    kubectl set env deploy/wavelog MAINTENANCE_MODE=true
    kubectl set env deploy/wavelog MAINTENANCE_MODE-

## Option 3: Custom file path (Kubernetes ConfigMap)

The environment variable `MAINTENANCE_FILE` overrides the path of the maintenance file. This is useful when running multiple replicas: mount a ConfigMap and toggle the maintenance mode for all pods at once, without restarting them.

Deployment snippet:

```yaml
spec:
  containers:
    - name: wavelog
      env:
        - name: MAINTENANCE_FILE
          value: /config/maintenance
      volumeMounts:
        - name: maintenance
          mountPath: /config
  volumes:
    - name: maintenance
      configMap:
        name: wavelog-maintenance
        optional: true
```

Enable and disable:

    kubectl create configmap wavelog-maintenance --from-literal=maintenance=1
    kubectl delete configmap wavelog-maintenance

!!! note
    Kubernetes syncs mounted ConfigMaps periodically, so the change takes effect within about a minute. Do not mount the file with `subPath`, as files mounted that way are not updated.
