# Notes

## Information about the tenant
```sh
swift stat
```

## List containers for account
```sh
swift list
```

## Upload files
> the container will automatically be created if it's not existed
```sh
swift upload <container-name> <file-name>
```

## Display container information
```sh
swift stat <container-name>
```

## Display object information
```sh
swift stat <container-name> <file-name>
```

## List all objects in container of tenant 
```sh
swift list <container-name>
```

## Download objects 
```sh
swift download <container-name> <file-name>
```

## Modify objects ACL rules
> Public accessible
>> Access URL: http://<Proxy-Server IP>:8080/v1/AUTH_admin/<container-name>/<file-name>
```sh
swift post -r '.r:*' <container-name>
```

> Reset permission
```sh
swift post -r '' <container-name>
```

## Delete objects 
```sh
swift delete <container-name> <file-name>
```

## Delete containers 
```sh
swift delete <container-name>
```

## Disk usage stats 
```sh
swift-recon -d
```

## Cluster load average stats 
```sh
swift-recon -l
```
## Cluster replication stats 
```sh
swift-recon -r
```
