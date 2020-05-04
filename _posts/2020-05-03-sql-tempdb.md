---
layout: post
title: SQL Server tempdb on EC2 instance storage, on Linux
description: You can set up SQL Server on Amazon Linux to use EC2 instance storage as long as you mount the volumes on every boot
---

Normally I'm a big fan of using managed AWS services like RDS, Redshift and Aurora,
so you don't have to be in the business of managing your own database.  Still, there
are some some edge cases where you need finer-grained control over storage, and
running a DB like SQL Server on EC2 makes sense.  AWS makes a SQL Server AMI
for Linux available on the marketplace.  

One advantage of running your own SQL Server on EC2 is you can put `tempdb` on
 EC2 instance storage (SSDs attached directly to the EC2 instance itself,
as opposed to EBS volumes).  This option is not available on RDS, which uses EBS
exclusively.  If you're running SQL Server on EC2, it's a best practice to use
instance storage for `tempdb` for lower latency.

 The main catch is that
 EC2 instance storage is tied to the instance lifecycle itself;
whenever the instance is stopped or terminated, the instance storage goes away.  This is not a
problem for `tempdb`, though, as SQL Server re-generates the `tempdb` data files on every restart.
So as long as the instance storage is re-initialized before SQL Server starts up, it will work.

There are two main components to this process:

* a script to initialize and mount EC2 instance storage, and

* a script to configure `tempdb` datafiles in SQL Server, executed once when
the instance is first created (e.g., userdata)

First you need to initialize EC2 instance storage.  Here's a script that will
do that:

```sh
## prerequisite: yum install nvme-cli
INSTANCE_VOLUMES=$(nvme list | grep Instance | cut -f1 -d' ')
i=0
for vol in $INSTANCE_VOLUMES; do
  mkfs -t xfs $vol
  mount_point=/var/opt/mssql/tempdb$i
  mkdir $mount_point
  mount $vol $mount_point
  chown mssql.mssql $mount_point
  i=$(expr $i + 1)
done
```

This code is generic to set up as many `tempdbX` mount points as there are instance storage
devices on the EC2 instance.  If the EC2 instance has one instance volume (e.g., r5d.large)
you'll get a single `tempdb0` folder.  If you have two devices (r5d.4xlarge) the same code will give you
a `tempdb0` and `tempdb1`, etc.

This works by using the `nvme` command to get the list of available NVMe device
names, and filtering for only the devices that are instance stores, not EBS volumes.  Then
for each matching device name (e.g., /dev/nvme1n1), create a filesystem on the device,
create the mount point, and mount it.  Also set the owner to the `mssql` user after
the filesystem is mounted.

Note that you can't depend on the order or numbering of devices
for NVMe, so merely adding a device name to `/etc/fstab` won't work!

Next, you have to figure out how to get this to run at the right time on boot.
SQL Server is managed by systemd, so it makes the most sense to add another systemd process
to execute this script, to be run *before* SQL Server starts.  (If SQL Server starts before the
volumes are re-mounted, SQL will happily re-create tempdb files in the mount point itself as a
regular folder -- remember the mount *point* exists on the root volume and is persistent across
instance restarts!)

I put this file in `/etc/systemd/system/multi-user.target.wants/remount-tempdb.service`
to make that happen:

```
[Unit]
Description=SQL TempDB setup
Before=mssql-server.service

[Service]
Type=oneshot
ExecStart=/path/to/mount-instance-volumes.sh # change to wherever the first script lives

[Install]
WantedBy=multi-user.target
```

Finally, you need to configure SQL Server itself to use the new tempdb volumes.
This only needs to be done once on instance creation, after the mount points
exist:

```
count=$(nvme list | grep Instance | wc -l)

sqlscript=/tmp/tempdb.sql
echo "ALTER DATABASE tempdb MODIFY FILE (NAME = tempdev, FILENAME = '/var/opt/mssql/tempdb0/tempdb.mdf');" > $sqlscript
echo "ALTER DATABASE tempdb MODIFY FILE (NAME = templog, FILENAME = '/var/opt/mssql/tempdb0/templog.ldf');" >> $sqlscript

# move existing tempdev / log plus create 7 more log files for 8 total
for i in {1..7}; do
  x=$(expr $i % $count)
  echo "ALTER DATABASE tempdb ADD FILE (NAME = tempdev$i, FILENAME = '/var/opt/mssql/tempdb$x/tempdev$i.ndf');" >> $sqlscript
done

/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P (password) -i $sqlscript

## RESTART to make sure tempdb actually moves
systemctl restart mssql-server
```

A few details to explain

* Again, use `nvme` to determine how many instance volumes are attached
* Assume creating 8 tempdb files in total (based on [this guidance](https://www.brentozar.com/blitz/tempdb-data-files/))
* Spread the 8 tempdb files across the instance volumes equally, using modulo arithmetic
* Treat the first tempdb data file plus logfile as a special case -- the existing files need to move, the others get created

At this point, tempdb should be ready to go on the instance volumes, and the instance volumes will be
re-initialized and mounted before SQL Server starts up on reboot.
