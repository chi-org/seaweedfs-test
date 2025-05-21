## SeaweedFS



1. Install SeaweedFS (for filer metadata store, need mariadb installed with `sw_database` database and `filemeta` table and a user with suitable privileges)

   ```bash
   helm repo add seaweedfs https://seaweedfs.github.io/seaweedfs/helm
   
   helm show values seaweedfs/seaweedfs > seaweed_values.yaml
   
   nano seaweed_values.yaml
   
   helm upgrade --install seaweed seaweedfs/seaweedfs   --namespace seaweedfs   --create-namespace -f seaweed_values.yaml
   ```

   ```yaml
   # seaweed_values.yaml
   global:
     enableReplication: true
     # 1 main + 1 replica, 4 nodes => if >2 nodes down, cannot write
     # Should fix.replication when 1 node down to prevent losing data when 2 nodes down
     replicationPlacement: "001" 
   filer:
     enabled: true
     replicas: 4
     extraEnvironmentVars:
       WEED_MYSQL_ENABLED: "true"
       WEED_MYSQL_HOSTNAME: "mariadb.nmaa.svc.cluster.local"
       WEED_MYSQL_PORT: "3306"
       WEED_MYSQL_DATABASE: "sw_database"
       WEED_MYSQL_CONNECTION_MAX_IDLE: "5"
       WEED_MYSQL_CONNECTION_MAX_OPEN: "75"
       # "refresh" connection every 10 minutes, eliminating mysql closing "old" connections
       WEED_MYSQL_CONNECTION_MAX_LIFETIME_SECONDS: "600"
       # enable usage of memsql as filer backend
       WEED_MYSQL_INTERPOLATEPARAMS: "true"
       # if you want to use leveldb2, then should enable "enablePVC". or you may lose your data.
       WEED_LEVELDB2_ENABLED: "false"
       # with http DELETE, by default the filer would check whether a folder is empty.
       # recursive_delete will delete all sub folders and files, similar to "rm -Rf"
       WEED_FILER_OPTIONS_RECURSIVE_DELETE: "false"
       # directories under this folder will be automatically creating a separate bucket
       WEED_FILER_BUCKETS_FOLDER: "/buckets"
     secretExtraEnvironmentVars:
       WEED_MYSQL_USERNAME:
         secretKeyRef:
           name: mariadb-secret
           key: mysql_user
       WEED_MYSQL_PASSWORD:
         secretKeyRef:
           name: mariadb-secret
           key: mysql_password
   master:
     replicas: 3 # for Raft Algorithm
     config: |-
       [master.maintenance]
       script = """
         lock
         volume.balance -force
         volume.fix.replication -force
         unlock
       """
       sleep_minutes = 30
     # BACKUP PROCESS TO LOCAL
     # sidecars:
     # - name: backup-filer-process
     #   image: chrislusf/seaweedfs:3.85
     #   command: 
     #   - sh
     #   - -c
     #	  - |
     #	    echo "[INFO] Generating replication.toml"
     #	  	cat <<EOF > replication.toml
     #     [sink.local]
     #     enabled = true
     #     directory = "/data/backup"
     #     is_incremental = false
     #     EOF
     #	    echo "[INFO] Starting filer backup process"
     #	    weed filer.backup -filer $WEED_CLUSTER_SW_FILER -doDeleteFiles
     #   env:
     #   - name: WEED_CLUSTER_SW_FILER
     #	 	value: seaweedfs-filer-client.seaweedfs:8888
     #	  volumeMounts:
     #   - name: data-seaweedfs
     #     mountPath: /data
   volume:
     replicas: 4
     dataDirs:
     - name: data1
       type: "hostPath"
       hostPathPrefix: /opt/shared
       maxVolumes: 0
   ```

   

2. Expose Filer Node Port:

   ```bash
   k expose pod seaweedfs-filer-0 --type NodePort --name seaweedfs-filer-np -n seaweedf
   ```

   

3. Fix replication and balance cronjob

   ```yaml
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: auto-fix-rep-and-balance
     namespace: seaweedfs
   spec:
     schedule: "*/30 * * * *"
     successfulJobsHistoryLimit: 1
     failedJobsHistoryLimit: 1
     jobTemplate:
       spec:
         template:
           spec:
             containers:
             - name: fix-replication-and-balance
               image: chrislusf/seaweedfs:3.85
               command:
                 - sh
                 - -c
                 - |
                   echo "[INFO] Volume List Before Fixing and Balancing"
                   echo "volume.list" | weed shell -master $WEED_CLUSTER_SW_MASTER
                   echo "[INFO] Start Fixing Replication..."
                   echo "lock; volume.fix.replication -force -doDelete false; unlock" | weed shell -master $WEED_CLUSTER_SW_MASTER
                   echo "[INFO] Start Balancing Volume..."
                   echo "lock; volume.balance -force; volume.list unlock" | weed shell -master $WEED_CLUSTER_SW_MASTER;
                   echo "[INFO] Volume List After Fixing and Balancing"                 	   echo "volume.list" | weed shell -master $WEED_CLUSTER_SW_MASTER
                   echo "[INFO] Done"
               env:
                 - name: WEED_CLUSTER_SW_MASTER
                   value: "seaweedfs-master.seaweedfs:9333"
             restartPolicy: Never
   ```

   

4. Install SeaweedFS CSI

   ```bash
   helm repo add seaweedfs-csi-driver https://seaweedfs.github.io/seaweedfs-csi-driver/helm
   
   helm show values seaweedfs-csi-driver/seaweedfs-csi-driver > seaweedfs_csi_values.yaml
   
   nano seaweedfs_csi_values.yaml
   
   helm upgrade --install seaweedfs-csi seaweedfs-csi-driver/seaweedfs-csi-driver   --namespace seaweedfs   --create-namespace -f seaweedfs_csi_values.yaml
   ```

   ```yaml
   # seaweedfs_csi_values.yaml
   seaweedfsFiler: seaweedfs-filer.seaweedfs.svc.cluster.local:8888
   controller:
     replicas: 4
   ```

   

5. Test CSI
   ```yaml
   # pvc.yml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: seaweedfs-pvc
     namespace: seaweedfs
   spec:
     accessModes:
       - ReadWriteMany
     resources:
       requests:
         storage: 5Gi
     storageClassName: seaweedfs-storage
   ```

   ```yaml
   # test_deploy_4_rep.yml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: ha-app
     namespace: seaweedfs
   spec:
     replicas: 4
     selector:
       matchLabels:
         app: ha-app
     template:
       metadata:
         labels:
           app: ha-app
       spec:
         containers:
           - name: test
             image: ubuntu:22.04
             command: ["sh", "-c", "sleep 36000"]
             volumeMounts:
               - name: shared-volume
                 mountPath: /data
         volumes:
           - name: shared-volume
             persistentVolumeClaim:
               claimName: seaweedfs-pvc
   ```

   ```bash
   k apply -f test_deploy_4_rep.yml,pvc.yml
   ```




5. API Support for File Operations

   https://github.com/seaweedfs/seaweedfs/wiki/Master-Server-API

   https://github.com/seaweedfs/seaweedfs/wiki/Volume-Server-API

   https://github.com/seaweedfs/seaweedfs/wiki/Filer-Server-API



6. Backup and Recovery

- Backup folders and files under filer dir (including meta data) 
  https://github.com/seaweedfs/seaweedfs/wiki/Async-Backup

  ```bash
  # filer server
  weed scaffold -config=replication -output=.
  weed filer.backup
  ```

  ```toml
  # replication.toml
  [sink.local]
  enabled = true
  directory = "/data"                                                  
  # all replicated files are under modified time as yyyy-mm-dd director# so each date directory contains all new and updated files.         
  is_incremental = false
  ```

- Recovery

  ```bash
  weed filer.copy ./path/to/folder_or_file http://filer_server:8888/
  ```

  
