*24/10/2024*

---
### RabbitMQ

![RabbitMQ](https://i.imgur.com/tO3rcSC.png "RabbitMQ")

RabbitMQ is an open-source message broker software that enables communication between different components of an application. It acts as an intermediary between producers and consumers of messages, allowing for asynchronous messaging and decoupling of the sender and receiver.

### Installation of RabbitMQ on Openshift Cluster

**RabbitMQ version:** 3.13.7
**Erlang version:** 26.2.5.4

There are several ways to install a RabbitMQ cluster on OpenShift, including manual deployments of StatefulSets, Helm charts or using operators. Here, deploying a RabbitMQ cluster using the RabbitMQ Cluster Operator provided by VMware Tanzu, available in the OpenShift OperatorHub from the community source catalogue.

1. Install the RabbitMQ cluster operator from OpenShift OperatorHub.

	Operators -> OperatorHub -> RabbitMQ-cluster-operator

![RabbitMQ Operator](https://i.imgur.com/vRXAWtg.png "RabbitMQ Operator")

During the installation of the operator, select the appropriate channel and version, and specify the namespace where the operator should be installed by choosing the desired namespace.
It may take a few minutes for the operator to be installed.

![RabbitMQ Operator Installed](https://i.imgur.com/YxHJOnT.png "RabbitMQ Operator Installed")

![RabbitMQ Operator Installed Page](https://i.imgur.com/jr35Azf.png "RabbitMQ Operator Installed Page")

2. Create a RabbitMQ cluster instance.

	Operators -> Installed Operators -> RabbitMQ-cluster-operator -> RabbitmqCluster

![RabbitMQ Cluster Instance](https://i.imgur.com/6J0Erwz.png "RabbitMQ Cluster Instance")

Now create a RabbitmqCluster instance to deploy the RabbitMQ cluster by configuring a YAML.

- **RabbitMQ cluster definition**

To create a RabbitMQ instance, a RabbitmqCluster resource definition must be created and applied. RabbitMQ Cluster Kubernetes Operator creates the necessary resources, such as Services and StatefulSet, in the same namespace in which the RabbitmqCluster was defined.

```yaml
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rabbitmq
  labels:
    app: rabbitmq
  namespace: rabbits
```

- **Number of Replicas**

Specify the number of replicas for the RabbitmqCluster. An even number of replicas is highly discouraged. Odd numbers (1, 3, 5, 7, and so on) must be used according to the quorum queue rules.

| Cluster Node Count | Tolerated Number of Node Failures | Tolerant to a Network Partition               |
|--------------------|-----------------------------------|-----------------------------------------------|
| 1                  | 0                                 | Not applicable                               |
| 2                  | 0                                 | No                                           |
| 3                  | 1                                 | Yes                                          |
| 4                  | 1                                 | Yes, if a majority exists on one side        |
| 5                  | 2                                 | Yes                                          |
| 6                  | 2                                 | Yes, if a majority exists on one side        |
| 7                  | 3                                 | Yes                                          |
| 8                  | 3                                 | Yes, if a majority exists on one side        |
| 9                  | 4                                 | Yes                                          |
```yaml
spec:
  replicas: 3
```

- **Image**

Specify the RabbitMQ image reference if the version to be installed is different then the default version as per the operator which is the latest version.

```yaml
  image: vinzyzk/rabbitmq:management
  imagePullPolicy: Always
```

- **Service Type**

Specify the Kubernetes Service type for the RabbitmqCluster Service.

```yaml
  service:
    type: LoadBalancer
```
*Note: RabbitMQ Cluster Kubernetes Operator currently does not support the ExternalName Service Type.*

- **Persistence**

Specify the persistence settings for the RabbitmqCluster Service.

```yaml
  persistence:
    storageClassName: standard-csi
    storage: 50Gi
```

- **Resource Requirements**

Specify the resource requests and limits of the RabbitmqCluster Pods.

```yaml
  resources:
    requests:
      cpu: 4000m
      memory: 8Gi
    limits:
      cpu: 4500m
      memory: 12Gi
```

- **Affinity and Anti-affinity Rules**

To schedule only on specific worker nodes if part of the requirement.

```yaml
            affinity:
              nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                    - matchExpressions:
                        - key: commit
                          operator: In
                          values:
                            - rabbitmq
```

- **RabbitMQ Additional Configuration**

Additional RabbitMQ configuration options that will be written to `/etc/rabbitmq/conf.d/90-userDefinedConfiguration.conf`.

The RabbitMQ Cluster Kubernetes Operator generates a configuration file `/etc/rabbitmq/conf.d/10-operatorDefaults.conf` with the following properties:

```conf
queue_master_locator = min-masters
disk_free_limit.absolute = 2GB
cluster_partition_handling = pause_minority
cluster_formation.peer_discovery_backend = rabbit_peer_discovery_k8s
cluster_formation.k8s.host = kubernetes.default
cluster_formation.k8s.address_type = hostname
cluster_formation.target_cluster_size_hint = ${number-of-replicas}
cluster_name = ${instance-name}
```
*Note: All the values in additional config will be applied after this list. If any property is specified twice, the latest will take effect.*

```yaml
  rabbitmq:
    additionalConfig: |
      log.file.level = debug
      log.file.rotation.date = $D0
      log.file.rotation.count = 5
      log.file.rotation.compress = true
```

- **RabbitMQ Additional Plugins**

Additional plugins to enable in RabbitMQ. RabbitMQ Cluster Kubernetes Operator enabled `rabbitmq_peer_discovery_k8s`, `rabbitmq_prometheus` and `rabbitmq_management` by default.

```yaml
    additionalPlugins:
      - rabbitmq_top
      - rabbitmq_shovel
      - rabbitmq_shovel_management
```

- **Support for Arbitrary User IDs**

By default, the RabbitMQ Cluster Operator deploys RabbitmqCluster Pods with fixed, non-root UIDs. To deploy on Openshift, it is necessary to override the Security Context for these Pods. This must be done for every RabbitmqCluster deployed under the `override` field:

```yaml
  override:
    statefulSet:
      spec:
        template:
          spec:
            containers: []
            securityContext: {}
```

3. Apply the following configuration and create the RabbitMQ cluster instance.

```yaml
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rabbitmq
  namespace: rabbits
  labels:
    app: rabbitmq
spec:
  replicas: 3
  persistence:
    storageClassName: standard-csi
    storage: 50Gi
  resources:
    requests:
      cpu: 4000m
      memory: 8Gi
    limits:
      cpu: 4500m
      memory: 12Gi
  rabbitmq:
    additionalConfig: |
      log.file.level = debug
      log.file.rotation.date = $D0
      log.file.rotation.count = 5
      log.file.rotation.compress = true
    additionalPlugins:
      - rabbitmq_top
      - rabbitmq_shovel
      - rabbitmq_shovel_management
  terminationGracePeriodSeconds: 120
  override:
    statefulSet:
      spec:
        template:
          spec:
            affinity:
              nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  nodeSelectorTerms:
                    - matchExpressions:
                        - key: commit
                          operator: In
                          values:
                            - rabbitmq
              podAntiAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                  - labelSelector:
                      matchLabels:
                        app: rabbitmq
                    topologyKey: "kubernetes.io/hostname"
            initContainers: []
            containers: []
            securityContext: {}
```

![RabbitMQ Operator Instance YAML](https://i.imgur.com/R1Wi9J3.png "RabbitMQ Operator Instance YAML")

4. Wait until the pods are running and ready.

![RabbitMQ Cluster Pods](https://i.imgur.com/8NZUPjC.png "RabbitMQ Cluster Pods")

5. Create a Route to the RabbitMQ management UI from the RabbitMQ service.
	Networking -> Routes -> Create Route

![RabbitMQ Cluster Route](https://i.imgur.com/fw9VuSA.png "RabbitMQ Cluster Route")

6. Get the default username and password from Secrets.
	Workload -> Secrets -> rabbitmq-default-user

![RabbitMQ Cluster Secret](https://i.imgur.com/d0plr60.png "RabbitMQ Cluster Secret")

7. Now with the acquired credentials access the RabbitMQ management UI from the browser.

![RabbitMQ Cluster UI](https://i.imgur.com/Sd5f9LP.png "RabbitMQ Cluster UI")

Here, a 3-node RabbitMQCluster is up and running on OpenShift.