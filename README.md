# CrateDB-Kubernetes
This CrateDB/crate/crate.io `StatefulSet` Deployment for Kubernetes 1.7 will get your [CrateDB](https://crate.io/) cluster up and running.

Discory of nodes is done via DNS. Please be advised to alter the template according to your needs, for example copy-pasting this will fail since you will likely need to modify the `storageClassName` of the provisioned volumes or you're opting to deploy into a different namespace than `default`. You may also want to create an additional service to expose the web interface.

I created this repo since the [official CrateDB Kubernetes template](https://crate.io/docs/crate/guide/getting_started/scale/kubernetes.html#the-cratedb-template) does not work and is still using the deprecated `PetSet` deployments (as of August 2017).

## K8s
```
apiVersion: v1
kind: Service
metadata:
  name: crate
spec:
  clusterIP: None
  ports:
    - port: 4200
      name: web
    - port: 4300
      name: transport
  selector:
    app: crate
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: crate
spec:
  replicas: 3
  podManagementPolicy: Parallel
  updateStrategy:
    type: RollingUpdate
  serviceName: crate
  template:
    metadata:
      name: crate
      labels:
        app: crate
    spec:
      terminationGracePeriodSeconds: 25
      containers:
        - name: crate
          image: crate:2.1.2
          args:
            - -Ccluster.name=crate
            - -Cdiscovery.zen.minimum_master_nodes=3
            - -Cgateway.expected_nodes=3
            - -Cgateway.recover_after_nodes=3
            - -Cnetwork.bind_host=0.0.0.0
            - -Cnetwork.publish_host=_site_
            - -Cdiscovery.type=zen
            - -Cdiscovery.zen.hosts_provider=srv
            - -Cdiscovery.srv.query=_transport._tcp.crate.default.svc.cluster.local
            - -Clicense.enterprise=false
          env:
            - name: CRATE_HEAP_SIZE
              value: 3g
          resources:
            requests:
              memory: 6G
              cpu: 2
            limits:
              memory: 10G
              cpu: 3
          volumeMounts:
            - mountPath: /data
              name: data
          ports:
            - containerPort: 4200
              name: web
            - containerPort: 4300
              name: cluster
      initContainers:
        - name: sysctl
          image: busybox
          imagePullPolicy: IfNotPresent
          command: ["sysctl", "-w", "vm.max_map_count=262144"]
          securityContext:
            privileged: true
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: ssd-normal
        resources:
          requests:
            storage: 8Gi
---
```
