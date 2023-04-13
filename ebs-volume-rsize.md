# _Warning - Steps to lower Root EBS Volume still a work in progress and being tested_

# Steps to Lower _Root_ EBS Volume Size on an Instance
1. Create new Ubuntu Micro instance (e.g., Name = Tools)
1. Stop instance you want to lower EBS volume.
1. Create AMI image as backup of your instance.
1. Create an empty X GB Amazon EBS volume in the same availability zone where X is smaller size you desire.
1. Detach the volume you wish to resize from the stopped instance from step 2 above.
1. Attach the original volume to the Tools instance (e.g., as /dev/xvdf)
1. Attach new downsized volume to the Tools instance (e.g., as /dev/xvdg)
1. SSH into Tools instance and run these steps:
  1. Verify newly attached partitions via ```cat /proc/partitions```
  1. To ensure that the existing original file system is in order, run ```e2fsck -f /dev/xvdf```.
  1. If the e2fsck command ran without errors, now run ```resize2fs -M -p /dev/xvdf```.
  1. The last line from the resize2fs command should tell you how many 4k blocks the filesystem now is. To calculate the number of 16MB blocks you need, use the following formula: blockcount * 4 / (16 * 1024). Round this number up to nearest integer and use this rounded number to give yourself a little buffer.
  1. If you dont yet have a partition on your new volume (/dev/xvdg1), use fdisk as follows:
    1. ```fdisk /dev/xvdg```
    1. p (print to view partitions)
    1. n (new partition)
      1. p (primary partition)
      1. Press enter to accept default first sector
      1. Press enter to accept default last sector
      1. w (write out changes)
  1. Execute the following command, using the number you came up with in the previous step. ```dd bs=16M if=/dev/xvdf of=/dev/xvdg count=numberfrompreviousstep```. Depending on how large your volume is this may take several minutes to run -- let it finish.
  1. After the copy finishes, resize and check and make sure that everything is in order with the new filesystem by running:
    1. ```resize2fs -p /dev/xvdg```
    1. ```e2fsck -f /dev/xvdg```
1. After this step is complete, time to wrap this up:
  1. Detach downsized volume from the Tools instance you created.
  1. Attach the shrunken volume to the old EC2 instance as /dev/sda1 (your boot device)
  1. Restart your old instance. 
  1. Ensure Instance boots up correctly and is working well.
  1. If all is well, cleanup:
    1. Save the previous, larger volume until you've validated that everything is working properly.
    1. When you've verified things are working well, feel free to delete the new EC2 Tools instance you created, plus the larger volume and any backup AMI's and/or snapshots you created.

# Steps to Lower _Non-Root_ EBS Volume Size on an Instance

1. Stop instance you want to lower EBS volume.
1. Create an empty X GB Amazon EBS volume in the same availability zone where X is smaller size you desire.
1. Attach new volume to the instance and again note all device name details.
1. Start instance.
1. SSH into instance.
1. Run these commands:

  ```bash
  cat /proc/partitions
  cat /etc/fstab
  mkfs -t ext4 /dev/xvdg (assuming xvdg is the new device you just attached)
  ```
1. Create mount directory and mount new volume

  ```bash
  mkdir /mnt/small
  mount /dev/xvdg /mnt/small
  ```
1. Sync the files to smaller volume (assumes /mnt/original/ is where your original volume you want to resize - substitute as appropriate)

  ```bash
  rsync -aHAXxSP  /mnt/original/ /mnt/small > rsync-aHAXxSP.out 2>&1 &
  diff -qr /mnt/original/ /mnt/small
  ```
1. Unmount smaller volume

  ```bash
  umount /dev/xvdg
  ```
1. Stop the instance
1. Detach both old and newly resized volumes
1. Attach just the newly resized volume now and make sure to use same /dev/… as detailed in step 3 of original volume you are replacing (this ensures that nothing bad will happen in case of auto-mounting scripts - e.g., referenced by rc0.d startup scripts referencing /etc/fstab).
1. Start instance
1. SSH into instance
1. Verify new volume is auto-mounted and working properly as expected.
  1. review ownership and permissions

# Steps to Lower EBS Volume Size on an AMI
1. Create new fresh instance from AMI you want to modify.
1. Follow Steps to Lower EBS Volume Size on an Instance (see above)
1. [Optional] Rename large sized AMI to prior AMI name -old or -large
1. Create new AMI from running instance to snapshot with lowered volume size
1. [Optional] Delete old/larger AMI

# Steps to Raise EBS Volume Size on an Instance
1. TODO

# Steps to Raise EBS Volume Size on an AMI
1. Launch new instance from AMI
1. Increase storage size from default AMI
1. [Optional] Copy small sized AMI to prior AMI name -old or -small
  1. [Optional] Delete prior AMI to make room for replacement
1. Wait for instance to be running
1. Resize the file system so that it fills up the entire newly increased EBS volume

  ```bash
  df -h
  resize2fs /dev/xvdf # (where xvdf is the device representing increased EBS volume)
  df -h
  ```
1. Create new AMI from running instance to snapshot with increased volume size

## References:
* https://matt.berther.io/2015/02/03/how-to-resize-aws-ec2-ebs-volumes/
* http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-expand-volume.html#migrate-data-larger-volume
* http://cloudacademy.com/blog/amazon-ebs-shink-volume/
* http://www.n2ws.com/how-to-guides/how-to-reduce-the-size-of-an-ebs-volume.html
* https://alestic.com/2009/12/ec2-ebs-boot-resize/
* http://superuser.com/questions/594203/how-to-copy-entire-linux-root-filesystem-to-new-hard-drive-on-with-ssh-and-tar
* https://wiki.archlinux.org/index.php/full_system_backup_with_rsync
