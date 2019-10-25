# Supermicro BMC firmware modification


How to modify the BMC-firmware to have an alternative way to gain factory settings again. 

Note Use this variant only, if anything else (booting a linux on that supermicro board) was unsuccessful. 
**WARNING: This is a dangerous approach, as it could ends with a non functional mainboard! Be aware of that!**

1. Create a backup of your bmc firmware. If you have several boards, do this on each individual board, as the BMC firmware includes the MAC address of the IPMI interface! 
Let's assume it is called "backup.bin" 

2. Have a look at the inside of the backed'up file: 

```
~/$ binwalk backup.bin 

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
109381        0x1AB45         Certificate in DER format (x509 v3), header length: 4, sequence length: 12291
109541        0x1ABE5         Certificate in DER format (x509 v3), header length: 4, sequence length: 12291
109777        0x1ACD1         Certificate in DER format (x509 v3), header length: 4, sequence length: 12291
109913        0x1AD59         Certificate in DER format (x509 v3), header length: 4, sequence length: 12291
110057        0x1ADE9         Certificate in DER format (x509 v3), header length: 4, sequence length: 12291
112368        0x1B6F0         CRC32 polynomial table, little endian
1048576       0x100000        JFFS2 filesystem, little endian
4194304       0x400000        CramFS filesystem, little endian, size: 11915264, version 2, sorted_dirs, CRC 0xD6771DEA, edition 0, 6818 blocks, 1038 files
20971520      0x1400000       uImage header, header size: 64 bytes, header CRC: 0xC5F4666A, created: 2015-10-05 10:52:56, image size: 1537322 bytes, Data Address: 0x40008000, Entry Point: 0x40008000, data CRC: 0x677BDAA8, OS: Linux, CPU: ARM, image type: OS Kernel Image, compression type: gzip, image name: "21400000"
20971584      0x1400040       gzip compressed data, maximum compression, has original file name: "linux.bin", from Unix, last modified: 2015-10-05 10:49:39
24117248      0x1700000       CramFS filesystem, little endian, size: 5435392, version 2, sorted_dirs, CRC 0x43329740, edition 0, 2071 blocks, 309 files
```
Our main interest is the JFFS2 filesystem. everything else will kept as it is. Note the HEXADECIMAL positions inside the binary file. 
We will cut the backup.bin file into 3 parts: ```start.bin```, then ```0x100000.jffs``` and finally ```end.bin```.

```start.bin``` starts at byte 0 and it's length is: 0x100000 (10000 in hexadecimal)
```0x10000.jffs2``` starts at 0x100000 and it's lenght is the difference between it's own startposition and the startposition of the next entry (CramFS at 0x40000). In this example 0x400000 - 0x100000 
```end.bin``` starts at 0x40000 and we don't need to specify the end as we just cat everything in until the end of the file. 

To do that in a unix shell: 
``` 
~/$ dd if=./backup.bin bs=1 of=./start.bin count=$((0x100000))
~/$ dd if=./backup.bin bs=1 of=./0x100000.jffs skip=$((0x100000)) count=$((0x400000-0x100000))
~/$ dd if=./backup.bin bs=1 of=./end.bin skip=$((0x400000))
``` 

3. Safecheck: Put all cutted pieces together again.

The worst thing is that you already messed up with the splittings above. So let's check. 
Get the md5sum (or sha256sum if you like) of the original backup:
```
~/$ md5sum backup.bin 
943585ff7de899e48aa627af579709bd  backup.bin
```
Then merge the 3 parts together and check again: 
```
cat ./start.bin ./0x100000.jffs ./end.bin >test_merge.bin 
~/$ md5sum backup.bin test_image.bin 
943585ff7de899e48aa627af579709bd  backup.bin
943585ff7de899e48aa627af579709bd  test_merge.bin
```
The checksums are same. Perfect!


4. Mounting the JFFS2 volume

```
~/$ sudo modprobe loop
~/$ sudo modprobe mtdblock
~/$ sudo losetup /dev/loop0 ./0x100000.jffs 
~/$ sudo modprobe block2mtd block2mtd=/dev/loop0,128ki
~/$ sudo mount -tjffs2 /dev/mtdblock0 /mnt
```
5. Removing the PSBlock to do a factory reset  
```
~/$ sudo rm /mnt/PSBlock 
```
6. Umount and unload all required linux modules 
```
~/$ sudo umount /mnt
~/$ sudo rmmod block2mtd
~/$ sudo rmmod mtdblock
~/$ sudo losetup -d /dev/loop0 
```

7. Merge the patched JFFS2 file into the firmware image
```
~/$ cat ./start.bin ./0x100000.jffs ./end.bin >patched_image.bin 
~/$ md5sum backup.bin test_image.bin patched_image.bin 
943585ff7de899e48aa627af579709bd  backup.bin
943585ff7de899e48aa627af579709bd  test_merge.bin
5fac4e900671cef5caae641b94a79066  patched_image.bin
```
As you can see the patched image differ between the original and the first test-merge.
Ensure that the filesize is still the same.

8. Flash the modified BMC firmware
Flash your modified BMC firmware to the board and power it up again. 
The IPMI interface will get a dynamic ip address via DHCP and your login/password is: ADMIN/ADMIN. 

Good luck!

