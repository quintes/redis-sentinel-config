Redis sentinel configuration and usage
===========

My config and setup of a master and 2 slave redis cluster using redis sentinel

## License

    This program is free software: you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License
    along with this program.  If not, see <http://www.gnu.org/licenses/>.

## Usage

Run the Master

  redis-server $PWD/master/master-redis.conf

Test
  redis-cli -p 6379
  set bob 123
  get bob

Run the main sentinel

  redis-server $PWD/master/sentinel.conf --sentinel

  you will see the sentinel monitor
        +monitor master redis-local-cluster 127.0.0.1 6379 quorum 2

Run Slave 1

  redis-server $PWD/slave1/s1-redis.conf
  We can tell it is a slave:
        MASTER <-> SLAVE sync: Finished with success

Run Slave 1 Sentinel

 redis-server $PWD/slave1/s1-sentinel.conf --sentinel

 Connecting and working
        +monitor master redis-local-cluster 127.0.0.1 6379 quorum 2

Run Slave 2

 redis-server $PWD/slave2/s2-redis.conf

 Connecting and working
      MASTER <-> SLAVE sync: Finished with success

Run Slave 2 Sentinel

 redis-server $PWD/slave2/s2-sentinel.conf --sentinel

 Connecting and working
        +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ redis-local-cluster 127.0.0.1 6379
        +slave slave 127.0.0.1:6381 127.0.0.1 6381 @ redis-local-cluster 127.0.0.1 6379
        +sentinel sentinel 01105752f94e78d38cab212f30a53ad139e09979 127.0.0.1 26379 @ redis-local-cluster 127.0.0.1 6379
        +sentinel sentinel 9348999cde7c3aa05dda31746e95f0fb964e1055 127.0.0.1 26380 @ redis-local-cluster 127.0.0.1 6379

Test using the redis-cli

  Connect to the main sentinel:

  redis-cli -p 26379
  127.0.0.1:26379> INFO [section]

  Returns:
    # Sentinel
    sentinel_masters:1
    sentinel_tilt:0
    sentinel_running_scripts:0
    sentinel_scripts_queue_length:0
    sentinel_simulate_failure_flags:0
    master0:name=redis-local-cluster,status=ok,address=127.0.0.1:6379,slaves=2,sentinels=3

  Figure out who is the master, asking slave 2
    redis-cli -p 26381 sentinel get-master-addr-by-name redis-local-cluster
  Returns, as expected:
    1) "127.0.0.1"
    2) "6379"

Test the slaves, ensure slaves are syncing
  Connect to the master redis

  redis-cli -p 6379
    127.0.0.1:6379> set mykey 123
    OK

  disconnect and connect to the first slave:

  redis-cli -p 6380
    127.0.0.1:6380> get mykey
    "123"
    127.0.0.1:6380>

Test the failover

  Kill the instance of redis which was started first

  Taking a look at the first sentinel output, we can see the redis instance went down
  Directly thereafter they all agreed on who to make the master

    +sdown master redis-local-cluster 127.0.0.1 6379
    +new-epoch 2
    +vote-for-leader 96f6cfd4193108b326fb15d1adbfb2ecc630ff97 2
    +config-update-from sentinel 96f6cfd4193108b326fb15d1adbfb2ecc630ff97 127.0.0.1 26381 @ redis-local-cluster 127.0.0.1 6379
    +switch-master redis-local-cluster 127.0.0.1 6379 127.0.0.1 6381

  The new master is on port 6381, lets try connect
  First check who the sentinel thinks is the master, ask either of the slave sentinels:

  redis-cli -p 26380 sentinel get-master-addr-by-name redis-local-cluster
    1) "127.0.0.1"
    2) "6381"

  Agreement, so lets see if we can set a key on what was previously a slave:

  redis-cli -p 6381
    127.0.0.1:6381> get mykey
    "123"
    127.0.0.1:6381> set mykey newhere
    OK

  Did it sync to 6380?
  redis-cli -p 6380
    127.0.0.1:6380> get mykey
    "newhere"
    127.0.0.1:6380>

  Performing an INFO on the main sentinel (which actually could've gone down too ) indicates that it too knows who is the new master
    # Sentinel
      sentinel_masters:1
      sentinel_tilt:0
      sentinel_running_scripts:0
      sentinel_scripts_queue_length:0
      sentinel_simulate_failure_flags:0
      master0:name=redis-local-cluster,status=ok,address=127.0.0.1:6381,slaves=2,sentinels=3

  All good.

End of document.
