---
layout: draft
title: Lessons in ZFS
subtitle: And one ring to bind them…
tags: filesystem os
---

Zetabyte File System  never overwrites existing data which allows for snapshots (readonly) and clones (read/write) to perform without 
much (or any) impact or overhead.

ZFS maintains clean state always. Makes good use of multicore processors.

#### Checksums detect

- Bit rot on disks
- Phantom writes
- Misdirected read/writes
- DMA parity errors
- Bugs in disk drivers and disk firmware
- Accidental overwrite of data
- Verification of reconstructed data (can be used with RAIDZ)
[Support](https://www.freebsdfoundation.org/blog/openzfs-raid-z-online-expansion-project-announcement/) 
for online expansion of RAIDZ coming to FreeBSD soon.

### BFS Architecture
UFS  - FS NS MGMT
VM/page - Cache MGMT
FFS - Fast File System
GEOM - VOL MGMT
appear as fixed partitions

### ZFS Architecture
Filesystem Namespace mgmt (Similar to UFS):
ZFS POSIX LAYER (ZPL), ZFS Attribute Processor (ZAP)
Filesystem Storage mgmt (Similar to FFS):
FS Data Mgmt (DMU), ZFS Intent Log (ZIL) 
Dataset & Snapshot layer

#### Cache
* Adaptive replacement cache (ARC) similar to VM/page
* L2ARC for slow access store similar to VM/Swap area

#### POSX
ZFS is a POSIX filesystem with features similar to UFS:
Offers NFSv4/SMB functionality such as case–insensitivity, and 
unicode normilization.


_ZFS uses more memory to reduce io operations at the cost of more memory (100k records ~> 200 MB compared to 50MB used by UFS) and
can utilize up to 1.6GB of memory._

ZFS logging with ZIL (Intent Logging): write/symlink/permissions

Sync store on log write completes an `fsync()` operation that 
requires stable storage before returning 
(32KB and higher, unless tuned for smaller with `logbias`)

_ZAP objects_ are key/value pair that map object id's to dnode's (like inodes for UFS) which are a pointer to the object referenced on disk.
ZFS Block pointers are 128B structures that can point to as many as 3 [copies of a block] different disks which consist of:
 disk_id:block_size:checksum

ZFS checksums every block it is managing

Each block pointer has three sizes associated with it:
lsize - logical size of the block
psize - physical size of block (maybe smaller than logical size if has been compressed)
asize - allocated size on disk (including RAIDZ parity and gang–block)


12K direct blocks for pointer of objects less than 128KB
16K indirect block can hold up to 128 pointers for objects greater than 128KB

Database that read/writes widely separated 4KB records the filesystem can be configured to allocate 4KB 
blocks to reduce the need to copy an entire 128KB block when only 4K has been modified.




#### Features  
Variable block sizes
Architecture independant on–disk format
Pooled storage zpools
Disk–level redundancy through mirroring and parity RAID(3x)
Hybrid pool support using using SSD's to cache reads and NVRAM to accelerate synchronous writes.
Intelligent prefetches multiple streams per file plus auto–detected stride patterns.
quotas
Admin features integrated with NFS shares
Fast remote replication and backups (?? Probably because of the prefetch and hybrid pool support ??)
iscsi target for zvol


- Deduplication is available if you select a "dedupe–capable" checksum (i.e. SHA256)

- A "volume" is similar to a block device, and can be loopback-mounted with a filesystem of any type, or perhaps presented as an iscsi target.


- "scrub" for integrity check of blocks used on vdev checksums

- performance degradation may occur for heavily used datasets at about 50% capacity. occurs at 80%+ capacity


ZFS tracks its used/available blocks with space maps, birth time and dead lists so that it never
has to traverse its space map when creating or freeing a snapshot , clone or ZVOL.

Blocks released when object in filesystem is overwritten, truncated or deleted unless the object is still being
referenced by a snapshot, at which point it is added to the deadlist.

## Deduplication
hash functions use SHA256 for accuracy or fletcher4 for performance (collision–prone).

write throughput may slow dramatically when hash–to–block ratio too high as the mapping table in memory becomes too full.
limit filesystem count
dedupe volumes which are likely to have many duplicate blocks.
perhaps block size could play a role in optimizing our mapping table

## Backup/Replication
ZFS can dump/restore stream of data from entire volume locally and remotely using the DMU which understands how
to traverse through different types of data structures similar to UFS. ZFS refers to these operations as send()/recv().

ZFS may choose to replicate an entire snapshot (full backup) or only the differences between two snapshots (Incremental backup).
