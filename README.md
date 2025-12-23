## SeaweedFS



1. Install SeaweedFS (for filer metadata store, need mariadb installed with `sw_database` database and `filemeta` table and a user with suitable privileges)

   ```mysql
   # MARIADB
   CREATE DATABASE IF NOT EXISTS sw_database;
   USE sw_database;
   CREATE TABLE IF NOT EXISTS filemeta (
     `dirhash`   BIGINT NOT NULL       COMMENT 'first 64 bits of MD5 hash value of directory field',
     `name`      VARCHAR(766) NOT NULL COMMENT 'directory or file name',
     `directory` TEXT NOT NULL         COMMENT 'full path to parent directory',
     `meta`      LONGBLOB,
     PRIMARY KEY (`dirhash`, `name`)
   ) DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
   
   CREATE USER 'juniper'@'%' IDENTIFIED BY 'juniper@123';
   GRANT ALL PRIVILEGES ON sw_database.* TO 'juniper'@'%';
   FLUSH PRIVILEGES;
   ```

   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: mariadb-secret
     namespace: seaweedfs
   type: Opaque
   stringData:
     mysql_user: juniper
     mysql_password: juniper@123
   ```

   

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
     replicationPlacement: "001" 
   filer:
     enabled: true
     replicas: 3
     data:
       type: "hostPath"
       size: ""
       storageClass: ""
       hostPathPrefix: /storage
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
     volumeSizeLimitMB: 1000
     config: |-
       [master.maintenance]
       scripts = """
         lock
         volume.check.disk 
         volume.balance -force
         volume.fix.replication -force
         unlock
       """
       sleep_minutes = 30
       [master.volume_growth]
   	copy_1 = 7                # create 1 x 7 = 7 actual volumes
   	copy_2 = 6                # create 2 x 6 = 12 actual volumes
   	copy_3 = 3                # create 3 x 3 = 9 actual volumes
   	copy_other = 1            # create n x 1 = n actual volumes
   	threshold = 0.9           # create threshold
       
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
     data:
       type: "hostPath"
       storageClass: ""
       hostPathPrefix: /ssd
   volume:
     replicas: 3
     dataDirs:
     - name: data1
       type: "hostPath"
       hostPathPrefix: /ssd
       maxVolumes: 0 # If set to zero on non-windows OS, the limit will be auto configured. (default "7")
   ```

   

2. Expose Filer Node Port:

   ```bash
   k expose pod seaweedfs-filer-0 --type NodePort --name seaweedfs-filer-np -n seaweedfs
   ```

   

3. Install SeaweedFS CSI

   ```bash
   helm repo add seaweedfs-csi-driver https://seaweedfs.github.io/seaweedfs-csi-driver/helm
   
   helm show values seaweedfs-csi-driver/seaweedfs-csi-driver > seaweedfs_csi_values.yaml
   
   nano seaweedfs_csi_values.yaml
   
   helm upgrade --install seaweedfs-csi seaweedfs-csi-driver/seaweedfs-csi-driver   --namespace seaweedfs   --create-namespace -f seaweedfs_csi_values.yaml
   ```

   ```yaml
   # seaweedfs_csi_values.yaml
   seaweedfsFiler: seaweedfs-filer.seaweedfs.svc.cluster.local:8888
   cacheCapacityMB: 1024 # enable fuse cache, find cache in /var/cache/seaweedfs of csi-node pod
   mountService:
    enabled: true
   controller:
     replicas: 1
   ```

   **!IMPORTANT NOTES!:**

   Updating seaweed-csi-driver DaemonSet (DS) will break processeses who implement fuse mount: newly created pods will not remount net device.

   For safe update set `node.updateStrategy.type: OnDelete` for manual update. Steps:

   1. delete DS pods on the node where there is no seaweedfs PV
   2. cordon or taint node
   3. evict or delete pods with seaweedfs PV
   4. delete DS pod on node
   5. uncordon or remove taint on node
   6. repeat all steps on [all nodes]

   

4. Test CSI

```yaml
# nmaa-pvc.yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nmaa
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Delete
  storageClassName: seaweedfs-storage
  csi:
    driver: seaweedfs-csi-driver
    volumeHandle: nmaa
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nmaa-pvc
  namespace: seaweedfs
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: seaweedfs-storage
  volumeName: nmaa
```

```yaml
# test_deploy_3_rep_nmaa.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-app
  namespace: seaweedfs
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
            claimName: nmaa-pvc
```

```bash
k apply -f test_deploy_3_rep_nmaa.yml,nmaa-pvc.yml
```



#### Objectives:

1. High Availability (HA)

   - Data is always distributed to 3 nodes

   - Each one always has 1 replica (need min 2/3 nodes available)

   - If there is only 1 node avalable (not enough for replication), seaweedfs will restrict write operator (read-only data) https://github.com/seaweedfs/seaweedfs/discussions/2312

   - Volume is only used for storing data, the metadata need to be kept in a metadata store (default is local leveldb2 - not good for HA)

   - `lock`: prevent multiple processes run at them same time
     ![image-20250514154938594](./README.assets/image-20250514154938594.png)

   - `volume.balance` (need lock first, use `-force` to apply the changes): 
     ![image-20250513171253174](./README.assets/image-20250513171253174.png)

   - `volume.fix.replication` (-force to apply the changes - need lock first if force): 
     ![image-20250514144856031](./README.assets/image-20250514144856031.png)
     - When node up again, there are some data have 3 replicas => run fix.replication => remove extra reps

2. I/O Performance

   - Current status: 1 deployment 3 replicas, 1 pvc

     - 3 nodes k8s (3 volume servers), 100gb hdd

     - Max volume each volume server: auto configured
     - Max volume size: 1gb
   
   - Run fio on pod:
   
     ```bash
     apt update && apt install -y fio
     
     # sequential write
     fio --name=write-test --filename=/mnt/test/testfile --size=1G --bs=1M --rw=write --ioengine=libaio --direct=1
     
     # random read
     fio --name=read-test --filename=/mnt/test/testfile --size=1G --bs=4k --rw=randread --ioengine=libaio --direct=1
     
     # random write
     fio --name=randwrite-test --filename=/mnt/test/testfile --size=1G --bs=4k --rw=randwrite --ioengine=libaio --direct=1
     
     ```
     

3. API Support for File Operations

   https://github.com/seaweedfs/seaweedfs/wiki/Master-Server-API

   https://github.com/seaweedfs/seaweedfs/wiki/Volume-Server-API

   https://github.com/seaweedfs/seaweedfs/wiki/Filer-Server-API

   

4. Backup and Recovery
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

     

5. Others

   1. maxVolumes: 0 => auto configured as free disk space divided by volume size, if not set: 8 
   
   2. `volume.dataDirs[0].maxVolumes: 2` & `master.volumeSizeLimitMB: 1000` => exceed the limit => stop write
      - upload file 14mb  => 3 volume ids created (2 each node)
      - ![image-20250515103520628](./README.assets/image-20250515103520628.png)
   
   3. `volume.dataDirs[0].maxVolumes: 2` & `master.volumeSizeLimitMB: 1`
      - upload file 14mb => 3 volume ids created (2 each node) => stop write (cannot upload since master check volumeSizeLimit)
   4. Whenever a file uploaded to pvc, it create a volume id which includes the name of the pv as prefix name (remember to have enough resource or the write operator will be restricted)
   5. Use `volume.delete -node seaweedfs-volume-2.seaweedfs-volume.seaweedfs:8080 -volumeId 60` to delete volume. To delete all volumes, clean all data from all host then helm reinstall. To delete only all data: volume.vacuum or api /vacuum



6. Test
   1. Current Status:
      - 3 nodes k8s (3 volume servers), 100gb hdd
      - Max volume each volume server: auto configured
      - Max volume size: 1gb
      
   2. Test 
      - HA
        - Auto `volume.fix.replication` and `volume.balance` by adding master.config in values file above or creating a cronjob:
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
                          echo "[INFO] Volume List After Fixing and Balancing"                 	   
                          echo "volume.list" | weed shell -master $WEED_CLUSTER_SW_MASTER
                          echo "[INFO] Done"
                      env:
                        - name: WEED_CLUSTER_SW_MASTER
                          value: "seaweedfs-master.seaweedfs:9333"
                    restartPolicy: Never
          ```
          
          ![image-20250516163152410](./README.assets/image-20250516163152410.png)
        
      - Scalability: add volume server, increase volume replica, master api pre allocate volumes to add volume data.
      
      - Auto backup: add sidecars container in master pod
      
      - Increase volume number to increase concurrent reads and writes
      
      - Monitoring: send metrics





### Testing NMAA:

- Icinga: mount conf, plugins, scripts, zones
  ![image-20250606161404336](./README.assets/image-20250606161404336.png)
  ![image-20250606161434574](./README.assets/image-20250606161434574.png)
  ![image-20250606161500340](./README.assets/image-20250606161500340.png)



![image-20250610161610141](./README.assets/image-20250610161610141.png)



![image-20250610155955850](./README.assets/image-20250610155955850.png)

![image-20250610165514656](./README.assets/image-20250610165514656.png)







![image-20250612152543447](./README.assets/image-20250612152543447.png)

![image-20250612152558556](./README.assets/image-20250612152558556.png)

![image-20250612162432229](./README.assets/image-20250612162432229.png)

![image-20250613104623430](./README.assets/image-20250613104623430.png)

force delete để nó chạy trên node mới

với sts thì các pod của nó sẽ không reschedule khi node down ạ https://github.com/kubernetes/kubernetes/issues/74689#:~:text=Member-,This%20is%20working%20as%20designed,-.%20If%20the%20node, vì nó cần giữ trạng thái của volume mount trên node đó, tương tự với cụm seaweedfs, khi có một node down, pod volume server sẽ ko nhảy sang node khác ạ, trừ khi mình force delete nó thì nó sẽ sang node khác ạ. nếu mình xác định sẽ bỏ node down đó thì sẽ force delete còn không thì sẽ tăng replica lên 4 ạ.

Ngoài ra khi ghi mới data, seaweedfs sẽ tìm các volume thỏa mãn replica để ghi ạ, đọc thì sẽ tìm theo chunk fid nên đọc ghi không có vấn đề gì, còn khi fix replication thì sẽ có vấn đề vì nó chỉ check có rep đó thôi chứ không check size nên khi thấy 1 volume cũ 1 volume mới sẽ ko làm gì => ko fix đc rep

=> trong tương lai có thể có khả năng bị mất các chunk, vào weed shell, dùng `fs.verify` để check xem file nào hỏng 



![image-20250613173558506](./README.assets/image-20250613173558506.png)

- tạo deployment, ghi file, sau đó tắt một node, bật node mới sang node mới ghi xong lại chuyển về node cũ
=> deployment thì pod sẽ bị stuck ở terminating và trên node mới sẽ được tạo pod mới running

- tạo sts với hostpath, tắt bật vm, tạo sts với pvc, tắt bật vm
=> pod bị stuck ở terminating, không pod nào được tạo mới cho den khi force delete pod

- nếu các volume đạt max size nhưng max volume trên một server vẫn còn thì nó có tự tạo volume không
=> có

- khi data ghi mới thì chuyện gi se xay ra khi node down up lai
   tại sao fix.replication lại ko hoạt động với 2 volume chung id khác size: https://github.com/seaweedfs/seaweedfs/issues/900

  nếu 2 volume chung id nhưng khác size thì có thể fix (cần stop volume server): https://github.com/seaweedfs/seaweedfs/issues/6356#:~:text=To%20fix%20two%20volumes%20with%20the%20same%20volume%20id%20but%20diverging%20content%2C%20you%20can%20concatenate%20the%20two%20volumes%20but%20skipping%20the%20second%20volume%27s%20superblock%2C%20which%20is%208%20bytes%2C%20and%20run%20%22weed%20fix%22%20and%20%22weed%20compact%22.%20There%20are%20no%20existing%20scripts%20or%20code%20for%20this.%20Welcome%20to%20add%20a%20PR%20for%20this!

