# ebs-automatic-nvme-mapping
Automatic Mapping of NVMe-style EBS Volumes to Standard Block Device Paths
Context:
So once you've decided to use the new c5.large or m5.large instance type in EC2.

Congratulations! You get more IOPS!

But wait there is something to take a NOTE of : -

The Newer Generation Instance Family of AWS like the t3, m5, c5 etc., and the instances with a NVMe storage attached in-build, seems to be reporting the EBS / NVMe Volumes differently when listed in the block devices. 


Although we mount the EBS volumes at a particular mount point on the system / OS at the `/dev/` location, with device alias as `xvdm` or the root initial mounts as `sda1` which is by default when an instance boots up.

The actual `lsblk` output is different.

# The Problem:
On the older t2.medium instance family with a separate EBS volume of 1 GB attached at `/dev/xvdm` location looks like the below:

```
ubuntu@ip-XXX:~$ df -hT /
Filesystem Type Size Used Avail Use% Mounted on
/dev/xvda1 ext4 7.7G 2.6G 5.1G 34% /
```

On a newer generation c5d.large instance family with an in-build NVMe Storage of 50 GB and with a separate EBS volume of 1 GB  attached at `/dev/xvdm` location looks like the below:
```
ubuntu@ip-XXX:~$ df -hT /
Filesystem Type Size Used Avail Use% Mounted on
/dev/nvme1n1p1 ext4 7.7G 2.6G 5.1G 34% /
```

In order to resolve the issue of having a static mount points / Block Device Path, which is a requirement to have in our fstab entries, which takes care of the Volume being in attached mode in event of any REBOOT or STOP & START of the instance takes place.

The Current fstab entry for one of the master looks as below: 
```
ubuntu@master:~$ cat /etc/fstab
LABEL=cloudimg-rootfs / ext4 defaults,discard 0 0
/dev/xvdm /data1 ext4 defaults,nobootwait,noatime,barrier=0 0 0
```

We rely on the device path to make sure the EBS Volume is on mounted state even for the BETA and ALPHA environments were the EBS volume is rotated with a new volume created from a latest un-sanitised Data Snapshot.



Rest of the Databases the EBS Volumes seems to be rarely rotated as this volumes will be serving a active Production Workload or attached to an active Replication Slave Instance. 

The Community or the Best Practise way of doing things w.r.t to fstab entries is to use the BLKID or map the UUID of the Volume in the /etc/fstab entry to boot and mount. 

Limitations of using the above:

1.) The UUID of a volume can only be retrieved only when once it is attached to the instance.

2.) UUID/BLKID of a volume remains the same as long as the same volume is being attached to the instance. (In case the volume is rotated there should be a change made to the fstab entry as well, else the ssh to the instance might fail)

BrainStorming:
There's more to this NVMe business. Let's get to it.

We'll start by install the NVMe tools, and requesting a list of all NVMe devices in the system.
```
sudo apt-get install nvme-cli
```
Run: sudo nvme --list 

```
ubuntu@ip-XXX:~$ sudo nvme id-ctrl /dev/nvme1n1
NVME Identify Controller:
vid     : 0x1d0f
ssvid   : 0x1d0f
sn      : vol09b1a6fc74e2f1855
mn      : Amazon Elastic Block Store
fr      : 1.0
rab     : 32
ieee    : dc02a0
cmic    : 0
mdts    : 6
cntlid  : 0
ver     : 0
rtd3r   : 0
rtd3e   : 0
oaes    : 0
ctratt  : 0
oacs    : 0
acl     : 4
aerl    : 0
frmw    : 0x3
lpa     : 0
elpe    : 0
npss    : 1
avscc   : 0x1
apsta   : 0
wctemp  : 0
cctemp  : 0
mtfa    : 0
hmpre   : 0
hmmin   : 0
tnvmcap : 0
unvmcap : 0
rpmbs   : 0
edstt   : 0
dsto    : 0
fwug    : 0
kas     : 0
hctma   : 0
mntmt   : 0
mxtmt   : 0
sanicap : 0
hmminds : 0
hmmaxd  : 0
sqes    : 0x66
cqes    : 0x44
maxcmd  : 0
nn      : 1
oncs    : 0
fuses   : 0
fna     : 0
vwc     : 0x1
awun    : 0
awupf   : 0
nvscc   : 0
acwu    : 0
sgls    : 0
subnqn  :
ioccsz  : 0
iorcsz  : 0
icdoff  : 0
ctrattr : 0
msdbd   : 0
ps    0 : mp:0.01W operational enlat:1000000 exlat:1000000 rrt:0 rrl:0
          rwt:0 rwl:0 idle_power:- active_power:-
ps    1 : mp:0.00W operational enlat:0 exlat:0 rrt:0 rrl:0
          rwt:0 rwl:0 idle_power:- active_power:-
```

How about MORE information? Yes.

```
ubuntu@ip-XXX:~$ sudo nvme id-ctrl --vendor-specific  /dev/nvme1n1
NVME Identify Controller:
vid     : 0x1d0f
ssvid   : 0x1d0f
sn      : vol09b1a6fc74e2f1855
mn      : Amazon Elastic Block Store
fr      : 1.0
rab     : 32
ieee    : dc02a0
cmic    : 0
mdts    : 6
cntlid  : 0
ver     : 0
rtd3r   : 0
rtd3e   : 0
oaes    : 0
ctratt  : 0
oacs    : 0
acl     : 4
aerl    : 0
frmw    : 0x3
lpa     : 0
elpe    : 0
npss    : 1
avscc   : 0x1
apsta   : 0
wctemp  : 0
cctemp  : 0
mtfa    : 0
hmpre   : 0
hmmin   : 0
tnvmcap : 0
unvmcap : 0
rpmbs   : 0
edstt   : 0
dsto    : 0
fwug    : 0
kas     : 0
hctma   : 0
mntmt   : 0
mxtmt   : 0
sanicap : 0
hmminds : 0
hmmaxd  : 0
sqes    : 0x66
cqes    : 0x44
maxcmd  : 0
nn      : 1
oncs    : 0
fuses   : 0
fna     : 0
vwc     : 0x1
awun    : 0
awupf   : 0
nvscc   : 0
acwu    : 0
sgls    : 0
subnqn  :
ioccsz  : 0
iorcsz  : 0
icdoff  : 0
ctrattr : 0
msdbd   : 0
ps    0 : mp:0.01W operational enlat:1000000 exlat:1000000 rrt:0 rrl:0
          rwt:0 rwl:0 idle_power:- active_power:-
ps    1 : mp:0.00W operational enlat:0 exlat:0 rrt:0 rrl:0
          rwt:0 rwl:0 idle_power:- active_power:-
vs[]:
       0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
0000: 73 64 61 31 20 20 20 20 20 20 20 20 20 20 20 20 "sda1............"
0010: 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 "................"
0020: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0030: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0040: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0050: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0060: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0070: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0080: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0090: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
00a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
00b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
00c0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
00d0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
00e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
00f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0100: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0110: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0120: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0130: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0140: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0150: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0160: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0170: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0180: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0190: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
01a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
01b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
01c0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
01d0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
01e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
01f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0200: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0210: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0220: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0230: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0240: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0250: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0260: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0270: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0280: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0290: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
02a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
02b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
02c0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
02d0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
02e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
02f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0300: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0310: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0320: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0330: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0340: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0350: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0360: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0370: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0380: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
0390: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
03a0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
03b0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
03c0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
03d0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
03e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
03f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 "................"
 There's our requested block device name, at the beginning of the vendor specific information.
```
Let's extract it. How? The nvme id-ctrl command takes an option called --raw-binary, which dumps out the information block inclusive of the vendor-specific data.
```
ubuntu@ip-XXX:~$ sudo nvme id-ctrl --raw-binary  /dev/nvme1n1 | hexdump -C
binary output
00000000  0f 1d 0f 1d 76 6f 6c 30  39 62 31 61 36 66 63 37  |....vol09b1a6fc7|
00000010  34 65 32 66 31 38 35 35  41 6d 61 7a 6f 6e 20 45  |4e2f1855Amazon E|
00000020  6c 61 73 74 69 63 20 42  6c 6f 63 6b 20 53 74 6f  |lastic Block Sto|
00000030  72 65 20 20 20 20 20 20  20 20 20 20 20 20 20 20  |re              |
00000040  31 2e 30 20 20 20 20 20  20 a0 02 dc 00 06 00 00  |1.0      .......|
00000050  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000100  00 00 04 00 03 00 00 01  01 00 00 00 00 00 00 00  |................|
00000110  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000200  66 44 00 00 01 00 00 00  00 00 00 00 00 01 00 00  |fD..............|
00000210  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000800  01 00 00 00 40 42 0f 00  40 42 0f 00 00 00 00 00  |....@B..@B......|
00000810  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00000c00  73 64 61 31 20 20 20 20  20 20 20 20 20 20 20 20  |sda1            |
00000c10  20 20 20 20 20 20 20 20  20 20 20 20 20 20 20 20  |                |
00000c20  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001000
```
The information within the vendor-specific data we are looking for appears to start at an offset of 3,072 bytes, is a 32-byte record, and is padded with spaces  

(0x20 == 32 dec == <SPACE>)
```
ubuntu@ip-XXX:~$ sudo nvme id-ctrl --raw-binary /dev/nvme1n1 | cut -c3073-3104
binary output
sda1
```  
It looks good but still has some trailing spaces at the end.

```
ubuntu@ip-XXX:~$ sudo nvme id-ctrl --raw-binary /dev/nvme1n1 | cut -c3073-3104 | tr -s ' ' | sed 's/ $//g'
binary output
sda1
```

Solution:

Using UDEV SPACE: 

udev, the userspace /dev manager, operates as a daemon, that receives events each time a device is attached/detached from the host.

It reads each event, compares the attributes of that event to a set of rules (located in /etc/udev/rules.d), and executes the specified actions of the rule.

```
sudo udevadm info --query=all --attribute-walk --path=/sys/block/nvme1n1
```
```
looking at parent device '/devices/pci0000:00/0000:00:1f.0/nvme/nvme1':
    KERNELS=="nvme1"
    SUBSYSTEMS=="nvme"
    DRIVERS==""
    ATTRS{model}=="Amazon Elastic Block Store              "
    ATTRS{serial}=="vol22222222222222222"
    ATTRS{firmware_rev}=="1.0     "
```
What are the attributes of our NVMe device that we can match on?

In order to identify EBS volumes, we want something that is stable across reboots. 

I've picked ATTRS{model}.

Let's combine what we've found into a shell script...

1.) script to add the ebs-nvme-mapping 

2.) script to remove the ebs-nvme-unlink

3.) configuration of the rules as a sub-module to the udev USER DEVICE SPACE

udev will automatically reload rules upon changes to files in the rules directory. So we're locked and loaded.

Now, when we attach and detach EBS volumes, our shell script will run.


References : 

1: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/nvme-ebs-volumes.html

2: https://en.wikipedia.org/wiki/Udev

