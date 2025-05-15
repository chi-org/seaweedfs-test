## SeaweedFS



1. Install SeaweedFS

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
   	# 1 main + 1 replica, 1/3 node down, new data will be written in 2 nodes left
     replicationPlacement: "001" 
   filer:
     enabled: true
     replicas: 3
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
     replicas: 3
   volume:
     replicas: 3
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
     replicas: 3
   ```
   
   

4. Test CSI

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
# test_deploy_3_rep.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-app
spec:
  replicas: 3
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
k apply -f pod.yml,pvc.yml
```



#### Objectives:

1. High Availability (HA)

   - Data is always distributed to 3 nodes

   - Each one always has 1 replica (need min 2/3 nodes available)

   - If there is only 1 node avalable (not enough for replication), seaweedfs will restrict write operator (read-only data) https://github.com/seaweedfs/seaweedfs/discussions/2312

   - Volume is only used for storing data, the metadata need to be kept in a metadata store (default is local leveldb2 - not good for HA)

   - `volume.balance`: 

   - `volume.fix.replication`

     

2. I/O Performance

   - Current status: 1 deployment 3 replicas, 1 pvc

   - Exec into 1 pod:

     ```bash
     apt update && apt install -y fio
     
     # sequential write
     fio --name=write-test --filename=/mnt/test/testfile --size=1G --bs=1M --rw=write --ioengine=libaio --direct=1
     
     # random read
     fio --name=read-test --filename=/mnt/test/testfile --size=1G --bs=4k --rw=randread --ioengine=libaio --direct=1
     
     # random write
     fio --name=randwrite-test --filename=/mnt/test/testfile --size=1G --bs=4k --rw=randwrite --ioengine=libaio --direct=1
     
     ```


   ![image-20250506105503903](./seaweedfs-notes.assets/image-20250506105503903.png)

   ![image-20250506105637356](./seaweedfs-notes.assets/image-20250506105637356.png)

   ![image-20250506105755394](./seaweedfs-notes.assets/image-20250506105755394.png)

   

3. Kubernetes Integration via CSI

   

4. API Support for File Operations

   https://github.com/seaweedfs/seaweedfs/wiki/Master-Server-API

   https://github.com/seaweedfs/seaweedfs/wiki/Volume-Server-API

   https://github.com/seaweedfs/seaweedfs/wiki/Filer-Server-API

   

5. Health Monitoring and Recovery
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
     
     ```bash
     weed filer.copy ./path/to/folder_or_file http://filer_server:8888/
     ```
     
     
     
   - Backup filer metadata store
   
     https://github.com/seaweedfs/seaweedfs/wiki/Async-Filer-Metadata-Backup
   
     ```bash
     # filer server
     weed scaffold -config=filer -output=.
     weed filer.meta.backup
     ```
   
     ```toml
     # filer.toml
     [mysql]  # or memsql, tidb
     # CREATE TABLE IF NOT EXISTS `filemeta` (
     #   `dirhash`   BIGINT NOT NULL       COMMENT 'first 64 bits of MD5 hash value of directory field',
     #   `name`      VARCHAR(766) NOT NULL COMMENT 'directory or file name',
     #   `directory` TEXT NOT NULL         COMMENT 'full path to parent directory',
     #   `meta`      LONGBLOB,
     #   PRIMARY KEY (`dirhash`, `name`)
     # ) DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
     
     enabled = false
     # dsn will take priority over "hostname, port, username, password, database".
     # [username[:password]@][protocol[(address)]]/dbname[?param1=value1&...&paramN=valueN]
     dsn = "root@tcp(localhost:3306)/seaweedfs?collation=utf8mb4_bin"
     hostname = "localhost"
     port = 3306
     username = "root"
     password = ""
     database = ""              # create or use an existing database
     connection_max_idle = 2
     connection_max_open = 100
     connection_max_lifetime_seconds = 0
     interpolateParams = false
     # if insert/upsert failing, you can disable upsert or update query syntax to match your RDBMS syntax:
     enableUpsert = true
     upsertQuery = """INSERT INTO `%s` (`dirhash`,`name`,`directory`,`meta`) VALUES (?,?,?,?) AS `new` ON DUPLICATE KEY UPDATE `meta` = `new`.`meta`"""
     ```
   
     
   
   - Backup volume data (manual backup data by volume ID):
   
     https://github.com/seaweedfs/seaweedfs/wiki/Data-Backup
   
     
   
   - Backup PVC
   
     https://github.com/seaweedfs/seaweedfs/wiki/Kubernetes-Backups-and-Recovery-with-K8up
   
     
   
   - Health monitoring: api health

