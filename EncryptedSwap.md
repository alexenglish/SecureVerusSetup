This is for the situation where you need an encrypted swap and do not have an encrypted system volume.

We'll look at how to encrypt and use an existing swap partition and how to create an encrypted swap volume in a file. There are some caveats - with these methods you have to manually mount the swap volume after login. It can be scripted, but you still have to provide a password manually. This also means that if your system maxes out its RAM during boot you may fail to boot. This setup will also not permit resume from suspend from what I understand, so it is only suitable for systems that will not suspend or hibernate.

## Encrypting a swap partition
First, disable the swap partition if it is in use. Use `swapon` with no arguments to get a list of the swap volumes in use. Use `swapoff /dev/sdb2`, substituting the path of your swap, to disable it. This may not go well if it is being utilized, so check to see that you're not using much of it using `free -h`.

In `/etc/crypttab` add this line (replacing the path to your swap volume):
```
swap  /dev/sdb2  /dev/urandom  swap,cipher=aes-xts-plain64,size=512
```

In your '/etc/fstab' edit your swap entry to use `/dev/mapper/swap` rather than the path to the volume itself. It should look something like this:
```
/dev/mapper/swap    none    swap    sw    0   0
```

## Encrypting a swap volume in a file
If you don't yet have a swap file create it (adjust the size to your needs), set the permissions:
```
sudo fallocate -l 12G /swapfile
sudo chmod 600 /swapfile
```

Now follow the steps used above for encrypting a swap partition, but substitute `/swapfile` for the path to the swap partition.

If you do not yet have a swap entry in your `/etc/fstab` you'll want to add this:
```
/dev/mapper/swap    none    swap    sw    0   0
```

## Confirming
This should be all that's needed. Restart and confirm using `swapon` to view active swap volumes and `free -h` to see swap size/utilization.
