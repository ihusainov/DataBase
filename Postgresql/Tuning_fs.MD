
**Solution 1
Different instances use different file systems.
for example**
```bash
mkfs.ext4 /dev/mapper/vgdata01-lv01  
mkfs.ext4 /dev/mapper/vgdata01-lv02  
mount /dev/mapper/vgdata01-lv01 /data01  
mount /dev/mapper/vgdata01-lv02 /data02 
```
**Two database instances are placed in / data01 and / data02, respectively.
Limiting the IOPS of / dev/mapper/vgdata01-lv01 will not affect another file system.
The disadvantage of this method is that if there are many instances, it needs to be split into many small file systems, which is not suitable for spatial flexible management and sharing.**


**Solution 2
Aiming at ext4
Normally, if you want to modify metadata, you must make sure that the block changes corresponding to metadata have been abolished, so there may be situations where writing metadata is forced to brush dirty data page.
pic
If dirty data page brushes are slow, metadata writes will be blocked. Writing metadata journals is serial, which inevitably affects the modification of metadata journals by other processes.
Use data=writeback to load the ext4 file system.
The principle of this method is that you don't have to wait for data to finish before you write metadata. Thus, metadata may be new, but the data is old. (For example, inode is new, data is old, maybe some inode reference blocks do not exist or old deleted blocks)
Writing metadata does not wait to write data. The advantage is that the serial operation is not good because the data is blocked.**
```bash
       data={journal|ordered|writeback}  
              Specifies the journalling mode for file data.  Metadata is always journaled.  To use modes other than ordered on the root filesystem, pass the mode to  the  kernel  as  boot  parameter,  e.g.   root-  
              flags=data=journal.  

              journal  
                     All data is committed into the journal prior to being written into the main filesystem.  

              ordered  
                     This is the default mode.  All data is forced directly out to the main file system prior to its metadata being committed to the journal.  

              writeback  
                     Data  ordering  is not preserved - data may be written into the main filesystem after its metadata has been committed to the journal.  This is rumoured to be the highest-throughput option.  It  
                     guarantees internal filesystem integrity, however it can allow old data to appear in files after a crash and journal recovery.  
```

The disadvantage is that after crash, the file system or the operating system may cause inconsistency between metadata and data, resulting in dirty blocks.


**Solution 3
It will be used as a journal block device independently. When IOPS is restricted, IO of journal block device is not limited (because metadata journal operates very little and quickly, and there is no need to restrict), only IO of data block device is restricted.
This method is only suitable for xfs file systems.
The ext4 filesystem does not work with this method. The ext4 separate journal dev method is as follows, but it doesn't work. You can try it.
Create logical volumes, one for DATA and one for journal**
```bash
#pvcreate /dev/dfa  
#pvcreate /dev/dfb  
#pvcreate /dev/dfc  
#vgcreate aliflash /dev/dfa /dev/dfb /dev/dfc  
#lvcreate -i 3 -I 8 -L 1T -n lv01 aliflash  
#lvcreate -i 3 -I 8 -L 2G -n lv02 aliflash  
```
**Create journal block device**
```bash
#mkfs.ext4 -O journal_dev -b 4096 /dev/mapper/aliflash-lv02  
mke2fs 1.41.12 (17-May-2010)  
Discarding device blocks: done                              
Filesystem label=  
OS type: Linux  
Block size=4096 (log=2)  
Fragment size=4096 (log=2)  
Stride=2 blocks, Stripe width=6 blocks  
0 inodes, 525312 blocks  
0 blocks (0.00%) reserved for the super user  
First data block=0  
0 block group  
32768 blocks per group, 32768 fragments per group  
0 inodes per group  
Superblock backups stored on blocks:   

Zeroing journal device: done     
```
**Create ext4 file system**
```bash
#mkfs.ext4 -E stride=16,stripe-width=48 -J device=/dev/mapper/aliflash-lv02 /dev/mapper/aliflash-lv01  
mke2fs 1.41.12 (17-May-2010)  
Using journal device's blocksize: 4096  
Discarding device blocks: done                              
Filesystem label=  
OS type: Linux  
Block size=4096 (log=2)  
Fragment size=4096 (log=2)  
Stride=16 blocks, Stripe width=48 blocks  
67117056 inodes, 268437504 blocks  
13421875 blocks (5.00%) reserved for the super user  
First data block=0  
Maximum filesystem blocks=4294967296  
8193 block groups  
32768 blocks per group, 32768 fragments per group  
8192 inodes per group  
Superblock backups stored on blocks:   
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,   
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,   
        102400000, 214990848  

Writing inode tables: done                              
Adding journal to device /dev/mapper/aliflash-lv02: done  
Writing superblocks and filesystem accounting information: done  

This filesystem will be automatically checked every 31 mounts or  
180 days, whichever comes first.  Use tune2fs -c or -i to override.  

#ll /dev/mapper/aliflash-lv0*  
lrwxrwxrwx 1 root root 7 Jan  7 11:12 /dev/mapper/aliflash-lv01 -> ../dm-0  
lrwxrwxrwx 1 root root 7 Jan  7 11:12 /dev/mapper/aliflash-lv02 -> ../dm-1  
#ll /dev/dm-0  
brw-rw---- 1 root disk 253, 0 Jan  7 11:22 /dev/dm-0  
#ll /dev/dm-1  
brw-rw---- 1 root disk 253, 1 Jan  7 11:22 /dev/dm-1 
```
**Mount file system**
```bash
#mount -o nobarrier,noatime,nodiratime,discard,defaults,nodelalloc /dev/mapper/aliflash-lv01 /data01
```
**Using the method at the beginning of this paper, only IOPS of / dev/mapper/vgdata01-lv01 is limited, and testing can not solve the problem.
The method of using journal dev in XFS file system**
```bash
# mkfs.xfs -f -b size=4096 -l logdev=/dev/mapper/vgdata01-lv02,size=2136997888,sunit=16 -d agcount=9000,sunit=16,swidth=48 /dev/mapper/vgdata01-lv01   
# mount -t xfs -o nobarrier,nolargeio,logbsize=262144,noatime,nodiratime,swalloc,logdev=/dev/mapper/vgdata01-lv02 /dev/mapper/vgdata01-lv01 /data01  
```
**Using the method at the beginning of this paper, only IOPS of / dev/mapper/vgdata01-lv01 is limited, and the problem is**
