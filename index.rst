
:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::


Introduction
============

Previous round of APDB tests with Apache Cassandra (`DMTN-156`_) performed
with a three-node cluster at NCSA showed promising results but it was not
possible to determine scaling behavior from those tests or estimate the size
of the cluster needed for production scale. New series of tests is needed to
measure performance as a function of cluster size and resolve the issues that
were outlined in `DMTN-156`_.


Hardware in previous test
=========================

This table copied from `DMTN-156`_ summarizes hardware specs for machines that
were used in previous tests:

+----------+---------------------------+----------------------------+
|          | master02                  | master03,04                |
+==========+===========================+============================+
| CPU      | 2 x Intel Xeon Gold 5120; | 2 x Intel Xeon Gold 5218;  |
|          | 2 x 14 cores              | 2 x 16 cores               |
+----------+---------------------------+----------------------------+
| RAM      | 256 GiB                   | 256 GiB                    |
+----------+---------------------------+----------------------------+
| Storage  | 12 x 480 GB SATA SSD;     | 5 x 4000 GB NVMe SSD       |
|          | RAID controller           |                            |
+----------+---------------------------+----------------------------+

Important distinction between them was that one of the machines had SATA disks
connected to RAID controller and two other more modern hosts used NVMe disks.
The tests showed that the machine with old SATA disks was a reason for
bottlenecks in response time, likely caused by limited parallelism of SATA and
RAID interfaces and resulting increased I/O latency and reduced IOPS rate.


Storage requirements
====================

Total data size grows linearly with the number of visits. `DMTN-156`_
estimates growth rate at 2TB per 100k visits per replica. Reasonable number of
replicas is three, maximum number of visits that we can generate is limited by
the time budget. Optimally we would like to generate a year worth of data but
6 month of data, or even shorter periods, may be useful if we need to test
different parameters, e.g. performance as a function of cluster size. One year
corresponds approximately to 400k visits or 24TB of data with three replicas.
Per-node data size depends on number of nodes in a cluster, with 10 nodes it
makes approximately 2.4TB of data per node, with 5 nodes - 4.8TB. Significant
overhead is needed for storage so that Cassandra can run its data compaction
algorithm which temporarily creates two copies of the compacted data.

Probably largest impact on Cassandra performance comes from data access
latency and concurrency. In that respect locally-attached NVMe storage is an
optimal approach. In previous tests faster storage supported 1M accumulated
IOPS, while slower machine could only provide 15k IOPS and that caused
performance degradation. It is not clear whether network-attached storage
could provide necessary latency and parallelism, likely comparable performance
is not achievable in that case.


Cluster size
============

Cassandra performance should scale linearly with the number of nodes in
Cassandra cluster. Main goal of the next series of tests is to determine
optimal cluster size which provides adequate performance within a budget. With
three replicas minimum cluster size is also three. To understand scaling
behavior we need to run tests with varying cluster size, reasonable guess for
that could be 3, 6, or 9 nodes, possibly 12 if we suspect non-linear scaling
behavior during tests. For this sort of tests we do not need to make full
1-year dataset, 3 or 6 months worth of data can be sufficient to estimate
resulting behavior.


Memory size
===========

In previous tests machines had large large 256GB RAM. In those tests we
learned that Cassandra, being a Java application, needs careful tuning for
memory parameters to reduce garbage collection overhead. In general Cassandra
should prefer large number of nodes with smaller memory size per node.
Optimal memory size could be around 64GB or less per node.


Clients and Connectivity
========================

We need to run tests in a realistic setup with 189 independent client
processes (`ap_proto`). Clients do not require significant resources but to
avoid contention on client side it may be better to run single client process
per CPU core utilizing multiple physical hosts. Obviously clients need to run
on machines which are separate from Cassandra cluster itself. Memory size is
not important for clients. Our test uses MPI as coordination mechanism between
clients, ideally we need support for MPI on hosts that are running client
software.

Each client process can connect to any number of Cassandra nodes. To optimally
spread the load between nodes all of the nodes in the cluster should be
accessible from every client.


Test time
=========

In test with three node typical time to generate 6 months of data (180k
visits) was about 10 days of uninterrupted running. Anticipated time with
larger cluster should be shorter but this obviously depends on scaling
behavior that we are yet to learn.


Proposed plan
=============

After some effort needed to setup both Cassandra nad client software the
reasonable plan for testing could be:

- Several short tests may be needed to understand and optimize performance
  of I/O subsystem.
- Study scaling behavior with cluster of different size, e.g. 3, 6, 9 nodes.
  For each such test 3 to 6 month of data could be enough for reasonable
  estimate. Total time to run this tests could be 3-4 weeks if there are no
  unexpected delays.
- Longer run with 6-12 months of data with optimal cluster size to understand
  scaling with the amount of data, this could take couple of weeks or shorter
  if we see an expected increase in performance.


.. _DMTN-156: https://dmtn-156.lsst.io/
