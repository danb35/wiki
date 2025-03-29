---
title: Special VDEVs
description: What they are and how to use them
published: true
date: 2024-11-14T00:14:58.481Z
tags: 
editor: markdown
dateCreated: 2024-05-24T16:06:27.962Z
---

The page below was written by user [Constantin](https://forums.truenas.com/u/Constantin) on the TrueNAS forums and is used with his gracious permission:

# Why consider a sVDEV? 
A sVDEV has the potential to significantly speed up the operations of a HDD pool by putting small files and pool metadata into a faster special VDEV (sVDEV), which usually consists of SSDs. You can also designate specific datasets to reside on a sVDEV by setting record size in the dataset configuration page to equal the sVDEV small file cutoff. Thus, the common practice of segregating pools by hard drive type (SSD vs. HDD) is no longer necessary.   

## What about L2ARC?
The main difference between Second Level Adaptive Replacement Cache (L2ARC) and sVDEV is that while L2ARC can cache frequently-accessed files, or metadata, or files *and* metadata, it is *read-only*, *redundant*, and it also needs to get "hot".  Unlike sVDEV, the L2ARC has to "miss" having a file or metadata in cache before the miss is noted and *might* be read into the L2ARC for later reuse. L2ARC also requires RAM to function, so the common recommendation here is not to enable L2ARC unless you have a minimum 64GB of RAM available to TrueNAS.  

[Based on my limited testing, it took my NAS about three passes with rsync for a metadata=only L2ARC to cache all the metadata it needed.](https://www.truenas.com/community/threads/l2arc-impact-on-rsync-performance-for-largely-dormant-data.75066/). As you can see from my test results, using L2ARC solely for metadata sped up the rsync task significantly. For an in-depth discussion of L2ARC, see this [TrueNAS documentation hub page](https://www.truenas.com/docs/references/l2arc/). Happily, L2ARC can be made *persistent*, allowing a "hot" cache to survive reboots. Because L2ARC is redundant, any SSD will do, as long as it's read performance is decent. 

Unlike L2ARC, the sVDEV doesn't need more RAM and it's 100% hot from the start since all metadata will reside on it. Additionally, the sVDEV will also host small files, which are the worst-performing category for HDDs to deal with. Unlike the L2ARC, your pool will die if the sVDEV stops functioning. For a L2ARC-sVDEV comparison, [see my results from a few years ago comparing the impact of L2ARC and sVDEVs in my rsync use case.](https://www.truenas.com/community/threads/impact-of-svdev-on-rsync-vs-l2arc.93371/)

## About record sizes and small file cutoffs…
What constitutes a small file depends in part on the recordsize used in your pool / dataset. By default, TrueNAS uses 128k recordsizes but you can adjust them on a per-dataset basis, if you wish (See Storage/Pool/Dataset triple-dot on the right of the GUI). Larger record sizes (up to 1M) used in conjunction with compression are great for media files. Smaller record sizes are helpful for databases and like use cases. 

For simple large-file data transfers, adjusting the record size up to 1M with zstd compression and adding the sVDEV, made transfer speeds to my pool jump from about 250MB/s to 400MB/s. Not bad for a single-VDEV Z3 pool. 

**Important: the pool depends on the sVDEV to function, so if the sVDEV dies, your pool dies also. Take the same care with sVDEVs re: redundancy and resilience as you did with the other VDEVs! Do not use crummy SSDs for the sVDEV unless you don't care about pool life expectancy.**. 

# Planning
Ideally, a special VDEV (sVDEV) is planned carefully in advance and is added to the pool before the pool is populated with data. In order to get moved to the sVDEV, a small file has to be copied into the pool or rebalanced in-situ, so figuring out how much storage the sVDEV of a new pool needs in advance is better than adding one later and rebalancing everything just to migrate small files into the sVDEV. 

Similarly, you will want to adjust the record size of each data set ideally in advance to figure out what the best balance between record size and sVDEV use is. The good news is that sVDEV cutoffs can be adjusted by dataset, so there is plenty of flexibility. 

This feature is also interesting in that you can make datasets that use the same small file cutoff as the recordsize, ensuring that the entire dataset sits on flash drives only. Thus, a sVDEV can selectively host **all** data that needs speed (databases, metadata, etc.) all as part of a common pool vs. setting up two pools (one flash, the other HDD). 

## Required sVDEV Size
The needed sVDEV size has be determined by how much metadata and small files data needs to be held. By default, the sVDEV features a 25/75% partition split between metadata and small files, respectively. It can be adjusted by adjusting the `zfs_special_class_metadata_reserve_pct` parameter (thank you, @HoneyBadger!). The sVDEV does not like to be filled more than 75% by default and you also have to allow for pool expansion in the future. Some extrapolation is likely needed. 

### Determining Metadata Space Needs
There is a rule of thumb that metadata consumes about 0.3% of the pool space but this will vary by use case. NAS' hosting only large video files will have comparatively little metadata compared to someone who makes a non-archived copy of a MacOS drive. If you have the time, there are CLI commands to allow the user to determine the size of the metadata exactly. In TrueNAS Core as of 13.0U6, the command to determine your metadata space needs is 


> zdb -LbbbA -U /data/zfs/zpool.cache *poolname*
 
That will spit out a series of tables. One will be long... so long in fact that I've truncated mine to only show the top and the bottom while cutting out the middle.

```

Blocks  LSIZE   PSIZE   ASIZE     avg    comp   %Total  Type
     -      -       -       -       -       -        -  unallocated
     2    32K      8K     24K     12K    4.00     0.00  object directory
                            .... SNIP! .....
 89.0K  2.80G    359M    718M   8.06K    8.00     0.00      L2 Total
 1.61M  51.6G   11.6G   23.2G   14.4K    4.44     0.08      L1 Total
  140M  17.3T   17.2T   30.1T    220K    1.01    99.92      L0 Total
  142M  17.4T   17.2T   30.1T    218K    1.01   100.00  Total
```

Draw your attention to the line with L1 total, i.e. three lines up from the bottom. There is our metadata information i.e. 23.2GB or 0.08% of my pool was metadata.  Extrapolating across 50TB of pool capacity in my use case, that comes out to about 40GB of metadata needs. 

Once TrueNAS transitions to OpenZFS 2.2.0 or higher, the current metadata space needs will be even easier to determine via *zdb -bbbs* with a nice summary at the bottom, see [here](https://github.com/openzfs/zfs/discussions/14542).  Now you know one minimum. 

### Small File Space Needs
The *zdb -LbbbA -U /data/zfs/zpool.cache poolname* command will also spit out the answer for your small file needs by publishing something called the "Block Size Histogram". It summarizes the block sizes used, i.e. how many of each type can be found in your pool today.

```
Block Size Histogram

  block   psize                  lsize                asize
   size   Count   Size      Cum.  Count   Size   Cum.  Count   Size   Cum.
    512:   422K   211M   211M   422K   211M   211M      0      0      0
     1K:   113K   123M   334M   113K   123M   334M      0      0      0
     2K:  65.7K   175M   510M  65.7K   175M   510M      0      0      0
     4K:  1.92M  7.71G  8.21G   119K   554M  1.04G  1.01M  4.04G  4.04G
     8K:   383K  3.56G  11.8G  39.6K   436M  1.46G  1.77M  14.6G  18.6G
    16K:   765K  13.2G  25.0G   316K  5.18G  6.64G   297K  5.87G  24.5G
    32K:   368K  16.5G  41.5G  1.80M  59.3G  65.9G   585K  18.8G  43.4G
    64K:   622K  56.3G  97.8G  34.0K  2.73G  68.6G   386K  32.4G  75.8G
   128K:   137M  17.1T  17.2T   139M  17.3T  17.4T   138M  30.0T  30.1T
   256K:      0      0  17.2T      0      0  17.4T    605   164M  30.1T
   512K:      0      0  17.2T      0      0  17.4T      0      0  30.1T
     1M:      0      0  17.2T      0      0  17.4T      0      0  30.1T
     2M:      0      0  17.2T      0      0  17.4T      0      0  30.1T
     4M:      0      0  17.2T      0      0  17.4T      0      0  30.1T
     8M:      0      0  17.2T      0      0  17.4T      0      0  30.1T
    16M:      0      0  17.2T      0      0  17.4T      0      0  30.1T
```
Pay attention to the column "Cum." to the right of the table. That shows how much room the small files are taking up in your pool on a cumulative basis. None of my files show a record size about 128k, since that is block size that this pool was created with.  It's also why the sVDEV cutoff has to be set below the pool record size. 

Once you have your small file distribution, you need to determine where to set your cutoff re: what small files to send to the sVDEV vs. the general pool. By default, I chose 32kB which consumed 18.8GB of metadata needs. Plenty of space for the pool to grow. Extrapolate your data needs and see what kind of storage you might need. 

For example, if my pool is 20% full and small files needs make up for 18GB as shown above, then a full pool would need 5x as much sVDEV space for small files, i.e. 90GB. If I leave the small files / metadata ratio at the default 75/25% ratio, then my sVDEV pool has to be at least 120GB. However, you might find that by adjusting the record sizes that the diversity of your pool will increase and more data could be stored in the sVDEV. If you suspect that you will host more smaller files in the future, you would do well to select a much larger sVDEV to account for expansion in the future.  

### sVDEV drive selection
At minimum, choose SSDs that can handle the workload and a proven track record re: endurance. Power Loss Protection (PLP) is also a nice to have, though that can get expensive.  I went for Intel S3610 SATA SSDs that (though used) have incredible write-endurance. I also bought two spares. [I qualified all my SSDs before using them](https://www.truenas.com/community/resources/hard-drive-burn-in-testing.92/), set aside the spares in caddies, ready to mount if one of the three SSDs I use for a sVDEV develop a failure and needs to be replaced.

**Because a loss of the sVDEV will result in a pool loss, you will want to mount sVDEV SSDs in mirrors, at minimum in a 2-way.** I use a four-way mirror for my Z3 pool. 

## Implementation
Yes, the sVDEV can be enabled / attached / assigned at the GUI level, [see here](https://www.truenas.com/docs/core/coretutorials/storage/pools/fusionpool/?u=constantin). Make sure all the SSDs you need are in the sVDEV, set up as mirrors, etc. 

The small file record size cutoff can be set via the GUI - select "Storage" then "Pools", followed by going over the pool/dataset table. The pool or individual dataset small file cutoff can be adjusted by opening "edit option" (see three dots to the right of each pool or dataset name in the menu) and adjusting the "Metadata small block size". If you do it for the pool, the datasets will inherit the pool value by default. You can also do this via the CLI, per [this Level 1 post](https://forum.level1techs.com/t/zfs-metadata-special-device-z/159954/1), the command is

> zfs set special_small_blocks=64K *poolname*

or if you want to vary small file cutoffs on a dataset basis:
> zfs set special_small_blocks=128K *poolname/dataset*

If you are about to enjoy a new pool of content, congratulations, you are pretty much done. If you have an extant pool, the small-files benefit won't really start to accrue until you move the small files onto the sVDEV. 

This is also a great time to brush up on recordsize and to determine if you should adjust the record sizes of particular pools to better suit the data in them - you can do this in the GUI by selecting, Storage -> Pools and then individual datasets. Lastly, there may be some small pools (think iocage jails) where likely all data should reside on the sVDEV. Go ahead and tune the recordsize cutoff by dataset to your liking. 

There is no GUI rebalancing command, though we may get one if ZFS VDEV expansion ever becomes a reality in TrueNAS. In the meantime, there is [an excellent script written in bash that can do the rebalancing](https://github.com/markusressel/zfs-inplace-rebalancing) for you. Before you rebalance, back up, turn off snapshot and then replication tasks. I also had to delete all snapshots since it becomes very difficult to see the impact of record size changes and rebalancing efforts if snapshots pollute the data. 

In TrueNAS Core, you have to "su" to become the owner of the data before you run the script, I'd also advise to tmux the session to ensure the command can actually execute across the whole dataset. After installing the script in my root folder, my steps are as follows:

> 1. *cd /mnt/pool/dataset*
> 2. *ls* (to see who the owner is)
> 3. *tmux*
> 4. *su owner*
> 5. *bash /root/zfs-inplace-rebalancing.sh --checksum true --passes 1 /pool/path/to/rebalance*

then CTRL-B and then "d" to detach.

It will run for a while, and as it does, you'll watch the pool start to redistribute data, especially if you have also tuned the recordsize to be larger for some pools like image, archive, or video content. Hopefully, you'll see the big sets of data migrate to larger recordsizs, allowing you to fine tune the cutoff limits for the sVDEV recordsizes, storing as much small file data on the sVDEVs as possible to maximize the speed boost. 

You may have to run the rebalancing script a few times. Only when your pool is "perfect" turn both Snapshot and Replication tasks back on. I would also hit the "Add" button in Storage / Snapshot GUI at this point to manually execute a snapshot. Replication tasks can only start if there is a extant snapshot. Then go to Tasks / Replication  and "Run Now" your replication tasks to verify they're still OK. 

# Maintenance
Sadly, the dashboard currently gives ZERO clues re: the sVDEV other than whether the disks are still available and/or whether they passed their latest SMART test. For example: zero data on how full the sVDEV partitions are. If you go into reporting, you can get a glimpse into I/O of the sVDEV drives but not on a group basis, which would be helpful if your sVDEV pool contains multiple mirrors. 

Even a basic check to see how full the sVDEV is requires CLI commands (i.e. use  `zpool list -vvv poolname` to see how full each VDEV is [thank you, @NickF1227!). 

I find the lack of good documentation somewhat disheartening, sVDEVs offer a significant performance gain but have to be handled with care. Documenting the issues, implementing GUI commands, etc. should have happened by now.

### Some other considerations
I would ensure that the remote NAS for a replication task has the exact same settings re: recordsize, compression, sVDEV small file cutoff (if fitted), etc. on a dataset by dataset basis as the primary NAS. If your remote NAS already has content, I'd either wipe and start over (safest option) or nuke all remote snapshots, change the recordsizes, etc. to match the primary NAS, rebalance, and only then re-enable replication. (seems like a less safe option)

### So what though?
Well, I have found the implementation of sVDEV on my pool to enable a improvement over L2ARC in terms of metadata, even if a L2ARC is "hot", persistent, and metadata-only (the L2ARC only really helps for read operations re: metadata, not writes, which have to go to the slow HDD pool). Between sVDEV, recordsize=1M, compression=zstd, and a rebalance, my sustained large-file pool write speed has risen to 400MB/s, well above the expected 250MB/s limit I used to experience. The 
> zdb -LbbbA -U /data/zfs/zpool.cache *poolname* 

command now executes in less than two minutes and the amount of files stored in the small files sVDEV has shot up considerably. Meanwhile, my NAS' metadata needs dropped to just 0.03% of the pool. 

```
17.0T completed (331016MB/s) estimated time remaining: 0hr 00min 00sec
        bp count:              13671449        ganged count:                 0
        bp logical:      12053360033792      avg: 881644
        bp physical:     11620838932480      avg: 850007     compression:   1.04
        bp allocated:    18659308601344      avg: 1364837     compression:   0.65
        bp deduped:                   0    ref>1:      0   deduplication:   1.00
        Normal class:    18553586663424     used: 23.21%
        Special class      106690252800     used:  6.68%
        Embedded log class              0     used:  0.00%

        additional, non-pointer bps of type 0:        131
         number of (compressed) bytes:  number of bps
```
and
```
Blocks  LSIZE   PSIZE   ASIZE     avg    comp   %Total  Type
     -      -       -       -       -       -        -  unallocated
     2    32K      8K     24K     12K    4.00     0.00  object directory
     1    32K      8K     24K     24K    4.00     0.00      L1 object array
    77  38.5K   38.5K    924K     12K    1.00     0.00      L0 object array
    78  70.5K   46.5K    948K   12.2K    1.52     0.00  object array
------ SNIP! ------
  586K  18.3G   2.61G   5.23G   9.15K    7.00     0.03      L1 Total
 12.5M  10.9T   10.6T   17.0T   1.36M    1.04    99.97      L0 Total
 13.0M  11.0T   10.6T   17.0T   1.30M    1.04   100.00  Total
```
and

```
Block Size Histogram

  block   psize                lsize                asize
   size   Count   Size   Cum.  Count   Size   Cum.  Count   Size   Cum.
    512:   238K   119M   119M   238K   119M   119M      0      0      0
     1K:  80.2K  99.0M   218M  80.2K  99.0M   218M      0      0      0
     2K:   126K   334M   552M   126K   334M   552M      0      0      0
     4K:   849K  3.36G  3.90G  97.3K   526M  1.05G   458K  1.79G  1.79G
     8K:  96.8K   914M  4.80G  60.3K   662M  1.70G   916K  7.29G  9.08G
    16K:   135K  2.42G  7.21G   229K  3.86G  5.56G  63.5K  1.28G  10.4G
    32K:   197K  9.10G  16.3G   701K  23.6G  29.1G   267K  11.1G  21.5G
    64K:   144K  11.8G  28.2G  31.6K  2.49G  31.6G   135K  11.0G  32.5G
   128K:   561K  75.9G   104G   620K  79.5G   111G   566K   119G   152G
   256K:   185K  67.5G   172G  57.1K  20.3G   131G   173K  63.0G   215G
   512K:   303K   214G   386G  58.3K  40.9G   172G   105K  85.4G   300G
     1M:  10.2M  10.2T  10.6T  10.8M  10.8T  11.0T  10.4M  16.7T  17.0T
     2M:      0      0  10.6T      0      0  11.0T      0      0  17.0T
     4M:      0      0  10.6T      0      0  11.0T      0      0  17.0T
     8M:      0      0  10.6T      0      0  11.0T      0      0  17.0T
    16M:      0      0  10.6T      0      0  11.0T      0      0  17.0
```

```
NAME                                             SIZE  ALLOC   FREE  CKPOINT  EXPZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
pool                                            74.2T  17.0T  57.2T        -     -     0%    22%  1.00x    ONLINE  /mnt
  raidz3-0                                      72.7T  16.9T  55.9T        -     -     0%  23.2%      -    ONLINE
    gptid/e6a453ca-90cf-11eb-acc9-ac1f6b738b00  9.09T      -      -        -     -      -      -      -    ONLINE
    gptid/e7dfdfc1-90cf-11eb-acc9-ac1f6b738b00  9.09T      -      -        -     -      -      -      -    ONLINE
    gptid/e85bad6d-90cf-11eb-acc9-ac1f6b738b00  9.09T      -      -        -     -      -      -      -    ONLINE
    gptid/6e4220ac-c6b0-11ed-88d6-ac1f6b738b00  9.09T      -      -        -     -      -      -      -    ONLINE
    gptid/e97464f9-90cf-11eb-acc9-ac1f6b738b00  9.09T      -      -        -     -      -      -      -    ONLINE
    gptid/e98fe8b5-90cf-11eb-acc9-ac1f6b738b00  9.09T      -      -        -     -      -      -      -    ONLINE
    gptid/e9d3abb0-90cf-11eb-acc9-ac1f6b738b00  9.09T      -      -        -     -      -      -      -    ONLINE
    gptid/ea54f6eb-90cf-11eb-acc9-ac1f6b738b00  9.09T      -      -        -     -      -      -      -    ONLINE
special                                             -      -      -        -     -      -      -      -  -
  mirror-2                                      1.45T  99.4G  1.36T        -     -    14%  6.67%      -    ONLINE
    da6p1                                       1.46T      -      -        -     -      -      -      -    ONLINE
    gptid/65cf991f-90d0-11eb-acc9-ac1f6b738b00  1.46T      -      -        -     -      -      -      -    ONLINE
    gptid/6602f626-90d0-11eb-acc9-ac1f6b738b00  1.46T      -      -        -     -      -      -      -    ONLINE
logs                                                -      -      -        -     -      -      -      -  -
  gptid/b4029892-c0d0-11eb-a3ed-ac1f6b738b00    93.2G  1.65M  93.0G        -     -     0%  0.00%      -    ONLINE
cache                                               -      -      -        -     -      -      -      -  -
  gptid/66d6684f-90d0-11eb-acc9-ac1f6b738b00     932G   927G  4.20G        -     -     0%  99.5%      -    ONLINE
freenas-boot                                    58.9G  7.74G  51.1G        -     -    23%    13%  1.00x    ONLINE  -
  mirror-0                                      58.9G  7.74G  51.1G        -     -    23%  13.1%      -    ONLINE
    ada1p2                                      59.0G      -      -        -     -      -      -      -    ONLINE
    ada0p2                                      59.0G      -      -        -     -      -      -      -    ONLINE
```