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

# Dive into Swift

## Architecture
  ```ditaa {cmd=true args=["-E"]}
Region N
  ------------------------------------------
Zone N                                    |
                +--------+                  |
                |        |                  |
                |  Load  |                  |
                |Balancer|                  |
                |        |                  |
                +---+----+                  |
                    |                       |
                    |                       |
        +--------+          +--------+      |
        |        |          |        |      |
        |Proxy A | ......   |Proxy N |      |
        |        |          |        |      |
        +---+----+          +---+----+      |
                    |                       |
                    |                       |
        +--------+          +--------+      |
        |        |          |        |      |
        |Node A  | ......   |Node N  |      |
        |        |          |        |      |
        +---+----+          +---+----+      |
 ------------------------------------------|                              
  ```
 > ***Locality***
 + Region: Cities or Contries.
 + Zone: Racks.
 + Node: Phsical server.
  
 > ***Components***
 + Load Balancer: Distribute traffic to proxies.
 + Proxy:
   + Provide Rest API for client
   + Forward client request to nodes' account, container, object, etc.   
   + Leverage memcached caching token, account and container data.
 + Account server: Provide tenant service, isolate by metadata.
 + Container server: Similar to folder, multiple files be stored in it.
 + Object server: Similar to files, object data would be stored in the nodes' file system.
 + Consistency server: Consist of
   + Replicators:
     + Replicate objects and make a system in a consistent state
     + Recover disk failure and network outages situation
     + Execution:
       + Start periodically, default 30s.
       + Proxy server failed to store the third copy, but it will return success to the client, and the replicator will keep trying to store the third copy.
       + A certain replication broken dected.
       + Physical disks updates, data needed rebalance.
   + Updaters:
     + Update metadata
     + Recover failure caused by container, account metadata high load
   + Auditors:
     + Delete problematic account, container or objects and replicate from other server
     + Recover dbs or files which have bit rot problem
 + SQLite: Store account, container data.

> ***Mechnanism***
+ Recommend XFS file system by Swift official.
+ Node Selection: Proxy server locate stored data, `by to the ring of account, container, object.`
    > Configuration of ring files needs to be put in all proxy nodes, also storage nodes for consistency servers.
```ditaa {cmd=true args=["-E"]}
                                                            
                        +--------+                          
                        |        |    find                  
Container name/ ---->   | Ring   |   -----> Node X/N/K      
file name               |        |                          
                        +---+----+                                            
```

+ Ring files format:
  +  object.ring.gz
  +  container.ring.gz
  +  account.ring.gz
+ Ring configuration:
  + Region, zone and disk: ...***skip***...
  + Partition: Swift plan each disks to several partitions, partition is a basic unit for replication.
  + Replica: Amount of replication, convention with 3.
+ Ring Model Definition:
```
Ring._devs
Ring._replica2part2dev
Ring.overload 
Ring.replicas 
Ring.part_power
....
```

```
Ring._devs: Store storage nodes information

id                 unique integer identifier amongst devices
index              offset into the primary node list for the partition
weight             a float of the relative weight of this device as compared to
                   others; this indicates how many partitions the builder will try
                   to assign to this device
region             -
zone               integer indicating which zone the device is in; a given
                   partition will not be assigned to multiple devices within the
                   same zone
ip                 the ip address of the device
port               the tcp port of the device
device             the device's name on disk (sdb1, for example)
meta               general use 'extra' field; for example: the online date, the
                   hardware description

replication_ip     -
replication_port   -
```

```
Ring._replica2part2dev: Define the mapping of logic model to phsical model
Generation principle:
    。 Assign amount of partition according to dev.weight
    。 As unique as possible, Ex: If cluster have 3 regions, replicate 3 data to 3 regions satify as unique as possible principle
    。 Not allow the backup of the same partition in the same disk

                         dev_id
                              |          
|   +--------|-----------------------------+
           | 0 | array([1, 3, 1, 5, 5, 3, 2, 1,.....])
replica    | 1 | array([0, 2, 4, 2, 6, 4, 3, 4,.....])
           | 2 | array([2, 1, 6, 4, 7, 5, 6, 5,.....])
|   +--------------------------------------+
                        0 1 2 3 4 5 6 7..............
     -------------------------------------->
             partition

3 array indicates the setting of replication
Each index of an array indicates partition_id (locate objects)
The same index of all array indicates dev_id (locate storage nodes)
```

```
Ring.replicas: Define ring replication, affect vertical length of _replica2part2dev
```

```
Ring.overload: To satify backup in different region/zone/node/dev as unique as possible
```

```
Ring.part_power: Define amount of partition, 2^part_power
```

+ Behind ring rebalance
  + Step 1: Make replicas plan according rebalance and dispersion principle, counting the amount of partition can be assigned for each devices.
  + Step 2: According to replica plan, plan total partitions (replica*part) for each tier and get each devices needed partion.
  + Step 3: Setup the length of _replica2part2dev.
  + Step 4: If administrator delete partition on disk, dev_id set to none in _replica2part2dev accordingly.
  + Step 5: Gather partitions which are not satify dispersion principle.
  + Step 6: Gather partitions which are overweight.
<details>
  <summary>Step 1 Code</summary>
  
  ### rebalance()
  ```python
  def rebalance(self, seed=None):
    ....
    replica_plan = self._build_replica_plan() # <--- call _build_target_replicas_by_tier()
  ```

  ### _build_target_replicas_by_tier()
  ```python
def _build_target_replicas_by_tier(self):
        # According dev_weight accumulate parent ratio
        weighted_replicas = self._build_weighted_replicas_by_tier()
        wanted_replicas = self._build_wanted_replicas_by_tier()
        max_overload = self.get_required_overload(weighted=weighted_replicas,
                                                  wanted=wanted_replicas)
        if max_overload <= 0.0:
            return wanted_replicas
        else:
            overload = min(self.overload, max_overload)
        self.logger.debug("Using effective overload of %f", overload)
        target_replicas = defaultdict(float)
        for tier, weighted in weighted_replicas.items():
            m = (wanted_replicas[tier] - weighted) / max_overload
            target_replicas[tier] = m * overload + weighted
  ```

### _build_weighted_replicas_by_tier()
#### Formula:
$$\sum_{i=1}^c\frac{dev_c[weight]}{\sum_{j=1}^d {dev_j[weight]}} * replicas$$
*c: Total children* <br />
*d: Total devices*

  ```python
    def _build_weighted_replicas_by_tier(self):
        """
        Returns a dict mapping <tier> => replicanths for all tiers in
        the ring based on their weights.
        """
        weight_of_one_part = self.weight_of_one_part()

        # assign each device some replicanths by weight (can't be > 1)
        weighted_replicas_for_dev = {}
        devices_with_room = []
        for dev in self._iter_devs():
            if not dev['weight']:
                continue
            weighted_replicas = (
                dev['weight'] * weight_of_one_part / self.parts)
            if weighted_replicas < 1:
                devices_with_room.append(dev['id'])
            else:
                weighted_replicas = 1
            weighted_replicas_for_dev[dev['id']] = weighted_replicas

        while True:
            remaining = self.replicas - sum(weighted_replicas_for_dev.values())
            if remaining < 1e-10:
                break
            devices_with_room = [d for d in devices_with_room if
                                 weighted_replicas_for_dev[d] < 1]
            rel_weight = remaining / sum(
                weighted_replicas_for_dev[d] for d in devices_with_room)
            for d in devices_with_room:
                weighted_replicas_for_dev[d] = min(
                    1, weighted_replicas_for_dev[d] * (rel_weight + 1))

        weighted_replicas_by_tier = defaultdict(float)
        for dev in self._iter_devs():
            if not dev['weight']:
                continue
            assigned_replicanths = weighted_replicas_for_dev[dev['id']]
            dev_tier = (dev['region'], dev['zone'], dev['ip'], dev['id'])
            for i in range(len(dev_tier) + 1):
                tier = dev_tier[:i]
                weighted_replicas_by_tier[tier] += assigned_replicanths

        # belts & suspenders/paranoia -  at every level, the sum of
        # weighted_replicas should be very close to the total number of
        # replicas for the ring
        validate_replicas_by_tier(self.replicas, weighted_replicas_by_tier)

        return weighted_replicas_by_tier
  ```


  
### _build_wanted_replicas_by_tier()
*dispersed_replicas: ignore weight and caculate replicas can be assigned*
*wanted_replicas: make a decision between weight_replicas and dispersed_replicas*
$$ wanted\_replicas=\left\{
\begin{array}{rcl}
dispersed\_replicas[min]       &      & {weighted\_replicas      <      dispersed\_replicas[min]}\\
dispersed\_replicas[max]     &      & {weighted\_replicas      >      dispersed\_replicas[max]}\\
weighted\_replicas      &      & {dispersed\_replicas[min] \leq weighted\_replicas \leq dispersed\_replicas[max]}\\
\end{array} \right. $$
  ```python
def _build_wanted_replicas_by_tier(self):
        ...
        dispersed_replicas = {
            t: {
                'min': math.floor(r),
                'max': math.ceil(r),
            } for (t, r) in
            self._build_max_replicas_by_tier(bound=float).items()
        }
        ...
        for t in tiers_to_spread:
                    replicas = to_place[t] + (
                        weighted_replicas[t] * rel_weight)
                    if replicas < dispersed_replicas[t]['min']:
                        replicas = dispersed_replicas[t]['min']
                    elif (replicas > dispersed_replicas[t]['max'] and
                          not device_limited):
                        replicas = dispersed_replicas[t]['max']
                    if replicas > num_devices[t]:
                        replicas = num_devices[t]
                    to_place[t] = replicas
        ...
  ```
  </details>

<details>
  <summary>Step 2 Code</summary>

### _build_wanted_replicas_by_tier()
```python
        def _set_parts_wanted(self, replica_plan):
            ....
            for dev in self._iter_devs():
                if not dev['weight']:
                    # With no weight, that means we wish to "drain" the device. So
                    # we set the parts_wanted to a really large negative number to
                    # indicate its strong desire to give up everything it has.
                    dev['parts_wanted'] = -self.parts * self.replicas
                else:
                    tier = (dev['region'], dev['zone'], dev['ip'], dev['id'])
                    dev['parts_wanted'] = parts_by_tier[tier] - dev['parts']
```
</details>

<details>
  <summary>Step 3 Code</summary>

### rebalance()
```python
        def rebalance(self, seed=None):
            ....
            assign_parts = defaultdict(list)
            # gather parts from replica count adjustment
            self._adjust_replica2part2dev_size(assign_parts)

'''
Considering two situation:
1. _replica2part2dev is None, iterate all partion add it to assign_part.
       
            
    |   +--------|-----------------------------+
         | 0 | array([NONE, NONE, NONE, .....])
replica  | 1 | array([NONE, NONE, NONE,.....])
         | 2 | array([NONE, NONE, NONE,.....])
    |   +--------------------------------------+
              0 1 2 3 4 5 6 7.....................
         -------------------------------------->

2. _replica2part2dev is not None, increase/decrease replication would affect the length of _replica2part2dev vertically, iterate partion and append to assign_part.
Ex: 3 -> 4 replicas
    |          
    |   +--------|-----------------------------+
         | 0 | array([1, 3, 4, .....])
replica  | 1 | array([2, 4, 5,.....])
         | 2 | array([3, 1, 6,.....])
         | 3 | array([NONE, NONE, NONE,.....])
    |   +--------------------------------------+
              0 1 2 3 4 5 6 7.....................
         -------------------------------------->
             partition

Ex: 3 -> 2 replicas
    |          
    |   +--------|-----------------------------+
         | 0 | array([1, 3, 4, .....])
replica  | 1 | array([2, 4, 5,.....])
    |   +--------------------------------------+
              0 1 2 3 4 5 6 7.....................
         -------------------------------------->
             partition
'''
```
</details>

<details>
  <summary>Step 4 Code</summary>

### _gather_parts_from_failed_devices()
```python
        def _gather_parts_from_failed_devices(self, assign_parts):
            ....
            while self._remove_devs:
            remove_dev_id = self._remove_devs.pop()['id']
            self.logger.debug("Removing dev %d", remove_dev_id)
            self.devs[remove_dev_id] = None
            removed_devs += 1
```
</details>

<details>
  <summary>Step 5 Code</summary>

### _gather_parts_for_dispersion()
- Iterate specific partition and get corresponding dev_id
- From root to leaf, if any node replicas greater than replica_plan (Means not satisfying dispersion principle), store it to undispersed_dev_replicas(dev,replica).
- Iterate undispersed_dev_replicas, store in assign_parts, and set original dev_id to none in _replica2part2dev (Wait for reassign).
```python
        def _gather_parts_for_dispersion(self, assign_parts, replica_plan):
      ......
      for replica in self._replicas_for_part(part):
                dev_id = self._replica2part2dev[replica][part]
                if dev_id == NONE_DEV:
                    continue
                dev = self.devs[dev_id]
      #From root to leaf, if any node replicas greater than replica_plan (Means not satisfying dispersion principle), store it to undispersed_dev_replicas(dev,replica).
                if all(replicas_at_tier[tier] <=
                       replica_plan[tier]['max']
                       for tier in dev['tiers']):
                    continue
                undispersed_dev_replicas.append((dev, replica))

       for dev, replica in undispersed_dev_replicas:
                ...... 
                if (self._last_part_moves[part] < self.min_part_hours and
                        not replicas_at_tier[dev['tiers'][-1]] > 1):
                    continue
                dev['parts_wanted'] += 1
                dev['parts'] -= 1
       #store in assign_parts
                assign_parts[part].append(replica)
       #set original dev_id to none in _replica2part2dev
                self._replica2part2dev[replica][part] = NONE_DEV
      ......
```
</details>

<details>
  <summary>Step 6 Code</summary>

### _gather_parts_for_balance()
- Iterate topology tree, if dev['parts_wanted']<0 (Means device is overweight, since the path is not balance weight, child parts_wanted is less than 0), then append specific replica(path) to overweight_dev_replica.
- Iterate overweight_dev_replica, set each partitions' dev_id to none for specific replica, then append replica to assign_parts for specific partition (Wait for reassign).
```python
    def _gather_parts_for_balance(self, assign_parts, replica_plan):
.....
  #From root to leaf, check if dev['parts_wanted']<0
  for replica in self._replicas_for_part(part):
                dev_id = self._replica2part2dev[replica][part]
                if dev_id == NONE_DEV:
                    continue
                dev = self.devs[dev_id]
                for tier in dev['tiers']:
                    replicas_at_tier[tier] += 1
                if dev['parts_wanted'] < 0:
                    overweight_dev_replica.append((dev, replica))
  #Set dev_id to none, and plan to be reassign
  for dev, replica in overweight_dev_replica:
                if self._last_part_moves[part] < self.min_part_hours:
                    break
                if any(replica_plan[tier]['min'] <=
                       replicas_at_tier[tier] <
                       replica_plan[tier]['max']
                       for tier in dev['tiers']):
                    continue
                dev['parts_wanted'] += 1
                dev['parts'] -= 1
                assign_parts[part].append(replica)
                self.logger.debug(
                    "Gathered %d/%d from dev %d [weight disperse]",
                    part, replica, dev['id'])
                self._replica2part2dev[replica][part] = NONE_DEV
```
</details>
