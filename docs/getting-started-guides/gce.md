## Getting started on Google Compute Engine

The example below creates a Kubernetes cluster with 4 worker node Virtual Machines and a master Virtual Machine (i.e. 5 VMs in your cluster). This cluster is set up and controlled from your workstation (or wherever you find convenient).

### Getting VMs

1. You need a Google Cloud Platform account with billing enabled. Visit the [Google Developers Console](http://cloud.google.com/console) for more details.
2. Make sure you can start up a GCE VM from the command line.  At least make sure you can do the [Create an instance](https://cloud.google.com/compute/docs/quickstart#create_an_instance) part of the GCE Quickstart.
3. Make sure you can ssh into the VM without interactive prompts.  See the [Log in to the instance](https://cloud.google.com/compute/docs/quickstart#ssh) part of the GCE Quickstart.
  * Your GCE SSH key must either have no passcode or you need to be using `ssh-agent`.
  * Ensure the GCE firewall isn't blocking port 22 to your VMs.  By default, this should work but if you have edited firewall rules or created a new non-default network, you'll need to expose it: `gcloud compute firewall-rules create --network=<network-name> --description "SSH allowed from anywhere" --allow tcp:22 default-ssh`
4. You need to have the Google Cloud Storage API, and the Google Cloud Storage JSON API enabled. It is activated by default for new projects. Otherwise, it can be done in the Google Cloud Console.  See the [Google Cloud Storage JSON API Overview](https://cloud.google.com/storage/docs/json_api/) for more details.

### Prerequisites for your workstation

1. You must be running Linux or Mac OS X on your workstation.
2. You must have the [Google Cloud SDK](https://developers.google.com/cloud/sdk/) installed.  This will get you `gcloud` and `gsutil`.
3. Install `gcloud preview`: run `gcloud components update preview` to make sure it is.
4. Ensure that your other `gcloud` components are up-to-date by running `gcloud components update`.
5. If you want to build your own release, you need to have [Docker installed](https://docs.docker.com/installation/).  On Mac OS X you can use [boot2docker](http://boot2docker.io/). (see also: https://docs.docker.com/installation/mac/)
6. Get or build a [binary release](binary_release.md) of Kubernetes.

### Starting a Cluster

Change into the `kubernetes` directory in which you have the binary release, and then do

```bash
cluster/kube-up.sh
```

(If it fails, do `cluster/kube-down.sh` to clean up before trying again; otherwise, you'll get errors about resources that already exist.)

The script above relies on Google Storage to stage the Kubernetes release. It
then will start (by default) a single master VM along with 4 worker VMs.  You
can tweak some of these parameters by editing `cluster/gce/config-default.sh`
You can view a transcript of a successful cluster creation
[here](https://gist.github.com/satnam6502/fc689d1b46db9772adea).

The instances must be able to connect to each other using their private IP. The
script uses the "default" network which should have a firewall rule called
"default-allow-internal" which allows traffic on any port on the private IPs.
If this rule is missing from the default network or if you change the network
being used in `cluster/config-default.sh` create a new rule with the following
field values:

* Source Ranges: `10.0.0.0/8`
* Allowed Protocols and Port: `tcp:1-65535;udp:1-65535;icmp`

### Running a container (simple version)

Once you have your instances up and running, use cluster/kubectl.sh to access
the kubernetes api.

Note: if you built the release from source you will need to run `hack/build-go.sh` to
build the go components, which include the `kubectl` commandline client. If you are
using a prebuilt release, the built client binaries are already included.

The `kubectl.sh` line below spins up two containers running
[Nginx](http://nginx.org/en/) running on port 80:

```bash
cluster/kubectl.sh run-container my-nginx --image=dockerfile/nginx --replicas=2 --port=80
```

To stop the containers:

```bash
cluster/kubectl.sh stop rc my-nginx
```

To delete the containers:

```bash
cluster/kubectl.sh delete rc my-nginx
```

### Running a container (more complete version)

```bash
cd kubernetes
cluster/kubectl.sh create -f docs/getting-started-guides/pod.json
```

Where pod.json contains something like:

```json
{
  "id": "php",
  "kind": "Pod",
  "apiVersion": "v1beta1",
  "desiredState": {
    "manifest": {
      "version": "v1beta1",
      "id": "php",
      "containers": [{
        "name": "nginx",
        "image": "dockerfile/nginx",
        "ports": [{
          "containerPort": 80,
          "hostPort": 8081
        }],
        "livenessProbe": {
          "enabled": true,
          "type": "http",
          "initialDelaySeconds": 30,
          "httpGet": {
            "path": "/index.html",
            "port": 8081
          }
        }
      }]
    }
  },
  "labels": {
    "name": "foo"
  }
}
```

You can see your cluster's pods:

```bash
cluster/kubectl.sh get pods
```

and delete the pod you just created:

```bash
cluster/kubectl.sh delete pods php
```

Since this pod is scheduled on a minion running in GCE, you will have to enable incoming tcp traffic via the port specified in the
pod manifest before you see the nginx welcome page. After doing so, it should be visible at http://<external ip of minion running nginx>:<port from manifest>.

Look in `examples/` for more examples

### Tearing down the cluster
```bash
cd kubernetes
cluster/kube-down.sh
```
