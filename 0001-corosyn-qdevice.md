1. Vocabulary

    corosync-qdevice: Daemon (not part of corosync process) running on every node in cluster. This one connects to votequorum API. Provides heartbeat to votequorum. Itself does nothing, needs plugin.
    corosync-qnetd: QDevice Network Daemon runs only on one node (at the beginning, make it clusterable in future). It is able to provide arbiter function for multiple clusters.
    Heuristic: "Script" running on every node. If it fails for some amount of runs, quorum is lost.
    Votequorum: Main corosync quorum provider and API. QDevice calls Votequorum API

2. Calculations

    Qdevice no votes = number of nodes - 1. For example, 5 nodes + Qdevice = 5 + 4 = 9 votes

3. Decision algorithms

    0 - Test - Vote is given by every client who asks for it
    1 - FFSplit - 50:50 Split
    2 - 2nodeLMS - Last man standing algorithm for 2 node cluster use case
    3 - LMS - Last man standing

actually there are also test and 2nodelms , both of which are mainly for developers and shouldn't be used for production clusters).
For a description of what each algorithm means and how the algorithms differ see their individual sections.  Default value is ffsplit.

       ffsplit
              This  one makes sense only for clusters with even number of nodes. It provides exactly one vote to the partition with the highest number of active nodes. If there are two exactly similar par-
              titions, it provides its vote to the partition that has the most clients connected to the qnetd server. If this number is also equal, then the tie_breaker is used. It is  able  to  transition
              its vote if the currently active partition becomes partitioned and a non-active partition still has at least 50% of the active nodes. Because of this, a vote is not provided if the qnetd con-
              nection is not active.

              To use this algorithm it's required to set the number of votes per node to 1 (default) and the qdevice number of votes has to be also 1. This is achieved by setting quorum.device.votes key in
              corosync.conf file to 1.

       lms    Last-man-standing. If the node is the only one left in the cluster that can see the qnetd server then we return a vote.

              If more than one node can see the qnetd server but some nodes can't see each other then the cluster is divided up into 'partitions' based on their ring_id and this algorithm returns a vote to
              the largest active partition or, if there is more than 1 equal partiton, the partition that contains the tie_breaker node (lowest, highest, etc). For LMS to work, the number of qdevice  votes
              has to be set to default (so just delete quorum.device.votes key from corosync.conf).

in this documentaion, I just have two node to run ha stack, one node to run corosync-qnetd

4. configuration without tls
1) corosync-qnetd is not part of cluster, you should setup it on a indepent server/PC
edit /etc/sysconfig/corosync-qnetd

COROSYNC_QNETD_OPTIONS="-4 -l ${IPADDR_TO_LISTEN} -p ${PORT_TO_LISTEN} -s off"

2) configuration for corosync-qdevice:
edit /etc/corosync/corosync.conf

quorum {
    #votequorum requires an expected_votes value to function
    expected_votes: 3 # expected_votes is "votes-of-nodes" + "votes-of-qdevice", while "votes-of-qdevice" is "votes-of-nodes" - 1
    #Disables two node cluster operations
    #two_node: 1 # two_node is conflict with qdevice
    #Enable and configure quorum subsystem
    provider: corosync_votequorum
    #these lines are for qdevice
    device {
        model: net
        timeout: 10000
        sync_timeout: 20000
        votes: 1
        net {
            host: 192.168.122.49
            port: 5403 # default is 5403
            tls: off
            algorithm: ffsplit
            tie_breaker: lowest
        }
    }
}

3) start corosync-qnetd on NETD server
# systemctl start corosync-qnetd

4) start pacemaker and corosync-qdevice on every cluster node
# systemctl start pacemaker
# systemctl start corosync-qdevice

5. configuration with tls
1) corosync-qnetd is not part of cluster, you should setup it on a indepent server/PC
edit /etc/sysconfig/corosync-qnetd

COROSYNC_QNETD_OPTIONS="-4 -l ${IPADDR_TO_LISTEN} -p ${PORT_TO_LISTEN} -s on" # by default tls is enabled

2) edit /etc/corosync/corosync.conf the same as section 4

3) Initialize database on QNetd server by running corosync-qnetd-certutil -i
# corosync-qnetd-certutil -i

4) Copy exported QNetd CA certificate (/etc/corosync/qnetd/nssdb/qnetd-cacert.crt) to every node
# scp /etc/corosync/qnetd/nssdb/qnetd-cacert.crt ${NODE}$i:/etc/corosync/qnetd/nssdb/qnetd-cacert.crt

5) On one of cluster node initialize database by running /usr/sbin/corosync-qdevice-net-certutil -i -c qnetd-cacert.crt
# /usr/sbin/corosync-qdevice-net-certutil -i -c /etc/corosync/qnetd/nssdb/qnetd-cacert.crt

6) still on the node in 3)
Generate certificate request: /usr/sbin/corosync-qdevice-net-certutil -r -n Cluster (Cluster name must match cluster_name key in the corosync.conf)
# /usr/sbin/corosync-qdevice-net-certutil -r -n hacluster

7) Copy exported CRQ to QNetd server
# scp /etc/corosync/qdevice/net/nssdb/qdevice-net-node.crq ${QNETD_SERVER}:/etc/corosync/qnetd/nssdb/

8) On QNetd server sign and export cluster certificate by running corosync-qnetd-certutil -s -c qdevice-net-node.crq -n Cluster
# corosync-qnetd-certutil -s -c /etc/corosync/qnetd/nssdb/qdevice-net-node.crq -n hacluster

9) Copy exported CRT to node where certificate request was created(the node in step 3)
scp /etc/corosync/qnetd/nssdb/cluster-hacluster.crt ${NODE}:/etc/corosync/qdevice/net/nssdb/cluster-hacluster.crt

10) Import certificate on node where certificate request was created by running /usr/sbin/corosync-qdevice-net-certutil -M -c cluster-Cluster.crt
# usr/sbin/corosync-qdevice-net-certutil -M -c /etc/corosync/qdevice/net/nssdb/cluster-hacluster.crt

11) Copy output qdevice-net-node.p12 to all other cluster nodes
# scp /etc/corosyc/qdevice/net/nssdb/qdevice-net-node.p12 ${NODE}$i:/etc/corosyc/qdevice/net/nssdb/qdevice-net-node.p12

12) - On all other nodes in cluster:
  - Init database by running /usr/sbin/corosync-qdevice-net-certutil -i -c qnetd-cacert.crt
  # /usr/sbin/corosync-qdevice-net-certutil -i -c /etc/corosync/qnetd/nssdb/qnetd-cacert.crt
  - Import cluster certificate and key: /usr/sbin/corosync-qdevice-net-certutil -m -c qdevice-net-node.p12
  # /usr/sbin/corosync-qdevice-net-certutil -m -c /etc/corosyc/qdevice/net/nssdb/qdevice-net-node.p12

13) start corosync-qnetd on netd server:
# systemctl start corosync-qnetd

14) start pacemaker and corosync-qdevice on every cluster node:
# systemctl start pacemaker
# systemctl start corosync-qdevice
