# nix-infra-test
This is a micro cluster setup for testing nix-infra. It is intended to allow you to try out nix-infra with minimal configuration. All you need is a Hetzner account and some super basic configuration.

1. Download [nix-infra](https://github.com/jhsware/nix-infra/releases) and install it

2. Run [this script](https://github.com/jhsware/nix-infra-test/blob/main/scripts/get-test.sh) in the terminal to download test scripts:

```sh
sh <(curl -L https://raw.githubusercontent.com/jhsware/nix-infra-test/refs/heads/main/scripts/get-test.sh)
```
3. Get an API-key for an empty Hetzner Cloud project

4. Edit the .env in the created folder

5. Run the test script

```sh
nix-infra-test/test-nix-infra-with-apps.sh --env=nix-infra-test/.env
```

Once you have set up .env properly, the downloaded script will provision, configure and deploy your cluster. It will then run some tests to check that it is working properly and finish by tearing down the cluster. Copy and modify the script to create your own experimental cluster.

## Test Script Options

To build without immediately tearing down the cluster:

```sh
test-nix-infra-with-apps.sh --no-teardown --env=nix-infra-test/.env
```

Useful commands to explore the running test cluster (check the bash script for more):

```sh
test-nix-infra-with-apps.sh etcd "/cluster" --env=nix-infra-test/.env
test-nix-infra-with-apps.sh cmd --target=ingress001 "uptime" --env=nix-infra-test/.env
test-nix-infra-with-apps.sh ssh ingress001 --env=nix-infra-test/.env
```

To tear down the cluster:

```sh
test-nix-infra-with-apps.sh teardown --env=nix-infra-test/.env
```

## Node Types

```mermaid
flowchart

control_node[Control Node]
cluster_node[Cluster Node]
ingress_node[Ingress Node]
```

The control node(s) make up the control plane and handles dynamic cluster state management such as:

- overlay network
- service mesh

This allows us to limit interaction to the specific nodes being changed during deployment. The cluster will automatically propagate changes to the other affected nodes in the cluster.

The worker nodes run databases and applications.

The ingress node(s) exposes the cluster to the internet.

## Cluster Topology

```mermaid
flowchart

subgraph ctrl-plane
direction LR
  etcd001 <--> etcd002 <--> etcd003
end

ctrl-plane <-- TLS --> overlay-network

subgraph overlay-network
direction TB
  registry001
  service001
  worker001
  ingress001
end
```

Orchestration and configuration of the cluster nodes is done over SSH directly to each target node. This allows parallell execution.

The overlay network is a Flanneld mesh network over a Wireguard encrypted network interface.

Services and applications are exposed over a service mesh through local haproxy loadbalancer. This can provide a fault tolerant setup when you deploy multiple instances of a service or app.

```mermaid
stateDiagram-v2

ctrl_node --> cluster_node : overlay network conf<br>service mesh conf
CLI --> ctrl_node : provision
CLI --> cluster_node : provision
CLI --> ctrl_node : deploy
CLI --> cluster_node : deploy

state ctrl_node {
  etcd
  systemd_ctrl
  state  "systemd" as systemd_ctrl
}

state cluster_node {
  haproxy
  confd
  systemd
  flanneld
}

cluster_node : cluster_node / ingress_node
```

## Deploying an Application
Each node in the cluster has it's own configuration in the `nodes/` folder.

In this configuration you can configure what apps to run on that node and how you want them to be configured.

The actual deployment is done using the `deploy-apps` command and specifying the target nodes you want to update. All app configurations or the node will be affected.

### Publish an OCI-Image
Build you local image, get the image tag and run:
```sh
$ scripts/publish [image_name] [image_tag] registry001
```

The image will be uploaded to the private registry and is then available for deployment in your cluster.

### Secrets
To securely provide secrets to your application, store them using the CLI `secrets` command or as an output from a CLI `action`command using the option `--store-as-secret=[name]`.

The secret will be encrypted in your local cluster configuration directory. When deploying an application, the CLI will pass any required secrets to the target and store it as a systemd credential. Systemd credentials are automatically encrypted/decrypted on demand.
