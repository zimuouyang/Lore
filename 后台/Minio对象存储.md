```shell
 ./minio server --console-address :9090 --address :9091 /usr/software/minio/data 
 
 nohup /usr/software/minio/minio server -console-address :9090 --address :9091  /usr/software/minio/data > /usr/software/minio/minio.log 2>&1 &
```

