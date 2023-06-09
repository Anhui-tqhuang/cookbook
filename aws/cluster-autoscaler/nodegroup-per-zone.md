# one nodegroup per zone

Firstly, prepare one EKS cluster config:

```yaml
# clusterconfig.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: "aws-tests"
  region: "us-west-2"
  version: "1.23"

iam:
  withOIDC: true

availabilityZones:
- us-west-2a
- us-west-2b
- us-west-2c

managedNodeGroups:
- name: "aws-tests-default"
  instanceType: m5.xlarge
  desiredCapacity: 3
  labels:
    tests: default
  iam:
    withAddonPolicies:
      awsLoadBalancerController: true
- name: "aws-tests-biganimal"
  instanceType: c5.large
  desiredCapacity: 0
  availabilityZones:
  - us-west-2a
  - us-west-2b
  - us-west-2c
  minSize: 0
  maxSize: 10
  labels:
    tests: biganimal
  taints:
  - key: biganimal
    value: "true"
    effect: NoSchedule
  iam:
    withAddonPolicies:
      awsLoadBalancerController: true
      autoScaler: true
  tags:
    k8s.io/cluster-autoscaler/node-template/label/tests: biganimal
    k8s.io/cluster-autoscaler/node-template/taint/biganimal: "true:NoSchedule"
  propagateASGTags: true

addons:
- name: vpc-cni
  attachPolicyARNs:
  - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
- name: coredns
  version: latest
- name: kube-proxy
  version: latest
- name: aws-ebs-csi-driver
  version: latest

cloudWatch:
  clusterLogging:
    enableTypes: ["all"]
```

Create the eks cluster

```
eksctl create cluster -f clusterconfig.yaml
```

Install cluster-autoscaler on EKS with the following doc: https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html

On this EKS cluster, we want to deploy one zone-redundant application, for example:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  # we want to have three replicas on three zones
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      nodeSelector:
        # we want to create pods on nodepools with this label
        tests: biganimal
      affinity:
        # we could use pod anti-affinity to make sure one pod consumes one
        # node, and three replicas live in three zones
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: Exists
            topologyKey: kubernetes.io/hostname
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx
            topologyKey: topology.kubernetes.io/zone
      tolerations:
      - key: biganimal
        operator: Equal
        value: "true"
        effect: NoSchedule
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

Then we could see the application triggers the scale up successfully

```
  Normal   TriggeredScaleUp  7s    cluster-autoscaler  pod triggered scale-up: [{eks-aws-tests-biganimal-c6c3db93-f3ce-46af-6376-e0ca25d44be4 0->1 (max: 10)}]
```

Once one pod gets scheduled successfully, the second one could not trigger new nodes anymore

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-64fb4654b7-lhv5s   1/1     Running   0          6m47s
nginx-deployment-64fb4654b7-tcbp6   0/1     Pending   0          6m47s
nginx-deployment-64fb4654b7-zwlmb   0/1     Pending   0          6m47s
```

```
  Normal   NotTriggerScaleUp  20s   cluster-autoscaler  pod didn't trigger scale-up: 1 node(s) didn't match pod anti-affinity rules, 1 max node group size reached
```

Scale down the replica of the deployment to `0`:

```
kubectl scale --replicas=0 deployment nginx-deployment
```

Delete the nodeGroup `aws-tests-biganimal`:

```
eksctl delete nodegroup --cluster aws-tests --name aws-tests-biganimal
```

Create three nodeGroups around three zones through the following config:

```yaml
# nodegroups.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: aws-tests
  region: us-west-2

managedNodeGroups:
- name: "aws-tests-biganimal-1"
  instanceType: c5.large
  desiredCapacity: 0
  availabilityZones:
  - us-west-2a
  minSize: 0
  maxSize: 10
  labels:
    tests: biganimal
  taints:
  - key: biganimal
    value: "true"
    effect: NoSchedule
  iam:
    withAddonPolicies:
      awsLoadBalancerController: true
      autoScaler: true
  tags:
    k8s.io/cluster-autoscaler/node-template/label/tests: biganimal
    k8s.io/cluster-autoscaler/node-template/taint/biganimal: "true:NoSchedule"
  propagateASGTags: true
- name: "aws-tests-biganimal-2"
  instanceType: c5.large
  desiredCapacity: 0
  availabilityZones:
  - us-west-2b
  minSize: 0
  maxSize: 10
  labels:
    tests: biganimal
  taints:
  - key: biganimal
    value: "true"
    effect: NoSchedule
  iam:
    withAddonPolicies:
      awsLoadBalancerController: true
      autoScaler: true
  tags:
    k8s.io/cluster-autoscaler/node-template/label/tests: biganimal
    k8s.io/cluster-autoscaler/node-template/taint/biganimal: "true:NoSchedule"
  propagateASGTags: true
- name: "aws-tests-biganimal-3"
  instanceType: c5.large
  desiredCapacity: 0
  availabilityZones:
  - us-west-2c
  minSize: 0
  maxSize: 10
  labels:
    tests: biganimal
  taints:
  - key: biganimal
    value: "true"
    effect: NoSchedule
  iam:
    withAddonPolicies:
      awsLoadBalancerController: true
      autoScaler: true
  tags:
    k8s.io/cluster-autoscaler/node-template/label/tests: biganimal
    k8s.io/cluster-autoscaler/node-template/taint/biganimal: "true:NoSchedule"
  propagateASGTags: true
```

```
eksctl create nodegroup --config-file=nodegroups.yaml
```

Scale up the replica of the deployment to `3`:

```
kubectl scale --replicas=3 deployment nginx-deployment
```

And all pods get scheduled successfully:

```
kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP               NODE                                           NOMINATED NODE   READINESS GATES
nginx-deployment-64fb4654b7-2xzsp   1/1     Running   0          3m58s   192.168.20.178   ip-192-168-10-33.us-west-2.compute.internal    <none>           <none>
nginx-deployment-64fb4654b7-5drvp   1/1     Running   0          3m58s   192.168.72.242   ip-192-168-84-21.us-west-2.compute.internal    <none>           <none>
nginx-deployment-64fb4654b7-dzknc   1/1     Running   0          3m58s   192.168.59.216   ip-192-168-40-240.us-west-2.compute.internal   <none>           <none>
```

```
kubectl get nodes -l eks.amazonaws.com/nodegroup=aws-tests-biganimal-1
NAME                                          STATUS   ROLES    AGE     VERSION
ip-192-168-10-33.us-west-2.compute.internal   Ready    <none>   2m16s   v1.23.17-eks-a59e1f0

kubectl get nodes -l eks.amazonaws.com/nodegroup=aws-tests-biganimal-2
NAME                                           STATUS   ROLES    AGE     VERSION
ip-192-168-40-240.us-west-2.compute.internal   Ready    <none>   2m19s   v1.23.17-eks-a59e1f0

kubectl get nodes -l eks.amazonaws.com/nodegroup=aws-tests-biganimal-3
NAME                                          STATUS   ROLES    AGE     VERSION
ip-192-168-84-21.us-west-2.compute.internal   Ready    <none>   2m23s   v1.23.17-eks-a59e1f0
```

Question:

- If i set the `minSize` of the zone-redundant nodeGroup to `3`, if each node gets created in different zones, these pods could get scheduled successfully, but could aws confirm that each node will be created in each zone? Moreover if i have two applications, could i set the `minSize` to `6` directly and they are distributed equally to each zone like the following topology:

    - us-west-2a: 2 nodes on the nodeGroup aws-tests-biganimal
    - us-west-2b: 2 nodes on the nodeGroup aws-tests-biganimal
    - us-west-2c: 2 nodes on the nodeGroup aws-tests-biganimal
