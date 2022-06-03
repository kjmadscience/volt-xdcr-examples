# volt-xdcr-examples
Examples configs and steps for the different variations of Volt XDCR
### Google Cloud Platform

if confused about version compatibility please refer, [Volt-k8s-versions](https://docs.voltdb.com/ReleaseNotes/#rn_k8sCompat)
## volt version: 10.2.5, per-pod NodePort based XDCR configuration

1. Start the two different Kubernetes clusters
```
gcloud container clusters create  --machine-type "c2-standard-8" --image-type UBUNTU_CONTAINERD  --num-nodes 3    xdcr-1 --disk-type "pd-ssd" --disk-size "300" --zone "europe-west1-b"

gcloud container clusters create  --machine-type "c2-standard-8" --image-type UBUNTU_CONTAINERD  --num-nodes 3    xdcr-2 --disk-type "pd-ssd" --disk-size "300" --zone "europe-west1-b"

```
2. Changine context in kubectl
To use both clusters over kubectl use below command,
- list contexts available

`kubectl config get-contexts`

- switch to desired context

`kubectl config use-context <<REPLACE_NAME_OF_CONTEXT>>`

example, `kubectl config use-context gke_fourth-epigram-293718_europe-west1-b_xdcr-1`

3. Create docker secrets in both cluster to pull docker images

```
kubectl create secret docker-registry dockerio-registry --docker-username=jadejakajal13  --docker-password=b461d1b4-82c4-499e-afc0-f17943a16411  --docker-email=jadejakajal13@gmail.com
```

4. Edit the Helm install command below to point to correct path of license and values file.

```
helm install xdcr1 --version=1.3.5 --values cluster1-values.yaml --set operator.image.tag=1.3.5 --set-file cluster.config.licenseXMLFile="/Users/thanos/Documents/license.xml" voltdb/voltdb
```

- change context before deploying second Volt cluster in second kubernetes cluster

```
helm install xdcr2 --version=1.3.5 --values cluster2-values.yaml --set operator.image.tag=1.3.5 --set-file cluster.config.licenseXMLFile="/Users/thanos/Documents/license.xml" voltdb/voltdb
```

Wait for both the clusters to be ready, as shown below,
Pods:
```
bash-5.1$ kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
xdcr1-voltdb-cluster-0                   1/1     Running   0          5m45s
xdcr1-voltdb-cluster-1                   1/1     Running   0          5m45s
xdcr1-voltdb-cluster-2                   1/1     Running   0          5m45s
xdcr1-voltdb-operator-7bb8dd4756-pm7xc   1/1     Running   0          5m53s

bash-5.1$ kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
xdcr2-voltdb-cluster-0                   1/1     Running   0          2m3s
xdcr2-voltdb-cluster-1                   1/1     Running   0          2m3s
xdcr2-voltdb-cluster-2                   1/1     Running   0          2m3s
xdcr2-voltdb-operator-6f968484bc-2b2sb   1/1     Running   0          2m29s
```

Services:
```
bash-5.1$ kubectl get svc
NAME                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                   AGE
kubernetes                      ClusterIP   10.104.0.1      <none>        443/TCP                                                   9m59s
voltdb-operator-metrics         ClusterIP   10.104.14.104   <none>        8383/TCP,8686/TCP                                         12s
xdcr2-voltdb-cluster-0-dr       NodePort    10.104.9.252    <none>        5555:32760/TCP                                            11s
xdcr2-voltdb-cluster-1-dr       NodePort    10.104.10.187   <none>        5555:32761/TCP                                            11s
xdcr2-voltdb-cluster-2-dr       NodePort    10.104.8.254    <none>        5555:32762/TCP                                            11s
xdcr2-voltdb-cluster-client     NodePort    10.104.3.144    <none>        21211:31211/TCP,21212:31212/TCP                           11s
xdcr2-voltdb-cluster-http       NodePort    10.104.9.163    <none>        8080:31080/TCP                                            11s
xdcr2-voltdb-cluster-internal   ClusterIP   None            <none>        8080/TCP,3021/TCP,7181/TCP,9090/TCP,11235/TCP,11780/TCP   11s

bash-5.1$ kubectl get svc
NAME                            TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                                                   AGE
kubernetes                      ClusterIP   10.68.0.1     <none>        443/TCP                                                   19m
voltdb-operator-metrics         ClusterIP   10.68.6.119   <none>        8383/TCP,8686/TCP                                         6m40s
xdcr1-voltdb-cluster-0-dr       NodePort    10.68.14.72   <none>        5555:32760/TCP                                            6m39s
xdcr1-voltdb-cluster-1-dr       NodePort    10.68.5.6     <none>        5555:32761/TCP                                            6m39s
xdcr1-voltdb-cluster-2-dr       NodePort    10.68.10.59   <none>        5555:32762/TCP                                            6m39s
xdcr1-voltdb-cluster-client     NodePort    10.68.1.152   <none>        21211:31211/TCP,21212:31212/TCP                           6m39s
xdcr1-voltdb-cluster-http       NodePort    10.68.8.136   <none>        8080:31080/TCP                                            6m39s
xdcr1-voltdb-cluster-internal   ClusterIP   None          <none>        8080/TCP,3021/TCP,7181/TCP,9090/TCP,11235/TCP,11780/TCP   6m39s
```

5. Connecting the two clusters with each other

### Note: If you set serviceSpec.dr.externalTrafficPolicy=Local, you'd need to specify the correct port with the correct node because the node port will only direct traffic to pods on the local node.  Since each pod will advertise the ip address of it's node and it's specific service replicationNodePort, mesh traffic should end up on the right node/port.

Get the node ip addresses and service ports.
Get Node ip:

```
kubectl get nodes -o wide


bash-5.1$ kubectl get nodes -o wide
NAME                                    STATUS   ROLES    AGE   VERSION           INTERNAL-IP     EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
gke-xdcr-1-default-pool-30f277ae-4jrr   Ready    <none>   20m   v1.22.8-gke.201   10.132.15.217   34.140.137.187   Ubuntu 20.04.4 LTS   5.4.0-1065-gke   containerd://1.5.2
gke-xdcr-1-default-pool-30f277ae-57sf   Ready    <none>   20m   v1.22.8-gke.201   10.132.15.219   35.205.164.177   Ubuntu 20.04.4 LTS   5.4.0-1065-gke   containerd://1.5.2
gke-xdcr-1-default-pool-30f277ae-k99t   Ready    <none>   20m   v1.22.8-gke.201   10.132.15.218   104.199.106.83   Ubuntu 20.04.4 LTS   5.4.0-1065-gke   containerd://1.5.2
```

Check DR-service Ports:

```
bash-5.1$ kubectl get svc | grep -- -dr
xdcr1-voltdb-cluster-0-dr       NodePort    10.68.14.72   <none>        5555:32760/TCP                                            9m31s
xdcr1-voltdb-cluster-1-dr       NodePort    10.68.5.6     <none>        5555:32761/TCP                                            9m31s
xdcr1-voltdb-cluster-2-dr       NodePort    10.68.10.59   <none>        5555:32762/TCP
```

If you need to check which pod is on which node:

```
bash-5.1$ kubectl get pods -o wide
NAME                                     READY   STATUS    RESTARTS   AGE   IP           NODE                                    NOMINATED NODE   READINESS GATES
xdcr1-voltdb-cluster-0                   1/1     Running   0          13m   10.64.1.6    gke-xdcr-1-default-pool-30f277ae-k99t   <none>           <none>
xdcr1-voltdb-cluster-1                   1/1     Running   0          13m   10.64.0.10   gke-xdcr-1-default-pool-30f277ae-4jrr   <none>           <none>
xdcr1-voltdb-cluster-2                   1/1     Running   0          13m   10.64.2.7    gke-xdcr-1-default-pool-30f277ae-57sf   <none>           <none>
xdcr1-voltdb-operator-7bb8dd4756-pm7xc   1/1     Running   0          13m   10.64.0.9    gke-xdcr-1-default-pool-30f277ae-4jrr   <none>           <none>
```

Create the connection strings in below format,

Address of cluster 1:
Con1='104.199.106.83:32760\,34.140.137.187:32761\,35.205.164.177:32762'
Address of cluster 2:
Con2='34.79.7.159:32760\,35.195.167.190:32761\,35.205.164.177:32762'

6. Update these values in the running Volt clusters using Helm upgrade

```
helm --kube-context=gke_fourth-epigram-293718_europe-west1-b_xdcr-2 upgrade xdcr2 voltdb/voltdb --reuse-values --version=1.3.5 --set cluster.config.deployment.dr.connection.source="$Con1"

helm --kube-context=gke_fourth-epigram-293718_europe-west1-b_xdcr-1 upgrade xdcr1 voltdb/voltdb --reuse-values --version=1.3.5 --set cluster.config.deployment.dr.connection.source="$Con2"
```

7. Load Same Schema in Both Clusters
Cluster 1
```
kubectl cp voter-procs.jar xdcr2-voltdb-cluster-0:/tmp/
kubectl exec -it xdcr2-voltdb-cluster-0 -- sqlcmd < ddl.sql
```

Cluster 2
```
kubectl cp voter-procs.jar xdcr2-voltdb-cluster-0:/tmp/
kubectl exec -it xdcr2-voltdb-cluster-0 -- sqlcmd < ddl.sql
```

8. Verify XDCR Readiness

`kubectl exec -it xdcr1-voltdb-cluster-0 -- sqlcmd`

example result if all is running fine,
```
bash-5.1$ kubectl exec -it xdcr1-voltdb-cluster-0 -- sqlcmd
SQL Command :: localhost:21212
1> exec @Statistics XDCR_READINESS 0;
TIMESTAMP      HOST_ID  HOSTNAME                IS_READY  DRROLE_STATE  DRPROD_STATE  DRPROD_ISSYNCED  DRPROD_CNXSTS  DRCONS_STATE  DRCONS_ISCOVERED  DRCONS_ISPAUSED
-------------- -------- ----------------------- --------- ------------- ------------- ---------------- -------------- ------------- ----------------- ----------------
 1654241878905        0 xdcr1-voltdb-cluster-0  true      ACTIVE        ACTIVE        true             UP             RECEIVE       true              false
 1654241878905        1 xdcr1-voltdb-cluster-1  true      ACTIVE        ACTIVE        true             UP             RECEIVE       true              false
 1654241878905        2 xdcr1-voltdb-cluster-2  true      ACTIVE        ACTIVE        true             UP             RECEIVE       true              false
(Returned 3 rows in 0.01s)
```
