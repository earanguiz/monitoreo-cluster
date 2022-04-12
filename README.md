# prometheus stack monitoring

Guía de instalación para stack de monitoreo prometheus grafana con uso de almacenamiento dinamico entregado por nfs (filestore)

# archivos



## desplegar helm chart para nfs storage dinamico 

nfs-subdir-external-provisiones 

El aprovisionador externo de subdirectorios NFS es un aprovisionador automático que utiliza su servidor NFS existente y ya configurado para soportar el aprovisionamiento dinámico de volúmenes persistentes de Kubernetes a través de reclamaciones de volúmenes persistentes. 

```
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
 --set nfs.server=10.136.221.202 \
 --set nfs.path=/development
```

Esto desplega un pod que controla el acceso  a la unidad nfs mediante una storageClass llamada  "nfs-client".
Dato importante es que el instalador de nfs-subdir-external-provisioner de helm tiene activado el flag de   archiveOnDelete: "true" lo que significa que al eliminar el pod la data se mantendra persistente en la unidad nfs
```
nfs-client           cluster.local/nfs-subdir-external-provisioner   Delete          Immediate              true                   104m
```
Para hacer uso del aprovisionamiento dinamico mediante el llamado al storageClass
a modo de ejemplo un manifiesto del tipo PersistentVolumeClaim llamando al storageClass nfs-client

````
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: volumen-example-pvc
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Mi
````

Posterior a la creacion del pvc,   ya está en condiciones de ser montado en un pod o deployment.

## Prometheus stack 
prometheus stack por defecto levanta volumen de tipo emptyDir
para poder mantener la informacion persistente dentro del stack dentro del archivo values.yaml es necesario modificar ambos apartados 
````

    ## Prometheus StorageSpec for persistent data
    ## ref: https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/user-guides/storage.md
   
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: [ "ReadWriteOnce" ]
          resources:
            requests:
              storage: 100Mi
          storageClassName: nfs-client


	##Using default values from https://github.com/grafana/helm-charts/blob/main/charts/grafana/values.yaml
	
grafana:
  deploymentStrategy:
    type: Recreate
  persistence:
    enabled: true
    type: pvc
    storageClassName: nfs-client
    accessModes:
    - ReadWriteOnce
    size: 100Mi
    finalizers:
    - kubernetes.io/pvc-protection
````

````
helm install --values values.yaml prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
````

````
➜  monitoreo k get pod -n monitoring
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-stack-kube-prom-alertmanager-0   2/2     Running   0          3m7s
prometheus-prometheus-stack-kube-prom-prometheus-0       2/2     Running   0          3m7s
prometheus-stack-grafana-5bfcd4fcd8-wqt4z                3/3     Running   0          40s
prometheus-stack-kube-prom-operator-5df6bf76-cmpn7       1/1     Running   0          3m10s
prometheus-stack-kube-state-metrics-6bd588fd5-9n9pf      1/1     Running   0          3m10s
prometheus-stack-prometheus-node-exporter-5b2v2          1/1     Running   0          3m10s
prometheus-stack-prometheus-node-exporter-9pmh2          1/1     Running   0          3m10s
prometheus-stack-prometheus-node-exporter-btgrh          1/1     Running   0          3m10s
prometheus-stack-prometheus-node-exporter-ljpmg          1/1     Running   0          3m10s
prometheus-stack-prometheus-node-exporter-q45c6          1/1     Running   0          3m10s
prometheus-stack-prometheus-node-exporter-s49qb          1/1     Running   0          3m10s
prometheus-stack-prometheus-node-exporter-sbbfl          1/1     Running   0          3m10s
prometheus-stack-prometheus-node-exporter-sz89l          1/1     Running   0          3m10s
prometheus-stack-prometheus-node-exporter-wmzvf          1/1     Running   0          3m10s
prometheus-stack-prometheus-node-exporter-zm6c2          1/1     Running   0          3m10s
````

````
💀 ➜ k get pvc -n monitoring
NAME                                                                                                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
prometheus-prometheus-stack-kube-prom-prometheus-db-prometheus-prometheus-stack-kube-prom-prometheus-0   Bound    pvc-482009c7-effb-412a-9f75-bba378f6e031   100Mi      RWO            nfs-client     65s
prometheus-stack-grafana                                                                                 Bound    pvc-4e0d30f8-409f-4b6f-ba20-0619472636fa   100Mi      RWO            nfs-client     71s
````

## 
