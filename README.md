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
 + Object (File+Metadata) server: Similar to files, object data would be stored in the nodes' file system.
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

 > ***User Manual***
+ Every Object has a URL: *http://example.com/v1/account/container/object*
  + account:
    + Each account has its own URL
    + Swift is multi-tenant
  + container:
    + Namespaces used to group objects within an account
    + Containers are unlimited
  + object:
    + Each object is represented as a URL
    + Users name the object
    + object names may contain /, so ***pseudo-nested directories are possible***
+ Swift use the first 4 bytes of hash to find the placement of ring
  + ETag: md5(object content)
  + Hash: md5(/account/container/object)
  + Partition hash calculation
    + Ex: Hash=3098f203544d22b361d6ee1cfc7406e1, Partition power=16
      + First 4 bytes: 30 98 f2 03
      + Right Shift = 32 - Partition power(16) = 16 bits 
      + Convert Hex to binary: 0011(3) 0000(0) 1001(9) 1000(8)
      + Convert binary to decimal: 0011000010011000=12440
      + Ans: Partition=12440
+ If the object is over the threshold (default: 5G), data would be fragmentd.
+ Storage Policy: Erasure coding (EC)
  + Proxy server streams an object and buffers up a segment (Portion of object's content) of data (size is configurable).
  + Proxy server invokes PyECLib to code data into smaller fragments (Encode the segment and cut into multiple chunks), if clients wants to read data just get the fragments aka fragment archives (If one fragment is missing, just find the other copy), then decode and concatnate into the segment.
    + Object reconstructor: Check all fragments and its peer, fix by copy the peer and replace to the broken fragments.
  + Proxy server send out the fragments based on ring.
  + Repeat until client is done sending data.
<details>
  <summary>Logical to physical mapping caculation code</summary>

### get_part()
```python
        #self._part_shift = 32 - self.part_power
def get_part(self, account, container=None, obj=None): """
        Get the partition for an account/container/object.

        :param account: account name
        :param container: container name
        :param obj: object name
        :returns: the partition number
        """
        key = hash_path(account, container, obj, raw_digest=True) if time() > self._rtime:
            self._reload()
        part = struct.unpack_from('>I', key)[0] >> self._part_shift
        return part
```

### _get_part_nodes()
- Get all devices of particular partition according get_part().
```python
        def _get_part_nodes(self, part):
        part_nodes = []
        seen_ids = set() for r2p2d in self._replica2part2dev_id: if part < len(r2p2d):
                dev_id = r2p2d[part] if dev_id not in seen_ids:
                    part_nodes.append(self.devs[dev_id])
                    seen_ids.add(dev_id) return [dict(node, index=i) for i, node in enumerate(part_nodes)]
```
</details>

  + Object Location Mapping:
    + Object: cloudcat.jpg
    + Partition: 12440
    + Hash: 3098f203544d22b361d6ee1cfc7406e1
    + Hash location on disk: /srv/node/d2/objects/12440/6e1(last characters of hash)/3098f203544d22b361d6ee1cfc7406e1(Full hash)/1478736472.70261.data(object timestamp)
+ HTTP methods for objects:
  + GET: Downloads an object with its metadata.
  + PUT: Creates new object with specified data content and metadata.
  + COPY: Copies an object to another object.
  + DELETE: Deletes an object.
  + HEAD: Shows object metadata.
  + POST: Creates or updates object metadata.
+ HTTP code for troubleshooting:
  + 302 moved permanently: SSL is required and was not used
  + 401 unauthorized:
    + memcached not running on a node
      + Check /var/log/memcached.log if problem persists.
    + memcached connection timeout during system heavy load
      + grep /var/log/all.log
      + swift-init proxy restart
  + 403 forbidden:
    + User doesn't have permission to access the swift account
  + 412 precondition failed:
    + Missing path root in the request
    + Request hanging
    + Check the connection between client and Swift API server
    + Check the setting of load balancer, and restart keepalived (option)
    + grep "ERROR" in /var/log/swift/all.log
  + 503 service unavailable:
    + Find transaction ID from /var/log/swift/proxy_access.log
    + grep the transaction ID through /var/log/swift/all.log
+ Check the health of 
  + mounted disks: sdt health </mount/point>
    + grep "Fail" or "sdt" in /var/log/ssnoded.log
  + node: sudo ssdiag
+ Display all drivers and status

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
  + Step 7: Assign appropriate devices for assign_parts.
  + Step 8: Save the results as ring file.
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


<details>
  <summary>Step 7 Code</summary>

### _reassign_parts()
- First of all, creates tier2devs list (tier -> [dev1, dev2...]), collects devices in each tiers.
- Select the most avaliable tier, which is not reach max replicas.
- Get the last device from the tier(tier2devs).
- Choose the selected device as new device of reassign_partition, update _replica2part2dev corresponding dev_id.
```python
        assign_parts_list = list(assign_parts.items())
self._reassign_parts(assign_parts_list, replica_plan)
def _reassign_parts(self, reassign_parts, replica_plan):
    .....
    # Create tier -> [dev1, dev2...] list
    for dev in available_devs:
        for tier in dev['tiers']:
            tier2devs[tier].append(dev)  # <-- starts out sorted!
                tier2dev_sort_key[tier].append(dev['sort_key'])
                tier2sort_key[tier] = dev['sort_key']
    # Assign new device for reassign_parts
    for part, replace_replicas in reassign_parts:
        .......
        for replica in replace_replicas:
                tier = ()
                depth = 1
                while depth <= max_tier_depth:
                    # Select the tier which are not reach max replicas
                    candidates = [t for t in tier2children[tier] if
                                  replicas_at_tier[t] <
                                  replica_plan[t]['max']]

                    if not candidates:
                        raise Exception('no home for %s/%s %s' % (
                            part, replica, {t: (
                                replicas_at_tier[t],
                                replica_plan[t]['max'],
                            ) for t in tier2children[tier]}))
                    # Select the most avaliable tier
                    tier = max(candidates, key=lambda t:
                               parts_available_in_tier[t])
                    depth += 1
                # Get the last device of the tier
                dev = tier2devs[tier][-1]
                dev['parts_wanted'] -= 1
                dev['parts'] += 1
                for tier in dev['tiers']:
                    parts_available_in_tier[tier] -= 1
                    replicas_at_tier[tier] += 1
                # Selected device as the new device of replica2part
                self._replica2part2dev[replica][part] = dev['id']
```
</details>

<br />

+ Behind store an object:
  + When send a Put request, proxy server:
    + Calculate hash value of (/account/container/object) to get partition ID.
    + Get all devices IP and port according to partition ID.
    + Try to connect all devices, denied the client request if half of devices couldn't connect.
    + Try to create object through object server (Object would asynchronous invoke container service to update container database).
    + If two out of three written succeed, proxy server would return success to client.
+ Behind get an object:
  + When send a Get request, proxy server:
    + Calculate hash value of (/account/container/object) to get partition ID.
    + Get all devices IP and port according to partition ID.
    + Sort all devices, connect to the device, if succeed, return data to client, otherwise try next device.
+ Behind the erasure codes:
  + Simple idea: two points define a line, if the data can be represent by a formula, we can find any pieces alone the line. (If the data encoded to multiple chunks, the degree of equation growth to quadratic, cubic, polynomial equation)
    + [![Linear equation](https://upload.wikimedia.org/wikipedia/commons/thumb/0/0e/Linear_Function_Graph.svg/220px-Linear_Function_Graph.svg.png)](https://zh-yue.wikipedia.org/wiki/%E7%B7%9A%E6%80%A7%E6%96%B9%E7%A8%8B%E5%BC%8F)
  + Ex: Set 4 chunks as parameter to encode "Good", correction code: P1 and P2 (Allow amount of P<sub>1...n</sub> broken data)
    + G = X1, o = X2, o = X3, d = X4, X1~X4 are real number:
      + X1 = D1
      + X2 = D2
      + X3 = D3
      + X4 = D4
      + X1 + X2 + X3 + X4 = P1
      + X1 + 2\*X2 + 4\*X3 + 8\*X4 = P2
      + 1~7 can be represented as identity matrix:
    $$\begin{bmatrix}
    {1}&{0}&{0}&{0}\\
    {0}&{1}&{0}&{0}\\
    {0}&{0}&{1}&{0}\\
    {0}&{0}&{0}&{1}\\
    {1}&{1}&{1}&{1}\\
    {1}&{2}&{4}&{8}\\
    \end{bmatrix}$$
    + Encode: We save the final result as encoded data ( *[D1 D2 D3 D4 P1 P2]* )
    $$\begin{bmatrix}
    {1}&{0}&{0}&{0}\\ 
    {0}&{1}&{0}&{0}\\
    {0}&{0}&{1}&{0}\\
    {0}&{0}&{0}&{1}\\
    {1}&{1}&{1}&{1}\\
    {1}&{2}&{4}&{8}\\
    \end{bmatrix} \times 
    \begin{bmatrix}
    {D1}\\ 
    {D2}\\
    {D3}\\
    {D4}\\
    \end{bmatrix} =
    \begin{bmatrix}
    {D1}\\ 
    {D2}\\
    {D3}\\
    {D4}\\
    {P1}\\
    {P2}\\
    \end{bmatrix}$$
     + Decode: Assume the "d" (D4) of "Good" is broken,
       + Select 4 rows (Since amount D<sub>1..n</sub>, n=4) from top to bottom of the matrix, and skip the broken row:
          + $$\begin{bmatrix}{1}&{0}&{0}&{0}\\{0}&{1}&{0}&{0}\\{0}&{0}&{1}&{0}\\{}&{}&{}&{}\\{1}&{1}&{1}&{1}\\\end{bmatrix}$$
        + Solve the equation:
          + $$\begin{bmatrix}{1}&{0}&{0}&{0}\\{0}&{1}&{0}&{0}\\{0}&{0}&{1}&{0}\\{}&{}&{}&{}\\{1}&{1}&{1}&{1}\\\end{bmatrix} \times \begin{bmatrix}{D1}\\{D2}\\{D3}\\{D4}\\\end{bmatrix} = \begin{bmatrix}{D1}\\{D2}\\{D3}\\{}\\{P1}\\\end{bmatrix}$$
        + A*A<sup>-1</sup> = E (Identity matrix), multiple A<sup>-1</sup> in both side:
          + $$\begin{bmatrix}{1}&{0}&{0}&{0}\\{0}&{1}&{0}&{0}\\{0}&{0}&{1}&{0}\\{}&{}&{}&{}\\{1}&{1}&{1}&{1}\\\end{bmatrix}^{-1} \times \begin{bmatrix}{1}&{0}&{0}&{0}\\{0}&{1}&{0}&{0}\\{0}&{0}&{1}&{0}\\{}&{}&{}&{}\\{1}&{1}&{1}&{1}\\\end{bmatrix} \times \begin{bmatrix}{D1}\\{D2}\\{D3}\\{D4}\\\end{bmatrix} = \begin{bmatrix}{1}&{0}&{0}&{0}\\{0}&{1}&{0}&{0}\\{0}&{0}&{1}&{0}\\{}&{}&{}&{}\\{1}&{1}&{1}&{1}\\\end{bmatrix}^{-1} \times \begin{bmatrix}{D1}\\{D2}\\{D3}\\{}\\{P1}\\\end{bmatrix}$$
          + $$ \begin{bmatrix}{D1}\\{D2}\\{D3}\\{D4}\\\end{bmatrix} = \begin{bmatrix}{1}&{0}&{0}&{0}\\{0}&{1}&{0}&{0}\\{0}&{0}&{1}&{0}\\{}&{}&{}&{}\\{1}&{1}&{1}&{1}\\\end{bmatrix}^{-1} \times \begin{bmatrix}{D1}\\{D2}\\{D3}\\{}\\{P1}\\\end{bmatrix}$$
        + Solve above equation to fix D4, but check whether the matrix is invertible before solving.
          + Evaluate invertible matrix by two ways:
            + Determinant of matrix not equal to 0.
            + Rank(matrix) = Dimension(matrix)
+ Behind API request:
  + HTTP Buffer:
    + Proxy server implement `bufferedhttp.py` to buffer the client's request for every entrypoints.
    ```python
    # swift/common/bufferedhttp.py
    ...
    httplib._MAXHEADERS = constraints.MAX_HEADER_COUNT * 1.6
    green_httplib._MAXHEADERS = constraints.MAX_HEADER_COUNT * 1.6
    ...
    ```
    + Ex: `swift/proxy/controller/obj.py`
    ```python
    ...
    # NOTE: swift_conn
    # You'll see swift_conn passed around a few places in this file. This is the
    # source bufferedhttp connection of whatever it is attached to.
    ...
    ```
  + TCP connection for HTTP request: 
    + Principle: Multiple accounts/containers/objects can't use the same TCP connection (Since the URL is different, Ex: a/b/c, a/b/d).
    + `swift upload container object`: Using the same connection from authentication to sending data, client sends FIN to server once done with transporting data. 
      ```
      09:43:29.400306 lo    In  IP (tos 0x0, ttl 64, id 51586, offset 0, flags [DF], proto TCP (6), length 60)
        127.0.0.1.50154 > 127.0.0.1.8080: Flags [S], cksum 0xfe30 (incorrect -> 0x4cb9), seq 1262896167, win 65495, options [mss 65495,sackOK,TS val 3516972735 ecr 0,nop,wscale 7], length 0
      09:43:29.400353 lo    In  IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 60)
          127.0.0.1.8080 > 127.0.0.1.50154: Flags [S.], cksum 0xfe30 (incorrect -> 0xa1b3), seq 1922934786, ack 1262896168, win 65483, options [mss 65495,sackOK,TS val 3516972735 ecr 3516972735,nop,wscale 7], length 0
      09:43:29.400390 lo    In  IP (tos 0x0, ttl 64, id 51587, offset 0, flags [DF], proto TCP (6), length 52)
          127.0.0.1.50154 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0xc86f), ack 1, win 512, options [nop,nop,TS val 3516972735 ecr 3516972735], length 0
      09:43:29.400502 lo    In  IP (tos 0x0, ttl 64, id 51588, offset 0, flags [DF], proto TCP (6), length 193)
          127.0.0.1.50154 > 127.0.0.1.8080: Flags [P.], cksum 0xfeb5 (incorrect -> 0xfa19), seq 1:142, ack 1, win 512, options [nop,nop,TS val 3516972735 ecr 3516972735], length 141: HTTP, length: 141
        GET /auth/v1.0 HTTP/1.1
        Host: 127.0.0.1:8080
        User-Agent: curl/7.81.0
        Accept: */*
        X-Storage-User: test:tester
        X-Storage-Pass: testing

      09:43:29.400509 lo    In  IP (tos 0x0, ttl 64, id 26464, offset 0, flags [DF], proto TCP (6), length 52)
          127.0.0.1.8080 > 127.0.0.1.50154: Flags [.], cksum 0xfe28 (incorrect -> 0xc7e3), ack 142, win 511, options [nop,nop,TS val 3516972735 ecr 3516972735], length 0
      09:43:29.404817 lo    In  IP (tos 0x0, ttl 64, id 26465, offset 0, flags [DF], proto TCP (6), length 468)
          127.0.0.1.8080 > 127.0.0.1.50154: Flags [P.], cksum 0xffc8 (incorrect -> 0xafbd), seq 1:417, ack 142, win 512, options [nop,nop,TS val 3516972740 ecr 3516972735], length 416: HTTP, length: 416
        HTTP/1.1 200 OK
        Content-Type: text/html; charset=UTF-8
        X-Auth-Token: AUTH_tk6ccb8979f39843b5b18d0bf4f3f23803
        X-Storage-Token: AUTH_tk6ccb8979f39843b5b18d0bf4f3f23803
        X-Auth-Token-Expires: 84782
        X-Storage-Url: http://127.0.0.1:8080/v1/AUTH_test
        Content-Length: 0
        X-Trans-Id: tx6e1ce3fc574f4a4c97342-0063b2a741
        X-Openstack-Request-Id: tx6e1ce3fc574f4a4c97342-0063b2a741
        Date: Mon, 02 Jan 2023 09:43:29 GMT

      09:43:29.404838 lo    In  IP (tos 0x0, ttl 64, id 51589, offset 0, flags [DF], proto TCP (6), length 52)
          127.0.0.1.50154 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0xc63b), ack 417, win 509, options [nop,nop,TS val 3516972740 ecr 3516972740], length 0
      09:43:29.405127 lo    In  IP (tos 0x0, ttl 64, id 51590, offset 0, flags [DF], proto TCP (6), length 52)
          127.0.0.1.50154 > 127.0.0.1.8080: Flags [F.], cksum 0xfe28 (incorrect -> 0xc637), seq 142, ack 417, win 512, options [nop,nop,TS val 3516972740 ecr 3516972740], length 0
      09:43:29.405275 lo    In  IP (tos 0x0, ttl 64, id 26466, offset 0, flags [DF], proto TCP (6), length 52)
          127.0.0.1.8080 > 127.0.0.1.50154: Flags [F.], cksum 0xfe28 (incorrect -> 0xc636), seq 417, ack 143, win 512, options [nop,nop,TS val 3516972740 ecr 3516972740], length 0
      09:43:29.405283 lo    In  IP (tos 0x0, ttl 64, id 51591, offset 0, flags [DF], proto TCP (6), length 52)
          127.0.0.1.50154 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0xc636), ack 418, win 512, options [nop,nop,TS val 3516972740 ecr 3516972740], length 0
      ```
    + `swift upload container object1 object2`: Using different TCP connection to upload the objects
    ```
    09:58:19.921273 lo    In  IP (tos 0x0, ttl 64, id 30618, offset 0, flags [DF], proto TCP (6), length 60)
    127.0.0.1.34706 > 127.0.0.1.8080: Flags [S], cksum 0xfe30 (incorrect -> 0x2a38), seq 402276262, win 65495, options [mss 65495,sackOK,TS val 3517863256 ecr 0,nop,wscale 7], length 0
    09:58:19.921283 lo    In  IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 60)
        127.0.0.1.8080 > 127.0.0.1.34706: Flags [S.], cksum 0xfe30 (incorrect -> 0x2aa8), seq 462732527, ack 402276263, win 65483, options [mss 65495,sackOK,TS val 3517863256 ecr 3517863256,nop,wscale 7], length 0
    09:58:19.921290 lo    In  IP (tos 0x0, ttl 64, id 30619, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34706 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0x5164), ack 1, win 512, options [nop,nop,TS val 3517863256 ecr 3517863256], length 0
    09:58:19.921312 lo    In  IP (tos 0x0, ttl 64, id 30620, offset 0, flags [DF], proto TCP (6), length 213)
        127.0.0.1.34706 > 127.0.0.1.8080: Flags [P.], cksum 0xfec9 (incorrect -> 0x4f54), seq 1:162, ack 1, win 512, options [nop,nop,TS val 3517863256 ecr 3517863256], length 161: HTTP, length: 161
      GET /auth/v1.0 HTTP/1.1
      Host: 127.0.0.1:8080
      Accept-Encoding: identity
      x-auth-user: test:tester
      x-auth-key: testing
      user-agent: python-swiftclient-1.2.3

    09:58:19.921314 lo    In  IP (tos 0x0, ttl 64, id 28763, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.8080 > 127.0.0.1.34706: Flags [.], cksum 0xfe28 (incorrect -> 0x50c4), ack 162, win 511, options [nop,nop,TS val 3517863256 ecr 3517863256], length 0
    09:58:19.922423 lo    In  IP (tos 0x0, ttl 64, id 28764, offset 0, flags [DF], proto TCP (6), length 468)
        127.0.0.1.8080 > 127.0.0.1.34706: Flags [P.], cksum 0xffc8 (incorrect -> 0x4df1), seq 1:417, ack 162, win 512, options [nop,nop,TS val 3517863257 ecr 3517863256], length 416: HTTP, length: 416
      HTTP/1.1 200 OK
      Content-Type: text/html; charset=UTF-8
      X-Auth-Token: AUTH_tk6ccb8979f39843b5b18d0bf4f3f23803
      X-Storage-Token: AUTH_tk6ccb8979f39843b5b18d0bf4f3f23803
      X-Auth-Token-Expires: 83891
      X-Storage-Url: http://127.0.0.1:8080/v1/AUTH_test
      Content-Length: 0
      X-Trans-Id: tx1ee8453cb0bf4254905a3-0063b2aabb
      X-Openstack-Request-Id: tx1ee8453cb0bf4254905a3-0063b2aabb
      Date: Mon, 02 Jan 2023 09:58:19 GMT

    09:58:19.922430 lo    In  IP (tos 0x0, ttl 64, id 30621, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34706 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0x4f24), ack 417, win 509, options [nop,nop,TS val 3517863257 ecr 3517863257], length 0
    09:58:19.922743 lo    In  IP (tos 0x0, ttl 64, id 30622, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34706 > 127.0.0.1.8080: Flags [F.], cksum 0xfe28 (incorrect -> 0x4f1f), seq 162, ack 417, win 512, options [nop,nop,TS val 3517863258 ecr 3517863257], length 0
    09:58:19.922812 lo    In  IP (tos 0x0, ttl 64, id 28765, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.8080 > 127.0.0.1.34706: Flags [F.], cksum 0xfe28 (incorrect -> 0x4f1d), seq 417, ack 163, win 512, options [nop,nop,TS val 3517863258 ecr 3517863258], length 0
    09:58:19.922815 lo    In  IP (tos 0x0, ttl 64, id 30623, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34706 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0x4f1d), ack 418, win 512, options [nop,nop,TS val 3517863258 ecr 3517863258], length 0
    09:58:19.923241 lo    In  IP (tos 0x0, ttl 64, id 50214, offset 0, flags [DF], proto TCP (6), length 60)
        127.0.0.1.34716 > 127.0.0.1.8080: Flags [S], cksum 0xfe30 (incorrect -> 0x83aa), seq 2191096712, win 65495, options [mss 65495,sackOK,TS val 3517863258 ecr 0,nop,wscale 7], length 0
    09:58:19.923244 lo    In  IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 60)
        127.0.0.1.8080 > 127.0.0.1.34716: Flags [S.], cksum 0xfe30 (incorrect -> 0x91a4), seq 2132233184, ack 2191096713, win 65483, options [mss 65495,sackOK,TS val 3517863258 ecr 3517863258,nop,wscale 7], length 0
    09:58:19.923247 lo    In  IP (tos 0x0, ttl 64, id 50215, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34716 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0xb860), ack 1, win 512, options [nop,nop,TS val 3517863258 ecr 3517863258], length 0
    09:58:19.923259 lo    In  IP (tos 0x0, ttl 64, id 50216, offset 0, flags [DF], proto TCP (6), length 254)
        127.0.0.1.34716 > 127.0.0.1.8080: Flags [P.], cksum 0xfef2 (incorrect -> 0x49ea), seq 1:203, ack 1, win 512, options [nop,nop,TS val 3517863258 ecr 3517863258], length 202: HTTP, length: 202
      PUT /v1/AUTH_test/containera HTTP/1.1
      Host: 127.0.0.1:8080
      Accept-Encoding: identity
      x-auth-token: AUTH_tk6ccb8979f39843b5b18d0bf4f3f23803
      content-length: 0
      user-agent: python-swiftclient-1.2.3

    09:58:19.923261 lo    In  IP (tos 0x0, ttl 64, id 51583, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.8080 > 127.0.0.1.34716: Flags [.], cksum 0xfe28 (incorrect -> 0xb797), ack 203, win 511, options [nop,nop,TS val 3517863258 ecr 3517863258], length 0
    09:58:19.986302 lo    In  IP (tos 0x0, ttl 64, id 51584, offset 0, flags [DF], proto TCP (6), length 358)
        127.0.0.1.8080 > 127.0.0.1.34716: Flags [P.], cksum 0xff5a (incorrect -> 0x5c98), seq 1:307, ack 203, win 512, options [nop,nop,TS val 3517863321 ecr 3517863258], length 306: HTTP, length: 306
      HTTP/1.1 202 Accepted
      Content-Type: text/html; charset=UTF-8
      Content-Length: 76
      X-Trans-Id: tx904e167fde594414bc426-0063b2aabb
      X-Openstack-Request-Id: tx904e167fde594414bc426-0063b2aabb
      Date: Mon, 02 Jan 2023 09:58:19 GMT

      <html><h1>Accepted</h1><p>The request is accepted for processing.</p></html> [|http]
    09:58:19.986310 lo    In  IP (tos 0x0, ttl 64, id 50217, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34716 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0xb5e8), ack 307, win 510, options [nop,nop,TS val 3517863321 ecr 3517863321], length 0
    09:58:19.988184 lo    In  IP (tos 0x0, ttl 64, id 57224, offset 0, flags [DF], proto TCP (6), length 60)
        127.0.0.1.34724 > 127.0.0.1.8080: Flags [S], cksum 0xfe30 (incorrect -> 0xdc55), seq 1239112530, win 65495, options [mss 65495,sackOK,TS val 3517863323 ecr 0,nop,wscale 7], length 0
    09:58:19.988188 lo    In  IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 60)
        127.0.0.1.8080 > 127.0.0.1.34724: Flags [S.], cksum 0xfe30 (incorrect -> 0xbcea), seq 2959689650, ack 1239112531, win 65483, options [mss 65495,sackOK,TS val 3517863323 ecr 3517863323,nop,wscale 7], length 0
    09:58:19.988190 lo    In  IP (tos 0x0, ttl 64, id 57225, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34724 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0xe3a6), ack 1, win 512, options [nop,nop,TS val 3517863323 ecr 3517863323], length 0
    09:58:19.988321 lo    In  IP (tos 0x0, ttl 64, id 15637, offset 0, flags [DF], proto TCP (6), length 60)
        127.0.0.1.34730 > 127.0.0.1.8080: Flags [S], cksum 0xfe30 (incorrect -> 0x7945), seq 923586859, win 65495, options [mss 65495,sackOK,TS val 3517863323 ecr 0,nop,wscale 7], length 0
    09:58:19.988324 lo    In  IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 60)
        127.0.0.1.8080 > 127.0.0.1.34730: Flags [S.], cksum 0xfe30 (incorrect -> 0x8fc7), seq 941263380, ack 923586860, win 65483, options [mss 65495,sackOK,TS val 3517863323 ecr 3517863323,nop,wscale 7], length 0
    09:58:19.988327 lo    In  IP (tos 0x0, ttl 64, id 15638, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34730 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0xb683), ack 1, win 512, options [nop,nop,TS val 3517863323 ecr 3517863323], length 0
    09:58:19.988439 lo    In  IP (tos 0x0, ttl 64, id 15639, offset 0, flags [DF], proto TCP (6), length 213)
        127.0.0.1.34730 > 127.0.0.1.8080: Flags [P.], cksum 0xfec9 (incorrect -> 0xb473), seq 1:162, ack 1, win 512, options [nop,nop,TS val 3517863323 ecr 3517863323], length 161: HTTP, length: 161
      GET /auth/v1.0 HTTP/1.1
      Host: 127.0.0.1:8080
      Accept-Encoding: identity
      x-auth-user: test:tester
      x-auth-key: testing
      user-agent: python-swiftclient-1.2.3

    09:58:19.988443 lo    In  IP (tos 0x0, ttl 64, id 54084, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.8080 > 127.0.0.1.34730: Flags [.], cksum 0xfe28 (incorrect -> 0xb5e3), ack 162, win 511, options [nop,nop,TS val 3517863323 ecr 3517863323], length 0
    09:58:19.988458 lo    In  IP (tos 0x0, ttl 64, id 57226, offset 0, flags [DF], proto TCP (6), length 213)
        127.0.0.1.34724 > 127.0.0.1.8080: Flags [P.], cksum 0xfec9 (incorrect -> 0xe196), seq 1:162, ack 1, win 512, options [nop,nop,TS val 3517863323 ecr 3517863323], length 161: HTTP, length: 161
      GET /auth/v1.0 HTTP/1.1
      Host: 127.0.0.1:8080
      Accept-Encoding: identity
      x-auth-user: test:tester
      x-auth-key: testing
      user-agent: python-swiftclient-1.2.3

    09:58:19.988462 lo    In  IP (tos 0x0, ttl 64, id 28143, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.8080 > 127.0.0.1.34724: Flags [.], cksum 0xfe28 (incorrect -> 0xe306), ack 162, win 511, options [nop,nop,TS val 3517863323 ecr 3517863323], length 0
    09:58:19.989381 lo    In  IP (tos 0x0, ttl 64, id 54085, offset 0, flags [DF], proto TCP (6), length 468)
        127.0.0.1.8080 > 127.0.0.1.34730: Flags [P.], cksum 0xffc8 (incorrect -> 0xe7e1), seq 1:417, ack 162, win 512, options [nop,nop,TS val 3517863324 ecr 3517863323], length 416: HTTP, length: 416
      HTTP/1.1 200 OK
      Content-Type: text/html; charset=UTF-8
      X-Auth-Token: AUTH_tk6ccb8979f39843b5b18d0bf4f3f23803
      X-Storage-Token: AUTH_tk6ccb8979f39843b5b18d0bf4f3f23803
      X-Auth-Token-Expires: 83891
      X-Storage-Url: http://127.0.0.1:8080/v1/AUTH_test
      Content-Length: 0
      X-Trans-Id: txb2cf7b58ae4641ec8bfde-0063b2aabb
      X-Openstack-Request-Id: txb2cf7b58ae4641ec8bfde-0063b2aabb
      Date: Mon, 02 Jan 2023 09:58:19 GMT

    09:58:19.989388 lo    In  IP (tos 0x0, ttl 64, id 15640, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34730 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0xb443), ack 417, win 509, options [nop,nop,TS val 3517863324 ecr 3517863324], length 0
    09:58:19.989618 lo    In  IP (tos 0x0, ttl 64, id 15641, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34730 > 127.0.0.1.8080: Flags [F.], cksum 0xfe28 (incorrect -> 0xb43f), seq 162, ack 417, win 512, options [nop,nop,TS val 3517863324 ecr 3517863324], length 0
    09:58:19.989784 lo    In  IP (tos 0x0, ttl 64, id 28144, offset 0, flags [DF], proto TCP (6), length 468)
        127.0.0.1.8080 > 127.0.0.1.34724: Flags [P.], cksum 0xffc8 (incorrect -> 0x30c3), seq 1:417, ack 162, win 512, options [nop,nop,TS val 3517863325 ecr 3517863323], length 416: HTTP, length: 416
      HTTP/1.1 200 OK
      Content-Type: text/html; charset=UTF-8
      X-Auth-Token: AUTH_tk6ccb8979f39843b5b18d0bf4f3f23803
      X-Storage-Token: AUTH_tk6ccb8979f39843b5b18d0bf4f3f23803
      X-Auth-Token-Expires: 83891
      X-Storage-Url: http://127.0.0.1:8080/v1/AUTH_test
      Content-Length: 0
      X-Trans-Id: tx4de11395969b4a9994474-0063b2aabb
      X-Openstack-Request-Id: tx4de11395969b4a9994474-0063b2aabb
      Date: Mon, 02 Jan 2023 09:58:19 GMT

    09:58:19.989793 lo    In  IP (tos 0x0, ttl 64, id 57227, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34724 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0xe164), ack 417, win 509, options [nop,nop,TS val 3517863325 ecr 3517863325], length 0
    09:58:19.989889 lo    In  IP (tos 0x0, ttl 64, id 54086, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.8080 > 127.0.0.1.34730: Flags [F.], cksum 0xfe28 (incorrect -> 0xb43d), seq 417, ack 163, win 512, options [nop,nop,TS val 3517863325 ecr 3517863324], length 0
    09:58:19.989892 lo    In  IP (tos 0x0, ttl 64, id 15642, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34730 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0xb43c), ack 418, win 512, options [nop,nop,TS val 3517863325 ecr 3517863325], length 0
    09:58:19.990325 lo    In  IP (tos 0x0, ttl 64, id 57228, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34724 > 127.0.0.1.8080: Flags [F.], cksum 0xfe28 (incorrect -> 0xe160), seq 162, ack 417, win 512, options [nop,nop,TS val 3517863325 ecr 3517863325], length 0
    09:58:19.990377 lo    In  IP (tos 0x0, ttl 64, id 28145, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.8080 > 127.0.0.1.34724: Flags [F.], cksum 0xfe28 (incorrect -> 0xe15f), seq 417, ack 163, win 512, options [nop,nop,TS val 3517863325 ecr 3517863325], length 0
    09:58:19.990379 lo    In  IP (tos 0x0, ttl 64, id 57229, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34724 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0xe15f), ack 418, win 512, options [nop,nop,TS val 3517863325 ecr 3517863325], length 0
    09:58:19.990848 lo    In  IP (tos 0x0, ttl 64, id 11154, offset 0, flags [DF], proto TCP (6), length 60)
        127.0.0.1.34742 > 127.0.0.1.8080: Flags [S], cksum 0xfe30 (incorrect -> 0x4998), seq 2251731359, win 65495, options [mss 65495,sackOK,TS val 3517863326 ecr 0,nop,wscale 7], length 0
    09:58:19.990851 lo    In  IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 60)
        127.0.0.1.8080 > 127.0.0.1.34742: Flags [S.], cksum 0xfe30 (incorrect -> 0x533a), seq 3858622733, ack 2251731360, win 65483, options [mss 65495,sackOK,TS val 3517863326 ecr 3517863326,nop,wscale 7], length 0
    09:58:19.990852 lo    In  IP (tos 0x0, ttl 64, id 44315, offset 0, flags [DF], proto TCP (6), length 60)
        127.0.0.1.34752 > 127.0.0.1.8080: Flags [S], cksum 0xfe30 (incorrect -> 0xb43e), seq 1562472452, win 65495, options [mss 65495,sackOK,TS val 3517863326 ecr 0,nop,wscale 7], length 0
    09:58:19.990854 lo    In  IP (tos 0x0, ttl 64, id 11155, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34742 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0x79f6), ack 1, win 512, options [nop,nop,TS val 3517863326 ecr 3517863326], length 0
    09:58:19.990855 lo    In  IP (tos 0x0, ttl 64, id 0, offset 0, flags [DF], proto TCP (6), length 60)
        127.0.0.1.8080 > 127.0.0.1.34752: Flags [S.], cksum 0xfe30 (incorrect -> 0x4fe7), seq 3559811286, ack 1562472453, win 65483, options [mss 65495,sackOK,TS val 3517863326 ecr 3517863326,nop,wscale 7], length 0
    09:58:19.990857 lo    In  IP (tos 0x0, ttl 64, id 44316, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34752 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0x76a3), ack 1, win 512, options [nop,nop,TS val 3517863326 ecr 3517863326], length 0
    09:58:19.990870 lo    In  IP (tos 0x0, ttl 64, id 11156, offset 0, flags [DF], proto TCP (6), length 243)
        127.0.0.1.34742 > 127.0.0.1.8080: Flags [P.], cksum 0xfee7 (incorrect -> 0x2944), seq 1:192, ack 1, win 512, options [nop,nop,TS val 3517863326 ecr 3517863326], length 191: HTTP, length: 191
      HEAD /v1/AUTH_test/containera/q1.txt HTTP/1.1
      Host: 127.0.0.1:8080
      Accept-Encoding: identity
      x-auth-token: AUTH_tk6ccb8979f39843b5b18d0bf4f3f23803
      user-agent: python-swiftclient-1.2.3

    09:58:19.990871 lo    In  IP (tos 0x0, ttl 64, id 28748, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.8080 > 127.0.0.1.34742: Flags [.], cksum 0xfe28 (incorrect -> 0x7938), ack 192, win 511, options [nop,nop,TS val 3517863326 ecr 3517863326], length 0
    09:58:19.990892 lo    In  IP (tos 0x0, ttl 64, id 44317, offset 0, flags [DF], proto TCP (6), length 243)
        127.0.0.1.34752 > 127.0.0.1.8080: Flags [P.], cksum 0xfee7 (incorrect -> 0x25f0), seq 1:192, ack 1, win 512, options [nop,nop,TS val 3517863326 ecr 3517863326], length 191: HTTP, length: 191
      HEAD /v1/AUTH_test/containera/q2.txt HTTP/1.1
      Host: 127.0.0.1:8080
      Accept-Encoding: identity
      x-auth-token: AUTH_tk6ccb8979f39843b5b18d0bf4f3f23803
      user-agent: python-swiftclient-1.2.3

    09:58:19.990894 lo    In  IP (tos 0x0, ttl 64, id 48252, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.8080 > 127.0.0.1.34752: Flags [.], cksum 0xfe28 (incorrect -> 0x75e5), ack 192, win 511, options [nop,nop,TS val 3517863326 ecr 3517863326], length 0
    09:58:20.010298 lo    In  IP (tos 0x0, ttl 64, id 48253, offset 0, flags [DF], proto TCP (6), length 282)
        127.0.0.1.8080 > 127.0.0.1.34752: Flags [P.], cksum 0xff0e (incorrect -> 0x0ed5), seq 1:231, ack 192, win 512, options [nop,nop,TS val 3517863345 ecr 3517863326], length 230: HTTP, length: 230
      HTTP/1.1 404 Not Found
      Content-Type: text/html; charset=UTF-8
      Content-Length: 0
      X-Trans-Id: tx06345d66a74d4803ba2ef-0063b2aabb
      X-Openstack-Request-Id: tx06345d66a74d4803ba2ef-0063b2aabb
      Date: Mon, 02 Jan 2023 09:58:20 GMT

    09:58:20.010306 lo    In  IP (tos 0x0, ttl 64, id 44318, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34752 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0x74d9), ack 231, win 511, options [nop,nop,TS val 3517863345 ecr 3517863345], length 0
    09:58:20.010901 lo    In  IP (tos 0x0, ttl 64, id 28749, offset 0, flags [DF], proto TCP (6), length 282)
        127.0.0.1.8080 > 127.0.0.1.34742: Flags [P.], cksum 0xff0e (incorrect -> 0x7615), seq 1:231, ack 192, win 512, options [nop,nop,TS val 3517863346 ecr 3517863326], length 230: HTTP, length: 230
      HTTP/1.1 404 Not Found
      Content-Type: text/html; charset=UTF-8
      Content-Length: 0
      X-Trans-Id: tx1e44d70df0f4421f96087-0063b2aabb
      X-Openstack-Request-Id: tx1e44d70df0f4421f96087-0063b2aabb
      Date: Mon, 02 Jan 2023 09:58:20 GMT

    09:58:20.010908 lo    In  IP (tos 0x0, ttl 64, id 11157, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34742 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0x782a), ack 231, win 511, options [nop,nop,TS val 3517863346 ecr 3517863346], length 0
    09:58:20.011196 lo    In  IP (tos 0x0, ttl 64, id 44319, offset 0, flags [DF], proto TCP (6), length 301)
        127.0.0.1.34752 > 127.0.0.1.8080: Flags [P.], cksum 0xff21 (incorrect -> 0x0c9b), seq 192:441, ack 231, win 512, options [nop,nop,TS val 3517863346 ecr 3517863345], length 249: HTTP, length: 249
      PUT /v1/AUTH_test/containera/q2.txt HTTP/1.1
      Host: 127.0.0.1:8080
      Accept-Encoding: identity
      x-object-meta-mtime: 1672653454.087197
      x-auth-token: AUTH_tk6ccb8979f39843b5b18d0bf4f3f23803
      content-length: 0
      user-agent: python-swiftclient-1.2.3

    09:58:20.011801 lo    In  IP (tos 0x0, ttl 64, id 11158, offset 0, flags [DF], proto TCP (6), length 301)
        127.0.0.1.34742 > 127.0.0.1.8080: Flags [P.], cksum 0xff21 (incorrect -> 0x15f0), seq 192:441, ack 231, win 512, options [nop,nop,TS val 3517863347 ecr 3517863346], length 249: HTTP, length: 249
      PUT /v1/AUTH_test/containera/q1.txt HTTP/1.1
      Host: 127.0.0.1:8080
      Accept-Encoding: identity
      x-object-meta-mtime: 1672653451.503279
      x-auth-token: AUTH_tk6ccb8979f39843b5b18d0bf4f3f23803
      content-length: 0
      user-agent: python-swiftclient-1.2.3

    09:58:20.054203 lo    In  IP (tos 0x0, ttl 64, id 28750, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.8080 > 127.0.0.1.34742: Flags [.], cksum 0xfe28 (incorrect -> 0x7704), ack 441, win 512, options [nop,nop,TS val 3517863389 ecr 3517863347], length 0
    09:58:20.057638 lo    In  IP (tos 0x0, ttl 64, id 48254, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.8080 > 127.0.0.1.34752: Flags [.], cksum 0xfe28 (incorrect -> 0x73ae), ack 441, win 512, options [nop,nop,TS val 3517863393 ecr 3517863346], length 0
    09:58:20.061757 lo    In  IP (tos 0x0, ttl 64, id 28751, offset 0, flags [DF], proto TCP (6), length 366)
        127.0.0.1.8080 > 127.0.0.1.34742: Flags [P.], cksum 0xff62 (incorrect -> 0x135d), seq 231:545, ack 441, win 512, options [nop,nop,TS val 3517863397 ecr 3517863347], length 314: HTTP, length: 314
      HTTP/1.1 201 Created
      Content-Type: text/html; charset=UTF-8
      Content-Length: 0
      Etag: d41d8cd98f00b204e9800998ecf8427e
      Last-Modified: Mon, 02 Jan 2023 09:58:21 GMT
      X-Trans-Id: tx84a18ea292d74bf4981e2-0063b2aabc
      X-Openstack-Request-Id: tx84a18ea292d74bf4981e2-0063b2aabc
      Date: Mon, 02 Jan 2023 09:58:20 GMT

    09:58:20.062306 lo    In  IP (tos 0x0, ttl 64, id 48255, offset 0, flags [DF], proto TCP (6), length 366)
        127.0.0.1.8080 > 127.0.0.1.34752: Flags [P.], cksum 0xff62 (incorrect -> 0xf004), seq 231:545, ack 441, win 512, options [nop,nop,TS val 3517863397 ecr 3517863346], length 314: HTTP, length: 314
      HTTP/1.1 201 Created
      Content-Type: text/html; charset=UTF-8
      Content-Length: 0
      Etag: d41d8cd98f00b204e9800998ecf8427e
      Last-Modified: Mon, 02 Jan 2023 09:58:21 GMT
      X-Trans-Id: txc6493a9756f8457abde98-0063b2aabc
      X-Openstack-Request-Id: txc6493a9756f8457abde98-0063b2aabc
      Date: Mon, 02 Jan 2023 09:58:20 GMT

    09:58:20.068267 lo    In  IP (tos 0x0, ttl 64, id 44320, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34752 > 127.0.0.1.8080: Flags [F.], cksum 0xfe28 (incorrect -> 0x7236), seq 441, ack 545, win 512, options [nop,nop,TS val 3517863403 ecr 3517863397], length 0
    09:58:20.068316 lo    In  IP (tos 0x0, ttl 64, id 50218, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34716 > 127.0.0.1.8080: Flags [F.], cksum 0xfe28 (incorrect -> 0xb593), seq 203, ack 307, win 512, options [nop,nop,TS val 3517863403 ecr 3517863321], length 0
    09:58:20.068331 lo    In  IP (tos 0x0, ttl 64, id 11159, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34742 > 127.0.0.1.8080: Flags [F.], cksum 0xfe28 (incorrect -> 0x7589), seq 441, ack 545, win 512, options [nop,nop,TS val 3517863403 ecr 3517863397], length 0
    09:58:20.068376 lo    In  IP (tos 0x0, ttl 64, id 48256, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.8080 > 127.0.0.1.34752: Flags [F.], cksum 0xfe28 (incorrect -> 0x722f), seq 545, ack 442, win 512, options [nop,nop,TS val 3517863403 ecr 3517863403], length 0
    09:58:20.068379 lo    In  IP (tos 0x0, ttl 64, id 44321, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34752 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0x722f), ack 546, win 512, options [nop,nop,TS val 3517863403 ecr 3517863403], length 0
    09:58:20.068461 lo    In  IP (tos 0x0, ttl 64, id 28752, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.8080 > 127.0.0.1.34742: Flags [F.], cksum 0xfe28 (incorrect -> 0x7582), seq 545, ack 442, win 512, options [nop,nop,TS val 3517863403 ecr 3517863403], length 0
    09:58:20.068466 lo    In  IP (tos 0x0, ttl 64, id 11160, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34742 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0x7582), ack 546, win 512, options [nop,nop,TS val 3517863403 ecr 3517863403], length 0
    09:58:20.068501 lo    In  IP (tos 0x0, ttl 64, id 51585, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.8080 > 127.0.0.1.34716: Flags [F.], cksum 0xfe28 (incorrect -> 0xb540), seq 307, ack 204, win 512, options [nop,nop,TS val 3517863403 ecr 3517863403], length 0
    09:58:20.068503 lo    In  IP (tos 0x0, ttl 64, id 50219, offset 0, flags [DF], proto TCP (6), length 52)
        127.0.0.1.34716 > 127.0.0.1.8080: Flags [.], cksum 0xfe28 (incorrect -> 0xb540), ack 308, win 512, options [nop,nop,TS val 3517863403 ecr 3517863403], length 0
    ```
  + Proxy server arrange proper components controller for request:
    ```python
        def handle_request(self, req):
        """
        Entry point for proxy server.
        ...
        """
        ...
            try:
                controller, path_parts = self.get_controller(req)
    ```
    ```python
        def get_controller(self, req):
        """
        Get the controller to handle a request.

        :param req: the request
        :returns: tuple of (controller class, path dictionary)

        :raises ValueError: (thrown by split_path) if given invalid path
        """
        if req.path == '/info':
            d = dict(version=None,
                     expose_info=self.expose_info,
                     disallowed_sections=self.disallowed_sections,
                     admin_key=self.admin_key)
            return InfoController, d

        version, account, container, obj = split_path(
            wsgi_to_str(req.path), 1, 4, True)
        d = dict(version=version,
                 account_name=account,
                 container_name=container,
                 object_name=obj)
        if account and not valid_api_version(version):
            raise APIVersionError('Invalid path')
        if obj and container and account:
            info = get_container_info(req.environ, self)
            if is_server_error(info.get('status')):
                raise HTTPServiceUnavailable(request=req)
            policy_index = req.headers.get('X-Backend-Storage-Policy-Index',
                                           info['storage_policy'])
            policy = POLICIES.get_by_index(policy_index)
            if not policy:
                # This indicates that a new policy has been created,
                # with rings, deployed, released (i.e. deprecated =
                # False), used by a client to create a container via
                # another proxy that was restarted after the policy
                # was released, and is now cached - all before this
                # worker was HUPed to stop accepting new
                # connections.  There should never be an "unknown"
                # index - but when there is - it's probably operator
                # error and hopefully temporary.
                raise HTTPServiceUnavailable('Unknown Storage Policy')
            return self.obj_controller_router[policy], d
        elif container and account:
            return ContainerController, d
        elif account and not container and not obj:
            return AccountController, d
        return None, d
    ```
  + Proxy server handle client adding objects (Policy: replication): Retrieve the object from storage nodes (Support multiple fragments downloads), send the data to client after object retreive completion.
    + Get all nodes by partition ID, and send object to nodes.
    ```python
    def PUT(self, req):
        """HTTP PUT request handler."""
        ...
        partition, nodes = obj_ring.get_nodes(
            self.account_name, self.container_name, self.object_name)

        ...
        # send object to storage nodes
        resp = self._store_object(
            req, data_source, nodes, partition, outgoing_headers)
        return resp
    ```
    + Filter enough nodes by write affinity, run multiple threads to store object
      ```python
          def _store_object(self, req, data_source, nodes, partition,
                      outgoing_headers):
            ...

            putters = self._get_put_connections(
                req, nodes, partition, outgoing_headers, policy)
            ...
      ```
      ```python
          def _get_put_connections(self, req, nodes, partition, outgoing_headers,
                             policy):
          ...
          node_iter = GreenthreadSafeIterator(
              self.iter_nodes_local_first(obj_ring, partition, policy=policy))
          pile = GreenPile(len(nodes))

          for nheaders in outgoing_headers:
              ...
              pile.spawn(self._connect_put_node, node_iter, partition,
                        req, nheaders, self.logger.thread_locals)

          ...
      ```
      ```python
          def iter_nodes_local_first(self, ring, partition, policy=None,
                               local_handoffs_first=False):
          ...
          policy_options = self.app.get_policy_options(policy)
          is_local = policy_options.write_affinity_is_local_fn
          ...
      ```
  + Proxy server handle client getting objects (Policy: Replication):
  ```python
      def GETorHEAD(self, req):
        ...
        partition = obj_ring.get_part(
            self.account_name, self.container_name, self.object_name)
        node_iter = self.app.iter_nodes(obj_ring, partition, self.logger,
                                        policy=policy)

        resp = self._get_or_head_response(req, node_iter, partition, policy)
        ...
  ```
  ```python
      def _get_or_head_response(self, req, node_iter, partition, policy):
            ...
            resp = self.GETorHEAD_base(
                req, 'Object', node_iter, partition,
                req.swift_entity_path, concurrency, policy)
            return resp
  ```
  ```python
      def GETorHEAD_base(self, req, server_type, node_iter, partition, path,
                       concurrency=1, policy=None, client_chunk_size=None):
        ...
        backend_headers = self.generate_request_headers(
            req, additional=req.headers)

        handler = GetOrHeadHandler(self.app, req, self.server_type, node_iter,
                                   partition, path, backend_headers,
                                   concurrency, policy=policy,
                                   client_chunk_size=client_chunk_size,
                                   logger=self.logger)
        res = handler.get_working_response(req)
        ...
  ```
  ```python
      def get_working_response(self, req):
            ...
            if source:
                res = Response(request=req)
                res.status = source.status
                update_headers(res, source.getheaders())
                if req.method == 'GET' and \
                        source.status in (HTTP_OK, HTTP_PARTIAL_CONTENT):
                    res.app_iter = self._make_app_iter(req, node, source)
                    ...
  ```
  ```python
      def _make_app_iter(self, req, node, source):
            """
            Returns an iterator over the contents of the source (via its read
            func).  There is also quite a bit of cleanup to ensure garbage
            collection works and the underlying socket of the source is closed.

            :param req: incoming request object
            :param source: The httplib.Response object this iterator should read
                          from.
            :param node: The node the source is reading from, for logging purposes.
            """

            ct = source.getheader('Content-Type')
            if ct:
                content_type, content_type_attrs = parse_content_type(ct)
                is_multipart = content_type == 'multipart/byteranges'
            else:
                is_multipart = False
            ...
            return document_iters_to_http_response_body(
                (add_content_type(pi) for pi in parts_iter),
                boundary, is_multipart, self.logger)
  ```
  ```python
      def response_parts_iter(self, req):
        try:
            self.source, self.node = next(self.source_and_node_iter)
        except StopIteration:
            return
        it = None
        if self.source:
            it = self._get_response_parts_iter(req)
        return it

    def _get_response_parts_iter(self, req):
        try:
            ...
            # This is safe; it sets up a generator but does not call next()
            # on it, so no IO is performed.
            parts_iter = [
                http_response_to_document_iters(
                    self.source, read_chunk_size=self.app.object_chunk_size)]
            ...
            def iter_bytes_from_response_part(part_file, nbytes):
                nchunks = 0
                buf = b''
                part_file = ByteCountEnforcer(part_file, nbytes)
                while True:
                    ...
                        if client_chunk_size is not None:
                            while len(buf) >= client_chunk_size:
                                client_chunk = buf[:client_chunk_size]
                                buf = buf[client_chunk_size:]
                                with WatchdogTimeout(self.app.watchdog,
                                                     self.app.client_timeout,
                                                     ChunkWriteTimeout):
                                    self.bytes_used_from_backend += \
                                        len(client_chunk)
                                    yield client_chunk
                    ...

            part_iter = None
            try:
                while True:
                    try:
                        start_byte, end_byte, length, headers, part = \
                            get_next_doc_part()
                    ...
                    byte_count = ((end_byte - start_byte + 1) - self.skip_bytes
                                  if (end_byte is not None
                                      and start_byte is not None)
                                  else None)
                    part_iter = iter_bytes_from_response_part(part, byte_count)
                    yield {'start_byte': start_byte, 'end_byte': end_byte,
                           'entity_length': length, 'headers': headers,
                           'part_iter': part_iter}
                    ...
  ```
  + Proxy server handle client getting objects (Policy: EC): Proxy server request multiple batches of bytes and send to client (Send out immediately once receive a batch), release CPU by sleep() to avoid starvation when every 5 chunks.
    ```python
      @ObjectControllerRouter.register(EC_POLICY)
      class ECObjectController(BaseObjectController):
          def _fragment_GET_request(
                  self, req, node_iter, partition, policy,
                  header_provider, logger_thread_locals):
              ...
              return (getter, getter.response_parts_iter(req))
    ```
    ```python
          def _get_response_parts_iter(self, req):
          try:
              ...
              parts_iter = [
                  http_response_to_document_iters(
                      self.source, read_chunk_size=self.app.object_chunk_size)]

              def get_next_doc_part():
                  while True:
                      try:
                          ...
                              start_byte, end_byte, length, headers, part = next(
                                  parts_iter[0])
                          return (start_byte, end_byte, length, headers, part)
                          ...
                          parts_iter[0] = http_response_to_document_iters(
                              new_source,
                              read_chunk_size=self.app.object_chunk_size)

              def iter_bytes_from_response_part(part_file, nbytes):
                nchunks = 0
                buf = b''
                part_file = ByteCountEnforcer(part_file, nbytes)
                while True:
                    try:
                        ...
                            parts_iter[0] = http_response_to_document_iters(
                                new_source,
                                read_chunk_size=self.app.object_chunk_size)
                        ...

                        # This is for fairness; if the network is outpacing
                        # the CPU, we'll always be able to read and write
                        # data without encountering an EWOULDBLOCK, and so
                        # eventlet will not switch greenthreads on its own.
                        # We do it manually so that clients don't starve.
                        ...
                        if nchunks % 5 == 0:
                            sleep()
              part_iter = None
              try:
                  while True:
                      try:
                          start_byte, end_byte, length, headers, part = \
                              get_next_doc_part()
                    ...
                      part_iter = iter_bytes_from_response_part(part, byte_count)
                      yield {'start_byte': start_byte, 'end_byte': end_byte,
                            'entity_length': length, 'headers': headers,
                            'part_iter': part_iter}
                      self.pop_range()
              finally:
                  if part_iter:
                      part_iter.close()

          ...
          finally:
              # Close-out the connection as best as possible.
              if getattr(self.source, 'swift_conn', None):
                  close_swift_conn(self.source)
    ```
  + `max_clients` option for wsgi server
  ```python
    def run_server(conf, logger, sock, global_conf=None, ready_callback=None,
               allow_modify_pipeline=True):
    ...
    max_clients = int(conf.get('max_clients', '1024'))
    pool = RestrictedGreenPool(size=max_clients)
    ...
    server_kwargs = {
        'custom_pool': pool,
        'protocol': protocol_class,
        'socket_timeout': float(conf.get('client_timeout') or 60),
        # Disable capitalizing headers in Eventlet. This is necessary for
        # the AWS SDK to work with s3api middleware (it needs an "ETag"
        # header; "Etag" just won't do).
        'capitalize_response_headers': False,
    }
    if ready_callback:
        ready_callback()
    try:
        wsgi.server(sock, app, wsgi_logger, **server_kwargs)
    except socket.error as err:
        if err.errno != errno.EINVAL:
            raise
    pool.waitall()
  ```
  ```python
  def server(sock, site,
           log=None,
           environ=None,
           max_size=None,
           max_http_version=DEFAULT_MAX_HTTP_VERSION,
           protocol=HttpProtocol,
           server_event=None,
           minimum_chunk_size=None,
           log_x_forwarded_for=True,
           custom_pool=None,
           keepalive=True,
           log_output=True,
           log_format=DEFAULT_LOG_FORMAT,
           url_length_limit=MAX_REQUEST_LINE,
           debug=True,
           socket_timeout=None,
           capitalize_response_headers=True):
    """
    ...
    :param custom_pool: A custom GreenPool instance which is used to spawn client green threads.
                If this is supplied, max_size is ignored.
    """
  ```
  + `concurrent_gets` for proxy server to retreive object: If the options is enabled, proxy server would create threads as the same amount as replica_count, wait for the first node complete request if inflight is greater than concurrency.
  ```python
    @ObjectControllerRouter.register(REPL_POLICY)
  class ReplicatedObjectController(BaseObjectController):
      def _get_or_head_response(self, req, node_iter, partition, policy):
          concurrency = self.app.get_object_ring(policy.idx).replica_count \
              if self.app.get_policy_options(policy).concurrent_gets else 1
          resp = self.GETorHEAD_base(
              req, 'Object', node_iter, partition,
              req.swift_entity_path, concurrency, policy)
          return resp
  ```
  ```python
      def GETorHEAD_base(self, req, server_type, node_iter, partition, path,
                       concurrency=1, policy=None, client_chunk_size=None):
        ...
        res = handler.get_working_response(req)
        ...
  ```
  ```python
      def get_working_response(self, req):
        source, node = self._get_source_and_node()
        ...
  ```
  ```python
   def _get_source_and_node(self):
        ...
        nodes = GreenthreadSafeIterator(self.node_iter)
        ...
        pile = GreenAsyncPile(self.concurrency)

        for node in nodes:
            pile.spawn(self._make_node_request, node, node_timeout,
                       self.logger.thread_locals)
            _timeout = self.app.get_policy_options(
                self.policy).concurrency_timeout \
                if pile.inflight < self.concurrency else None
            if pile.waitfirst(_timeout):
                break
        else:
            # ran out of nodes, see if any stragglers will finish
            any(pile)
        ...
  ```
  + `eventlet_debug` for wsgi server: eventlet.debug.hub_exceptions: Toggles whether the hub prints exceptions that are raised from its timers. This can be useful to see how greenthreads are terminating.
  ```python
  def run_server(conf, logger, sock, global_conf=None, ready_callback=None,
               allow_modify_pipeline=True):
    ...
    eventlet_debug = config_true_value(conf.get('eventlet_debug', 'no'))
    eventlet.debug.hub_exceptions(eventlet_debug)
  ```
  + Authentication before any API request:
    + Before do any API request, swift client would request proxy server `GET /auth/v1.0` with headers `{'X-Auth-User', 'X-Auth-Key'}`
    ```python
    # Python-swiftclient/swiftclient/client.py
    def get_auth_1_0(url, user, key, snet, **kwargs):
      ...
      method = 'GET'
      headers = {'X-Auth-User': user, 'X-Auth-Key': key}
      ...
    ```
    + Return 401 unauthorized in middleware if account is suspicious, otherwise response `X-Auth-Token`
    ```python
        def get_controller(self, req):
        """
        Get the controller to handle a request.
        ...
        """
        ...
        if obj and container and account:
            info = get_container_info(req.environ, self)
            if is_server_error(info.get('status')):
                raise HTTPServiceUnavailable(request=req)
            policy_index = req.headers.get('X-Backend-Storage-Policy-Index',
                                           info['storage_policy'])
            policy = POLICIES.get_by_index(policy_index)
            if not policy:
                raise HTTPServiceUnavailable('Unknown Storage Policy')
            return self.obj_controller_router[policy], d
        elif container and account:
            return ContainerController, d
        elif account and not container and not obj:
            return AccountController, d
        return None, d
    ```
    + Swift client request the desired API, with header `X-Auth-Token`
      
       
