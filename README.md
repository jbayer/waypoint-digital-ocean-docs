# Waypoint Digital Ocean Documentation

How to Set up Waypoint 0.1.4 on Digital Ocean Kubernetes

## Provision a k8s cluster

1st I provisioned a k8s cluster in the UI with a single $10/month node. I named mine: `do-sfo2-default-k8s`

My local `kubectl` has a context set to the new k8s cluster. Validate that it works with `kubectl get nodes`

## Prepare to install Waypoint

The standard Waypoint installation for Kubernetes will not work with Digital
Ocean. Digital Ocean k8s containers that use PVCs with a non-root user need to
[change the file permissions with InitContainers](https://www.digitalocean.com/docs/kubernetes/how-to/add-volumes/#setting-permissions-on-volumes). So I adapted the Digital Ocean example in the
documentation to Waypoint's k8s resource yaml.

Use the `-show-yaml` option on the `waypoint install` command, and save that to a file:

`waypoint install -platform=kubernetes -accept-tos -show-yaml > digital-ocean-waypoint.yml`

Modify `digital-ocean-waypoint.yml` to add an InitContainers section ([my full example here](https://gist.github.com/jbayer/eb7374511b4a9d37efade7a7ae9093a9#file-waypoint-k8s-do-yml-L37-L43)) so the waypoint user can write to the `/data` volume.

```yaml
      initContainers:
      - name: data-permission-fix
        image: busybox
        command: ["/bin/chmod","-R","777", "/data"]
        volumeMounts:
        - name: data
          mountPath: /data
```

## Install Waypoint

Note that the k8s Service of type LoadBalancer will create a Digital Ocean Load Balancer that costs $10/month.

`kubectl apply -f digital-ocean-waypoint.yml`

Wait for the StatefulSet and Service become available.

`watch kubectl get all`

Look for:
* statefulset.apps/waypoint-server is Ready
* service/waypoint gets an external IP

Retrieve the IP and set it as an env variable:

`export WAYPOINT_IP=$(kubectl get service waypoint --template="{{range .status.loadBalancer.ingress}}{{.ip}}{{end}}")`

`echo WAYPOINT_IP: $WAYPOINT_IP`

I setup a DNS record for that IP and mapped that to waypoint.iamjambay.com, or you can use the IP.

## Bootstrap Waypoint Server

Bootstrap the waypoint server and name the context "do":

`waypoint server bootstrap -server-addr=waypoint.iamjambay.com:9701 -server-tls-skip-verify -context-create=do`

Set the server configuration:

`waypoint server config-set -advertise-addr=waypoint.iamjambay.com:9701 -advertise-tls-skip-verify`

Try deploying a k8s example app such as the [nodejs example](https://github.com/hashicorp/waypoint-examples/tree/main/kubernetes/nodejs).

`waypoint init`

`waypoint up`

Check to see if `waypoint logs`, `waypoint exec`, and the Deployment URL function properly.

