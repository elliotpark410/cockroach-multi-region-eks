# Cockroach-Multi-Region-EKS

Multi-Region deployment of CockroachDB in AWS EKS

# TO DO

- create preliminary guide from research
- get required AWS resources
- deploy
- add global load balancer
- add schema and seed data
- add dbworkload
- configure cluster for multi-region
- test regional survivability demo
- create Loom video
- create step-by-step guide with screenshots and diagrams

# Deploy TO DO

-

# Architecture

- 3 EKS clusters
- 3 regions (us-east-1, us-east-2, and us-west-2)
- 1 CockroachDB pod per region - each running a single CockroachDB node
- VPC peering between 3 regional VPCs so pod IPs can talk cross-region
- CoreDNS forwards + Network Load Balancer

# Prerequisites

AWS account with permissions for:

- EKS
- EC2
- VPC
- CloudFormation
- IAM
- Route Tables
- Network Load Balancer
- Security Groups

CLI Tooling:

- Kubernetes version 1.18+
- AWS CLI
- eksctl
- kubectl
- cockroach binary and on PATH (for cert generation)
- Pyhton 3
- dig (for looking up Network Load Balancer IPs)

# Create 3 EKS clusters

- you need non-overlapping VPC CIDRs (private IP address for VPC) for VPC peering to work

For example:

- us-east-1: 10.0.0.0/16
- us-east-2: 10.1.0.0/16
- us-west-2: 10.2.0.0/16

# Pick a a node type

- something that is 24 vCPU since we're only deploying 3 nodes (i.e. instances)

- use different node-type instead of m5.2xlarge

```
# us-east-1
eksctl create cluster \
  --name crdb-use1 \
  --nodegroup-name standard-workers \
  --node-type m5.2xlarge \
  --nodes 3 \
  --region us-east-1 \
  --vpc-cidr 10.0.0.0/16

# us-east-2
eksctl create cluster \
  --name crdb-use2 \
  --nodegroup-name standard-workers \
  --node-type m5.2xlarge \
  --nodes 3 \
  --region us-east-2 \
  --vpc-cidr 10.1.0.0/16

# us-west-2
eksctl create cluster \
  --name crdb-usw2 \
  --nodegroup-name standard-workers \
  --node-type m5.2xlarge \
  --nodes 3 \
  --region us-west-2 \
  --vpc-cidr 10.2.0.0/16
```

- Verify contexts

`kubectl config get-contexts`

- Output should be

```
youruser@crdb-use1.us-east-1.eksctl.io
youruser@crdb-use2.us-east-2.eksctl.io
youruser@crdb-usw2.us-west-2.eksctl.io
```

# Create namespaces per region

```
# us-east-1
kubectl create namespace us-east-1 \
  --context youruser@crdb-use1.us-east-1.eksctl.io

# us-east-2
kubectl create namespace us-east-2 \
  --context youruser@crdb-use2.us-east-2.eksctl.io

# us-west-2
kubectl create namespace us-west-2 \
  --context youruser@crdb-usw2.us-west-2.eksctl.io
```

# VPC Peering and Routing

- need full pod-to-pod connectivity across cluster

1. In VPC console, for each region, note the VPC ID created by EKS
2. Create three VPC peering connections
3. In each region, update VPC's route table (usually PublicRouteTable) with two routes:

- Destination: each other region's CIDR (10.1.0.0/16, 10.2.0.0/16, etc.)
- Target: the peering connection to that VPC

# Security Groups (ports 26257, 8080) - do this all 3 regions

- In each region's EC2 console, find the EKS worker ClusterSharedNodeSecurityGroup (or similarly named)
- Add inbound rules:
  - inter-region Cockroach traffic (26257) for each other VPC's CIDR
    - Type: Custom TCP
    - Port: 26257
    - Source: 10.0.0.0/16, 10.1.0.0/16, 10.2.0.0/16 (one rule per CIDR)
  - Client app connections (26257) - your app subnets
  - DB Console & Health (8080)
    - For DB Console: port 808, source = your admin IP/ CIDR
    - For Network Load Balancer Health Checks: port 8080, source = local VPC CIDR

# DNS load balancers for CoreDNS

- Each cluster gets a Network Load Balancer targeting its CoreDNS service, using the official `dns-lb-eks.yaml`

- you should see a new Network Load Balancer in each region's EC2 -> Load Balancers

Example dns-lb-eks.yaml

```
# us-east-1 cluster
kubectl apply -f \
  https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/eks/dns-lb-eks.yaml \
  --context youruser@crdb-use1.us-east-1.eksctl.io

# us-east-2 cluster
kubectl apply -f \
  https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/eks/dns-lb-eks.yaml \
  --context youruser@crdb-use2.us-east-2.eksctl.io

# us-west-2 cluster
kubectl apply -f \
  https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/eks/dns-lb-eks.yaml \
  --context youruser@crdb-usw2.us-west-2.eksctl.io
```

For each Network Load Balancer (NLB):

1. Copy its DNS name (e.g., abcd1234.elb.us-east-1.amazonaws.com)
2. Run `dig` to get its IP addresses:
   example
   `dig abcd1234.elb.us-east-1.amazonaws.com`
3. You'll see 3 IPs (one per AZ) - call them ip1, ip2, ip3 for that region

# Configure CoreDNS cross-region forwarding

- Download the ConfigMap template and create three region-specific variants

```
curl -O \
  https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/eks/configmap.yaml
```

- Edit three copies:

  - For us-east-1 config (configmap-use1.yaml)
    - Replace region2 / region3 with us-east-2 and us-west-2 (your Cockroach namespaces in those regions)
    - Replace `ip1`, `ip2`, and `ip3` under each `forward` block with th three IPs of the us-east-2 and us-west Neetwork Load Balancers

- Do analogous mapping for `us-east-2` and `us-west-2` configs (each config lists the other two regions and forward to their Network Load Balancer IPs)

- Apply each config to the corresponding cluster (after backing up the existing one)

```
# Backup existing
kubectl -n kube-system get configmap coredns -o yaml \
  > coredns-backup-use1.yaml \
  --context youruser@crdb-use1.us-east-1.eksctl.io

# Apply new config
kubectl apply -f configmap-use1.yaml \
  --context youruser@crdb-use1.us-east-1.eksctl.io

# Repeat backup/apply for us-east-2 and us-west-2 with their files/contexts
```

- Verify

```
kubectl get -n kube-system cm/coredns --export -o yaml \
  --context youruser@crdb-use1.us-east-1.eksctl.io
```

# Exclude VPC CIDRs from SNAT (AWS CNI)

- tell the AWS CNI plugin not to SNAT traffic between your three VPCs

- On each cluster:

```
kubectl set env ds aws-node -n kube-system \
  AWS_VPC_K8S_CNI_EXCLUDE_SNAT_CIDRS="10.0.0.0/16,10.1.0.0/16,10.2.0.0/16" \
  --context youruser@crdb-use1.us-east-1.eksctl.io

kubectl set env ds aws-node -n kube-system \
  AWS_VPC_K8S_CNI_EXCLUDE_SNAT_CIDRS="10.0.0.0/16,10.1.0.0/16,10.2.0.0/16" \
  --context youruser@crdb-use2.us-east-2.eksctl.io

kubectl set env ds aws-node -n kube-system \
  AWS_VPC_K8S_CNI_EXCLUDE_SNAT_CIDRS="10.0.0.0/16,10.1.0.0/16,10.2.0.0/16" \
  --context youruser@crdb-usw2.us-west-2.eksctl.io
```

# Generate TLS certs (single CA, all regions)

- Create directories and CA cert locally

```
mkdir certs my-safe-directory

cockroach cert create-ca \
  --certs-dir=certs \
  --ca-key=my-safe-directory/ca.key
```

Create root client cert:

```
cockroach cert create-client root \
  --certs-dir=certs \
  --ca-key=my-safe-directory/ca.key
```

Upload the root client cert bundle to each cluster / namespace as cockroachdb.client.root:

```
kubectl create secret generic cockroachdb.client.root \
  --from-file=certs \
  --context youruser@crdb-use1.us-east-1.eksctl.io \
  --namespace us-east-1

kubectl create secret generic cockroachdb.client.root \
  --from-file=certs \
  --context youruser@crdb-use2.us-east-2.eksctl.io \
  --namespace us-east-2

kubectl create secret generic cockroachdb.client.root \
  --from-file=certs \
  --context youruser@crdb-usw2.us-west-2.eksctl.io \
  --namespace us-west-2
```

- Now create node certs for each namespace (one region at a time). Example for us-east-1 namespace

```
cockroach cert create-node \
  localhost 127.0.0.1 \
  cockroachdb-public \
  cockroachdb-public.us-east-1 \
  cockroachdb-public.us-east-1.svc.cluster.local \
  *.cockroachdb \
  *.cockroachdb.us-east-1 \
  *.cockroachdb.us-east-1.svc.cluster.local \
  --certs-dir=certs \
  --ca-key=my-safe-directory/ca.key
```

- Upload as cockroachdb.node in the cluster / namespace

```
kubectl create secret generic cockroachdb.node \
  --from-file=certs \
  --context youruser@crdb-use1.us-east-1.eksctl.io \
  --namespace us-east-1
```

- Repeat for us-east-2 and us-west-2, regenerating a node cert each time (you can delete node.crt/node.key between runs) and uploading as cockroachdb.node in the respective namespace

- Ensure the same CA is used across all; that's crucial for a single logical cluster

# Prepare StatefulSets (1 pod per region, 8 vCPU / 32 GiB RAM [izing to be updated])

- Download the EKS multi-region StatefulSet template

```
curl -O \
  https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/eks/cockroachdb-statefulset-secure-eks.yaml
```

- Make 3 copies

```
cp cockroachdb-statefulset-secure-eks.yaml ss-use1.yaml
cp cockroachdb-statefulset-secure-eks.yaml ss-use2.yaml
cp cockroachdb-statefulset-secure-eks.yaml ss-usw2.yaml
```

For each file:

1. Set namespace (under `metadat` or via `kubetctl` command) to the correct region namespace

2. Set replicas to 1:

- This is aadapting from the generic StatefulSet which defaults to `replicas: 3`

```
spec:
  serviceName: "cockroachdb"
  replicas: 1      # was 3
```

3. Resource requests / limits from 8 vCPU / 32 GiB (might need to be updated)

- In the CockroachDB container spec: (.spec.template.spec.containers[0].resources), set e.g.

- The manifeset computes `--cache` and `--max-sql-memory` as `MEMORY_LIMIT_MIB / 4`, so each will be ~8 GiB when the limit is 32 GiB

```
resources:
  requests:
    cpu: "8"
    memory: "32Gi"
  limits:
    cpu: "8"
    memory: "32Gi"
```

4. Locality flag -tune --locality per region

- In each file's `command:` block, find `--locality` and set:

ss-use1.yaml:

```
--locality=region=us-east-1,az=$(cat /etc/cockroach-env/zone),dns=$(hostname -f)
```

ss-use2.yaml:

```
--locality=region=us-east-2,az=$(cat /etc/cockroach-env/zone),dns=$(hostname -f)
```

ss-usw2.yaml:

```
--locality=region=us-west-2,az=$(cat /etc/cockroach-env/zone),dns=$(hostname -f)
```

5. Join list - seed nodes across all three regions

Simplify to a single pod per region (since we have `replicas: 1`)

- Place this in place of the default join string

```
--join=cockroachdb-0.cockroachdb.us-east-1,\
cockroachdb-0.cockroachdb.us-east-2,\
cockroachdb-0.cockroachdb.us-west-2
```

6. PVC size – adjust `volumeClaimTemplates[].spec.resources.requests.storage` as needed (default is 100Gi in one of the templates)

```
volumeClaimTemplates:
- metadata:
    name: datadir
  spec:
    accessModes:
    - "ReadWriteOnce"
    resources:
      requests:
        storage: 500Gi   # example
```

# Deploy StatefulSets and initialize cluster

- Apply each StatefulSet in its region:

```
# us-east-1
kubectl create -f ss-use1.yaml \
  --context youruser@crdb-use1.us-east-1.eksctl.io \
  --namespace us-east-1

# us-east-2
kubectl create -f ss-use2.yaml \
  --context youruser@crdb-use2.us-east-2.eksctl.io \
  --namespace us-east-2

# us-west-2
kubectl create -f ss-usw2.yaml \
  --context youruser@crdb-usw2.us-west-2.eksctl.io \
  --namespace us-west-2
```

- Wait until each pod `1/1 Running`:

```
kubectl get pods --context youruser@crdb-use1.us-east-1.eksctl.io --namespace us-east-1
kubectl get pods --context youruser@crdb-use2.us-east-2.eksctl.io --namespace us-east-2
kubectl get pods --context youruser@crdb-usw2.us-west-2.eksctl.io --namespace us-west-2
```

- You should see:

`cockroachdb-0 1/1 Running` in each namespace

- Now initialize the cluster once, from any region (example: us-east-2):

```
kubectl exec \
  --context youruser@crdb-use2.us-east-2.eksctl.io \
  --namespace us-east-2 \
  -it cockroachdb-0 -- \
  /cockroach/cockroach init \
    --certs-dir=/cockroach/cockroach-certs
```

- You should see `Cluster successflly initialized` in the output

- Re-check pods in all regions to ensure they remain `1/1 Running`

# Connect via SQL client and verify

- Use the official `client-secure.yaml` to run a client pod in one region (e.g., us-east-2):

```
kubectl create -f \
  https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/client-secure.yaml \
  --context youruser@crdb-use2.us-east-2.eksctl.io \
  --namespace us-east-2
```

- Open a SQL shell

```
kubectl exec -it cockroachdb-client-secure \
  --context youruser@crdb-use2.us-east-2.eksctl.io \
  --namespace us-east-2 -- \
  ./cockroach sql \
    --certs-dir=/cockroach-certs \
    --host=cockroachdb-public
```

- Access the DB Console

```
kubectl port-forward cockroachdb-0 8080 \
  --context youruser@crdb-use2.us-east-2.eksctl.io \
  --namespace us-east-2
```

- Browse to `https://localhost:8080`, log in with `roach / strongpassword`, and verify:

- All 3 nodes present in the Node List.
- Latencies in the Network Latency page show the three regions.
