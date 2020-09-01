# Create the AWS Cloud Formation Stack

Only needed once per account:

```
clusterawsadm alpha bootstrap create-stack
```

Example output:

```
$ clusterawsadm alpha bootstrap create-stack
Attempting to create CloudFormation stack cluster-api-provider-aws-sigs-k8s-io

Following resources are in the stack: 

Resource                  |Type                                                                                |Status
AWS::IAM::Group           |bootstrapper.cluster-api-provider-aws.sigs.k8s.io                                   |CREATE_COMPLETE
AWS::IAM::InstanceProfile |control-plane.cluster-api-provider-aws.sigs.k8s.io                                  |CREATE_COMPLETE
AWS::IAM::InstanceProfile |controllers.cluster-api-provider-aws.sigs.k8s.io                                    |CREATE_COMPLETE
AWS::IAM::InstanceProfile |nodes.cluster-api-provider-aws.sigs.k8s.io                                          |CREATE_COMPLETE
AWS::IAM::ManagedPolicy   |arn:aws:iam::076481223434:policy/control-plane.cluster-api-provider-aws.sigs.k8s.io |CREATE_COMPLETE
AWS::IAM::ManagedPolicy   |arn:aws:iam::076481223434:policy/nodes.cluster-api-provider-aws.sigs.k8s.io         |CREATE_COMPLETE
AWS::IAM::ManagedPolicy   |arn:aws:iam::076481223434:policy/controllers.cluster-api-provider-aws.sigs.k8s.io   |CREATE_COMPLETE
AWS::IAM::Role            |control-plane.cluster-api-provider-aws.sigs.k8s.io                                  |CREATE_COMPLETE
AWS::IAM::Role            |controllers.cluster-api-provider-aws.sigs.k8s.io                                    |CREATE_COMPLETE
AWS::IAM::Role            |nodes.cluster-api-provider-aws.sigs.k8s.io                                          |CREATE_COMPLETE
AWS::IAM::User            |bootstrapper.cluster-api-provider-aws.sigs.k8s.io                                   |CREATE_COMPLETE
$
```

Init the management cluster via UI:

```
$ tkg init --ui
Logs of the command execution can also be found at: /tmp/tkg-20200422T103601705557834.log

Validating the pre-requisites...
Serving kickstart UI at http://127.0.0.1:8080
Validating configuration...
web socket connection established
sending pending 2 logs to UI
Using infrastructure provider aws:v0.5.2
Generating cluster configuration...
Setting up bootstrapper...
Installing providers on bootstrapper...
Fetching providers
Installing cert-manager
Waiting for cert-manager to be available...
Installing Provider="cluster-api" Version="v0.3.3" TargetNamespace="capi-system"
Installing Provider="bootstrap-kubeadm" Version="v0.3.3" TargetNamespace="capi-kubeadm-bootstrap-system"
Installing Provider="control-plane-kubeadm" Version="v0.3.3" TargetNamespace="capi-kubeadm-control-plane-system"
Installing Provider="infrastructure-aws" Version="v0.5.2" TargetNamespace="capa-system"
Start creating management cluster...
Installing providers on management cluster...
Fetching providers
Installing cert-manager
Waiting for cert-manager to be available...
Installing Provider="cluster-api" Version="v0.3.3" TargetNamespace="capi-system"
Installing Provider="bootstrap-kubeadm" Version="v0.3.3" TargetNamespace="capi-kubeadm-bootstrap-system"
Installing Provider="control-plane-kubeadm" Version="v0.3.3" TargetNamespace="capi-kubeadm-control-plane-system"
Installing Provider="infrastructure-aws" Version="v0.5.2" TargetNamespace="capa-system"
Waiting for the management cluster to get ready for move...
Moving all Cluster API objects from bootstrap cluster to management cluster...
Performing move...
Discovering Cluster API objects
Moving Cluster API objects Clusters=1
Creating objects in the target cluster
Deleting objects from the source cluster
Context set for management cluster tkg-aws-dashaun-cloud as 'tkg-aws-dashaun-cloud-admin@tkg-aws-dashaun-cloud'.

Management cluster created!


You can now create your first workload cluster by running the following:

  tkg create cluster [name] --kubernetes-version=[version] --plan=[plan]
$
```

Done!

Test things out:

```
$ tkg get management-cluster
+-------------------------+---------------------------------------------------+
| MANAGEMENT CLUSTER NAME | CONTEXT NAME                                      |
+-------------------------+---------------------------------------------------+
| tkg-aws-dashaun-cloud * | tkg-aws-dashaun-cloud-admin@tkg-aws-dashaun-cloud |
+-------------------------+---------------------------------------------------+
$
```

Now lets deploy a cluster!

Dry Run First:

```
$ tkg create cluster my-cluster --plan=dev --dry-run
apiVersion: cluster.x-k8s.io/v1alpha3
kind: Cluster
metadata:
  name: my-cluster
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks:
      - 100.96.0.0/11
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1alpha3
    kind: KubeadmControlPlane
    name: my-cluster-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
    kind: AWSCluster
    name: my-cluster
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
kind: AWSCluster
metadata:
  name: my-cluster
  namespace: default
spec:
  bastion:
    enabled: true
  networkSpec:
    subnets:
    - availabilityZone: us-east-1a
      cidrBlock: 10.0.1.0/24
      isPublic: true
    - availabilityZone: us-east-1a
      cidrBlock: 10.0.0.0/24
    vpc:
      cidrBlock: 10.0.0.0/16
  region: us-east-1
  sshKeyName: default
---
apiVersion: controlplane.cluster.x-k8s.io/v1alpha3
kind: KubeadmControlPlane
metadata:
  name: my-cluster-control-plane
  namespace: default
spec:
  infrastructureTemplate:
    apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
    kind: AWSMachineTemplate
    name: my-cluster-control-plane
  kubeadmConfigSpec:
    clusterConfiguration:
      apiServer:
        extraArgs:
          cloud-provider: aws
          tls-cipher-suites: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_RSA_WITH_AES_256_GCM_SHA384
        timeoutForControlPlane: 8m0s
      controllerManager:
        extraArgs:
          cloud-provider: aws
          tls-cipher-suites: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
      dns:
        imageRepository: registry.tkg.vmware.run
        imageTag: v1.6.5_vmware.4
        type: CoreDNS
      etcd:
        local:
          dataDir: /var/lib/etcd
          extraArgs:
            cipher-suites: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
          imageRepository: registry.tkg.vmware.run
          imageTag: v3.4.3_vmware.4
      imageRepository: registry.tkg.vmware.run
      kubernetesVersion: v1.17.3+vmware.2
      scheduler:
        extraArgs:
          tls-cipher-suites: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
    initConfiguration:
      nodeRegistration:
        kubeletExtraArgs:
          cloud-provider: aws
          tls-cipher-suites: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
        name: '{{ ds.meta_data.local_hostname }}'
    joinConfiguration:
      nodeRegistration:
        kubeletExtraArgs:
          cloud-provider: aws
          tls-cipher-suites: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
        name: '{{ ds.meta_data.local_hostname }}'
    useExperimentalRetryJoin: true
  replicas: 1
  version: v1.17.3+vmware.2
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
kind: AWSMachineTemplate
metadata:
  name: my-cluster-control-plane
  namespace: default
spec:
  template:
    spec:
      ami:
        id: ami-0cdd7837e1fdd81f8
      iamInstanceProfile: control-plane.cluster-api-provider-aws.sigs.k8s.io
      instanceType: t3.medium
      rootVolume:
        size: 80
      sshKeyName: default
---
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
kind: AWSMachineTemplate
metadata:
  name: my-cluster-md-0
  namespace: default
spec:
  template:
    spec:
      ami:
        id: ami-0cdd7837e1fdd81f8
      iamInstanceProfile: nodes.cluster-api-provider-aws.sigs.k8s.io
      instanceType: t3.medium
      rootVolume:
        size: 80
      sshKeyName: default
---
apiVersion: bootstrap.cluster.x-k8s.io/v1alpha3
kind: KubeadmConfigTemplate
metadata:
  name: my-cluster-md-0
  namespace: default
spec:
  template:
    spec:
      joinConfiguration:
        nodeRegistration:
          kubeletExtraArgs:
            cloud-provider: aws
            tls-cipher-suites: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
          name: '{{ ds.meta_data.local_hostname }}'
      useExperimentalRetryJoin: true
---
apiVersion: cluster.x-k8s.io/v1alpha3
kind: MachineDeployment
metadata:
  name: my-cluster-md-0
  namespace: default
spec:
  clusterName: my-cluster
  replicas: 1
  selector:
    matchLabels: null
  template:
    spec:
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1alpha3
          kind: KubeadmConfigTemplate
          name: my-cluster-md-0
      clusterName: my-cluster
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
        kind: AWSMachineTemplate
        name: my-cluster-md-0
      version: v1.17.3+vmware.2
---
apiVersion: v1
kind: Secret
metadata:
  name: my-cluster-postcreate
  namespace: default
stringData:
  calicoYaml: |
    ---
    # Source: calico/templates/calico-config.yaml
    # This ConfigMap is used to configure a self-hosted Calico installation.
    kind: ConfigMap
    apiVersion: v1
    metadata:
      name: calico-config
      namespace: kube-system
    data:
      # Typha is disabled.
      typha_service_name: "none"
      # Configure the backend to use.
      calico_backend: "bird"

      # Configure the MTU to use
      veth_mtu: "1440"

      # The CNI network configuration to install on each node.  The special
      # values in this config will be automatically populated.
      cni_network_config: |-
        {
          "name": "k8s-pod-network",
          "cniVersion": "0.3.1",
          "plugins": [
            {
              "type": "calico",
              "log_level": "info",
              "datastore_type": "kubernetes",
              "nodename": "__KUBERNETES_NODE_NAME__",
              "mtu": __CNI_MTU__,
              "ipam": {
                  "type": "calico-ipam"
              },
              "policy": {
                  "type": "k8s"
              },
              "kubernetes": {
                  "kubeconfig": "__KUBECONFIG_FILEPATH__"
              }
            },
            {
              "type": "portmap",
              "snat": true,
              "capabilities": {"portMappings": true}
            }
          ]
        }

    ---
    # Source: calico/templates/kdd-crds.yaml
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: felixconfigurations.crd.projectcalico.org
    spec:
      scope: Cluster
      group: crd.projectcalico.org
      version: v1
      names:
        kind: FelixConfiguration
        plural: felixconfigurations
        singular: felixconfiguration
    ---

    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: ipamblocks.crd.projectcalico.org
    spec:
      scope: Cluster
      group: crd.projectcalico.org
      version: v1
      names:
        kind: IPAMBlock
        plural: ipamblocks
        singular: ipamblock

    ---

    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: blockaffinities.crd.projectcalico.org
    spec:
      scope: Cluster
      group: crd.projectcalico.org
      version: v1
      names:
        kind: BlockAffinity
        plural: blockaffinities
        singular: blockaffinity

    ---

    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: ipamhandles.crd.projectcalico.org
    spec:
      scope: Cluster
      group: crd.projectcalico.org
      version: v1
      names:
        kind: IPAMHandle
        plural: ipamhandles
        singular: ipamhandle

    ---

    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: ipamconfigs.crd.projectcalico.org
    spec:
      scope: Cluster
      group: crd.projectcalico.org
      version: v1
      names:
        kind: IPAMConfig
        plural: ipamconfigs
        singular: ipamconfig

    ---

    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: bgppeers.crd.projectcalico.org
    spec:
      scope: Cluster
      group: crd.projectcalico.org
      version: v1
      names:
        kind: BGPPeer
        plural: bgppeers
        singular: bgppeer

    ---

    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: bgpconfigurations.crd.projectcalico.org
    spec:
      scope: Cluster
      group: crd.projectcalico.org
      version: v1
      names:
        kind: BGPConfiguration
        plural: bgpconfigurations
        singular: bgpconfiguration

    ---

    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: ippools.crd.projectcalico.org
    spec:
      scope: Cluster
      group: crd.projectcalico.org
      version: v1
      names:
        kind: IPPool
        plural: ippools
        singular: ippool

    ---

    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: hostendpoints.crd.projectcalico.org
    spec:
      scope: Cluster
      group: crd.projectcalico.org
      version: v1
      names:
        kind: HostEndpoint
        plural: hostendpoints
        singular: hostendpoint

    ---

    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: clusterinformations.crd.projectcalico.org
    spec:
      scope: Cluster
      group: crd.projectcalico.org
      version: v1
      names:
        kind: ClusterInformation
        plural: clusterinformations
        singular: clusterinformation

    ---

    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: globalnetworkpolicies.crd.projectcalico.org
    spec:
      scope: Cluster
      group: crd.projectcalico.org
      version: v1
      names:
        kind: GlobalNetworkPolicy
        plural: globalnetworkpolicies
        singular: globalnetworkpolicy

    ---

    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: globalnetworksets.crd.projectcalico.org
    spec:
      scope: Cluster
      group: crd.projectcalico.org
      version: v1
      names:
        kind: GlobalNetworkSet
        plural: globalnetworksets
        singular: globalnetworkset

    ---

    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: networkpolicies.crd.projectcalico.org
    spec:
      scope: Namespaced
      group: crd.projectcalico.org
      version: v1
      names:
        kind: NetworkPolicy
        plural: networkpolicies
        singular: networkpolicy

    ---

    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: networksets.crd.projectcalico.org
    spec:
      scope: Namespaced
      group: crd.projectcalico.org
      version: v1
      names:
        kind: NetworkSet
        plural: networksets
        singular: networkset
    ---
    # Source: calico/templates/rbac.yaml

    # Include a clusterrole for the kube-controllers component,
    # and bind it to the calico-kube-controllers serviceaccount.
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: calico-kube-controllers
    rules:
      # Nodes are watched to monitor for deletions.
      - apiGroups: [""]
        resources:
          - nodes
        verbs:
          - watch
          - list
          - get
      # Pods are queried to check for existence.
      - apiGroups: [""]
        resources:
          - pods
        verbs:
          - get
      # IPAM resources are manipulated when nodes are deleted.
      - apiGroups: ["crd.projectcalico.org"]
        resources:
          - ippools
        verbs:
          - list
      - apiGroups: ["crd.projectcalico.org"]
        resources:
          - blockaffinities
          - ipamblocks
          - ipamhandles
        verbs:
          - get
          - list
          - create
          - update
          - delete
      # Needs access to update clusterinformations.
      - apiGroups: ["crd.projectcalico.org"]
        resources:
          - clusterinformations
        verbs:
          - get
          - create
          - update
    ---
    kind: ClusterRoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: calico-kube-controllers
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: calico-kube-controllers
    subjects:
    - kind: ServiceAccount
      name: calico-kube-controllers
      namespace: kube-system
    ---
    # Include a clusterrole for the calico-node DaemonSet,
    # and bind it to the calico-node serviceaccount.
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      name: calico-node
    rules:
      # The CNI plugin needs to get pods, nodes, and namespaces.
      - apiGroups: [""]
        resources:
          - pods
          - nodes
          - namespaces
        verbs:
          - get
      - apiGroups: [""]
        resources:
          - endpoints
          - services
        verbs:
          # Used to discover service IPs for advertisement.
          - watch
          - list
          # Used to discover Typhas.
          - get
      - apiGroups: [""]
        resources:
          - nodes/status
        verbs:
          # Needed for clearing NodeNetworkUnavailable flag.
          - patch
          # Calico stores some configuration information in node annotations.
          - update
      # Watch for changes to Kubernetes NetworkPolicies.
      - apiGroups: ["networking.k8s.io"]
        resources:
          - networkpolicies
        verbs:
          - watch
          - list
      # Used by Calico for policy information.
      - apiGroups: [""]
        resources:
          - pods
          - namespaces
          - serviceaccounts
        verbs:
          - list
          - watch
      # The CNI plugin patches pods/status.
      - apiGroups: [""]
        resources:
          - pods/status
        verbs:
          - patch
      # Calico monitors various CRDs for config.
      - apiGroups: ["crd.projectcalico.org"]
        resources:
          - globalfelixconfigs
          - felixconfigurations
          - bgppeers
          - globalbgpconfigs
          - bgpconfigurations
          - ippools
          - ipamblocks
          - globalnetworkpolicies
          - globalnetworksets
          - networkpolicies
          - networksets
          - clusterinformations
          - hostendpoints
          - blockaffinities
        verbs:
          - get
          - list
          - watch
      # Calico must create and update some CRDs on startup.
      - apiGroups: ["crd.projectcalico.org"]
        resources:
          - ippools
          - felixconfigurations
          - clusterinformations
        verbs:
          - create
          - update
      # Calico stores some configuration information on the node.
      - apiGroups: [""]
        resources:
          - nodes
        verbs:
          - get
          - list
          - watch
      # These permissions are only requried for upgrade from v2.6, and can
      # be removed after upgrade or on fresh installations.
      - apiGroups: ["crd.projectcalico.org"]
        resources:
          - bgpconfigurations
          - bgppeers
        verbs:
          - create
          - update
      # These permissions are required for Calico CNI to perform IPAM allocations.
      - apiGroups: ["crd.projectcalico.org"]
        resources:
          - blockaffinities
          - ipamblocks
          - ipamhandles
        verbs:
          - get
          - list
          - create
          - update
          - delete
      - apiGroups: ["crd.projectcalico.org"]
        resources:
          - ipamconfigs
        verbs:
          - get
      # Block affinities must also be watchable by confd for route aggregation.
      - apiGroups: ["crd.projectcalico.org"]
        resources:
          - blockaffinities
        verbs:
          - watch
      # The Calico IPAM migration needs to get daemonsets. These permissions can be
      # removed if not upgrading from an installation using host-local IPAM.
      - apiGroups: ["apps"]
        resources:
          - daemonsets
        verbs:
          - get
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: calico-node
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: calico-node
    subjects:
    - kind: ServiceAccount
      name: calico-node
      namespace: kube-system

    ---
    # Source: calico/templates/calico-node.yaml
    # This manifest installs the calico-node container, as well
    # as the CNI plugins and network config on
    # each master and worker node in a Kubernetes cluster.
    kind: DaemonSet
    apiVersion: apps/v1
    metadata:
      name: calico-node
      namespace: kube-system
      labels:
        k8s-app: calico-node
    spec:
      selector:
        matchLabels:
          k8s-app: calico-node
      updateStrategy:
        type: RollingUpdate
        rollingUpdate:
          maxUnavailable: 1
      template:
        metadata:
          labels:
            k8s-app: calico-node
          annotations:
            # This, along with the CriticalAddonsOnly toleration below,
            # marks the pod as a critical add-on, ensuring it gets
            # priority scheduling and that its resources are reserved
            # if it ever gets evicted.
            scheduler.alpha.kubernetes.io/critical-pod: ''
        spec:
          nodeSelector:
            beta.kubernetes.io/os: linux
          hostNetwork: true
          tolerations:
            # Make sure calico-node gets scheduled on all nodes.
            - effect: NoSchedule
              operator: Exists
            # Mark the pod as a critical add-on for rescheduling.
            - key: CriticalAddonsOnly
              operator: Exists
            - effect: NoExecute
              operator: Exists
          serviceAccountName: calico-node
          # Minimize downtime during a rolling upgrade or deletion; tell Kubernetes to do a "force
          # deletion": https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods.
          terminationGracePeriodSeconds: 0
          priorityClassName: system-node-critical
          initContainers:
            # This container performs upgrade from host-local IPAM to calico-ipam.
            # It can be deleted if this is a fresh installation, or if you have already
            # upgraded to use calico-ipam.
            - name: upgrade-ipam
              image: registry.tkg.vmware.run/calico-all/cni-plugin:v3.11.2_vmware.1
              command: ["/opt/cni/bin/calico-ipam", "-upgrade"]
              env:
                - name: KUBERNETES_NODE_NAME
                  valueFrom:
                    fieldRef:
                      fieldPath: spec.nodeName
                - name: CALICO_NETWORKING_BACKEND
                  valueFrom:
                    configMapKeyRef:
                      name: calico-config
                      key: calico_backend
              volumeMounts:
                - mountPath: /var/lib/cni/networks
                  name: host-local-net-dir
                - mountPath: /host/opt/cni/bin
                  name: cni-bin-dir
              securityContext:
                privileged: true
            # This container installs the CNI binaries
            # and CNI network config file on each node.
            - name: install-cni
              image: registry.tkg.vmware.run/calico-all/cni-plugin:v3.11.2_vmware.1
              command: ["/install-cni.sh"]
              env:
                # Name of the CNI config file to create.
                - name: CNI_CONF_NAME
                  value: "10-calico.conflist"
                # The CNI network config to install on each node.
                - name: CNI_NETWORK_CONFIG
                  valueFrom:
                    configMapKeyRef:
                      name: calico-config
                      key: cni_network_config
                # Set the hostname based on the k8s node name.
                - name: KUBERNETES_NODE_NAME
                  valueFrom:
                    fieldRef:
                      fieldPath: spec.nodeName
                # CNI MTU Config variable
                - name: CNI_MTU
                  valueFrom:
                    configMapKeyRef:
                      name: calico-config
                      key: veth_mtu
                # Prevents the container from sleeping forever.
                - name: SLEEP
                  value: "false"
              volumeMounts:
                - mountPath: /host/opt/cni/bin
                  name: cni-bin-dir
                - mountPath: /host/etc/cni/net.d
                  name: cni-net-dir
              securityContext:
                privileged: true
            # Adds a Flex Volume Driver that creates a per-pod Unix Domain Socket to allow Dikastes
            # to communicate with Felix over the Policy Sync API.
            - name: flexvol-driver
              image: registry.tkg.vmware.run/calico-all/pod2daemon:v3.11.2_vmware.1
              volumeMounts:
              - name: flexvol-driver-host
                mountPath: /host/driver
              securityContext:
                privileged: true
          containers:
            # Runs calico-node container on each Kubernetes node.  This
            # container programs network policy and routes on each
            # host.
            - name: calico-node
              image: registry.tkg.vmware.run/calico-all/node:v3.11.2_vmware.1
              env:
                # Use Kubernetes API as the backing datastore.
                - name: DATASTORE_TYPE
                  value: "kubernetes"
                # Wait for the datastore.
                - name: WAIT_FOR_DATASTORE
                  value: "true"
                # Set based on the k8s node name.
                - name: NODENAME
                  valueFrom:
                    fieldRef:
                      fieldPath: spec.nodeName
                # Choose the backend to use.
                - name: CALICO_NETWORKING_BACKEND
                  valueFrom:
                    configMapKeyRef:
                      name: calico-config
                      key: calico_backend
                # Cluster type to identify the deployment type
                - name: CLUSTER_TYPE
                  value: "k8s,bgp"
                # Auto-detect the BGP IP address.
                - name: IP
                  value: "autodetect"
                # Enable IPIP
                - name: CALICO_IPV4POOL_IPIP
                  value: "Always"
                # Set MTU for tunnel device used if ipip is enabled
                - name: FELIX_IPINIPMTU
                  valueFrom:
                    configMapKeyRef:
                      name: calico-config
                      key: veth_mtu
                # The default IPv4 pool to create on startup if none exists. Pod IPs will be
                # chosen from this range. Changing this value after installation will have
                # no effect. This should fall within `--cluster-cidr`.
                - name: CALICO_IPV4POOL_CIDR
                  value: "100.96.0.0/11"
                # Disable file logging so `kubectl logs` works.
                - name: CALICO_DISABLE_FILE_LOGGING
                  value: "true"
                # Set Felix endpoint to host default action to ACCEPT.
                - name: FELIX_DEFAULTENDPOINTTOHOSTACTION
                  value: "ACCEPT"
                # Disable IPv6 on Kubernetes.
                - name: FELIX_IPV6SUPPORT
                  value: "false"
                # Set Felix logging to "info"
                - name: FELIX_LOGSEVERITYSCREEN
                  value: "info"
                - name: FELIX_HEALTHENABLED
                  value: "true"
              securityContext:
                privileged: true
              resources:
                requests:
                  cpu: 250m
              livenessProbe:
                exec:
                  command:
                  - /bin/calico-node
                  - -felix-live
                  - -bird-live
                periodSeconds: 10
                initialDelaySeconds: 10
                failureThreshold: 6
              readinessProbe:
                exec:
                  command:
                  - /bin/calico-node
                  - -felix-ready
                  - -bird-ready
                periodSeconds: 10
              volumeMounts:
                - mountPath: /lib/modules
                  name: lib-modules
                  readOnly: true
                - mountPath: /run/xtables.lock
                  name: xtables-lock
                  readOnly: false
                - mountPath: /var/run/calico
                  name: var-run-calico
                  readOnly: false
                - mountPath: /var/lib/calico
                  name: var-lib-calico
                  readOnly: false
                - name: policysync
                  mountPath: /var/run/nodeagent
          volumes:
            # Used by calico-node.
            - name: lib-modules
              hostPath:
                path: /lib/modules
            - name: var-run-calico
              hostPath:
                path: /var/run/calico
            - name: var-lib-calico
              hostPath:
                path: /var/lib/calico
            - name: xtables-lock
              hostPath:
                path: /run/xtables.lock
                type: FileOrCreate
            # Used to install CNI.
            - name: cni-bin-dir
              hostPath:
                path: /opt/cni/bin
            - name: cni-net-dir
              hostPath:
                path: /etc/cni/net.d
            # Mount in the directory for host-local IPAM allocations. This is
            # used when upgrading from host-local to calico-ipam, and can be removed
            # if not using the upgrade-ipam init container.
            - name: host-local-net-dir
              hostPath:
                path: /var/lib/cni/networks
            # Used to create per-pod Unix Domain Sockets
            - name: policysync
              hostPath:
                type: DirectoryOrCreate
                path: /var/run/nodeagent
            # Used to install Flex Volume Driver
            - name: flexvol-driver-host
              hostPath:
                type: DirectoryOrCreate
                path: /usr/libexec/kubernetes/kubelet-plugins/volume/exec/nodeagent~uds
    ---

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: calico-node
      namespace: kube-system

    ---
    # Source: calico/templates/calico-kube-controllers.yaml

    # See https://github.com/projectcalico/kube-controllers
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: calico-kube-controllers
      namespace: kube-system
      labels:
        k8s-app: calico-kube-controllers
    spec:
      # The controllers can only have a single active instance.
      replicas: 1
      selector:
        matchLabels:
          k8s-app: calico-kube-controllers
      strategy:
        type: Recreate
      template:
        metadata:
          name: calico-kube-controllers
          namespace: kube-system
          labels:
            k8s-app: calico-kube-controllers
          annotations:
            scheduler.alpha.kubernetes.io/critical-pod: ''
        spec:
          nodeSelector:
            beta.kubernetes.io/os: linux
          tolerations:
            # Mark the pod as a critical add-on for rescheduling.
            - key: CriticalAddonsOnly
              operator: Exists
            - key: node-role.kubernetes.io/master
              effect: NoSchedule
          serviceAccountName: calico-kube-controllers
          priorityClassName: system-cluster-critical
          containers:
            - name: calico-kube-controllers
              image: registry.tkg.vmware.run/calico-all/kube-controllers:v3.11.2_vmware.1
              env:
                # Choose which controllers to run.
                - name: ENABLED_CONTROLLERS
                  value: node
                - name: DATASTORE_TYPE
                  value: kubernetes
              readinessProbe:
                exec:
                  command:
                  - /usr/bin/check-status
                  - -r

    ---

    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: calico-kube-controllers
      namespace: kube-system
    ---
    # Source: calico/templates/calico-etcd-secrets.yaml

    ---
    # Source: calico/templates/calico-typha.yaml

    ---
    # Source: calico/templates/configure-canal.yaml

```

Let's save that output:

```
$ tkg create cluster my-cluster --plan=dev --dry-run > my-cluster-config.yaml
$
```

Ok, thats cool!

```
$ tkg create cluster my-cluster --plan=dev
Logs of the command execution can also be found at: /tmp/tkg-20200422T172454862591371.log
Creating workload cluster 'my-cluster'...


Context set for workload cluster my-cluster as my-cluster-admin@my-cluster

Waiting for cluster nodes to be available...

Workload cluster 'my-cluster' created

$
```

Lets get those kubectl credentials:

```
$ tkg get credentials my-cluster
Credentials of workload cluster my-cluster have been saved 
You can now access the cluster by switching the context to my-cluster-admin@my-cluster under /home/dashaun/.kube/config 
$
```

Switch Context:

```
$ kubectl config use-context my-cluster-admin@my-cluster
Switched to context "my-cluster-admin@my-cluster".
$
```

All set:

```
$ kubectl get nodes
NAME                         STATUS   ROLES    AGE     VERSION
ip-10-0-0-135.ec2.internal   Ready    master   9m33s   v1.17.3+vmware.2
ip-10-0-0-68.ec2.internal    Ready    <none>   7m36s   v1.17.3+vmware.2
$
```

Of course add wavefront integration:

```
$ helm repo add wavefront https://wavefronthq.github.io/helm/
"wavefront" has been added to your repositories                 
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "wavefront" chart repository
Update Complete. ⎈ Happy Helming!⎈ 
$ kubectl create namespace wavefront
namespace/wavefront created
$ helm install wavefront wavefront/wavefront --set wavefront.url=https://demo.wavefront.com --set wavefront.token="$(echo $WAVEFRONT_TOKEN)" --set clusterName=my-cluster --namespace wavefront
NAME: wavefront
LAST DEPLOYED: Fri Apr 24 13:41:50 2020
NAMESPACE: wavefront
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Wavefront is setup and configured to collect metrics from your Kubernetes cluster.  You
should see metrics flowing within a few minutes.

You can visit this dashboard in Wavefront to see your Kubernetes metrics:

https://surf.wavefront.com/dashboard/integration-kubernetes-summary
```


Delete the cluster we created:

```
$ tkg delete cluster my-cluster
Deleting workload cluster 'my-cluster'. Are you sure?: y█
workload cluster my-cluster is being deleted 
$
```

Delete the management cluster:

```
$ tkg delete management-cluster

```
