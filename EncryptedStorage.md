This method will allow you to create a file that will contain an encrypted volume, on which you'll create a filesystem that can be mounted like any other storage device.
The tools to do this should generally be installed on most Debian/Ubuntu systems by default, if not or if using another system you may need to hunt them down in your package manager. On a very bare-bones install you may need to install `cryptsetup`.
DO NOT STORE YOUR ONLY COPY OF ANY CRITICAL DATA IN ONE OF THESE VAULTS. THEY ARE SAFE, THEY ARE SECURE, BUT ALWAYS HAVE A BACKUP. FILES AND DEVICES CAN BE CORRUPTED, AND PASSWORDS CAN BE FORGOTTEN OR MISPLACED. ALSO MAKE SURE YOU'VE PRACTICED WITH OPENING AND MOUNTING, THEN UNMOUNTING AND CLOSING BEFORE RELYING ON THIS SETUP, GET FAMILIAR WITH IT.

*Please read this whole guide before starting as you'll benefit as you make decisions about how to name or size things.*

## Creating the file and populating it with the encrypted volume
Start by creating a file filled with random data in the size you'd like for your volume. The random data provides both some extra security (hard to know what is random data and what is encrypted data) and some plausible deniability about the utilization of space within the volume.

In this case, we'll name the file vault.img, and we'll make it 200GB, specified as 200000 MB:
```
dd if=/dev/urandom of=vault.img bs=1M count=200000
```

Now create the encrypted volume in the file. It should prompt for a password twice:
```
sudo cryptsetup --verify-passphrase luksFormat vault.img
```

## Opening the volume, creating the filesystem
Now we can open that volume (it should ask for the password):
```
sudo cryptsetup open --type luks vault.img vaultcrypt
```

You should now see vaultcrypt in /dev/mapper/ if you run `ls /dev/mapper`

Now you can create the filesystem on the volume, in this case we're labeling the volume "Vault":
```
sudo mkfs.ext4 -L Vault /dev/mapper/vaultcrypt
```

## Storing your Verus data
If you're creating this vault for securing Verus data, the recommended setup is either:
1. Create a mount point in the home directory with subfolders for ~/.komodo/VRSC and for ~/.verus, both symlinked.
2. Log in as one user to mount a secure volume as the home directory of another user you'll then log into and run systems with.

### For option 1 - storing data directories
Starting in your home directory - 
#### Make the mount point and mount the filesystem
```
mkdir vault
sudo mount /dev/mapper/vaultcrypt vault
```

#### Setting permissions
Set the ownership (to root) and permissions of the vault file to make accidental overwrite more difficult.
```
sudo chown root:root vault.img
sudo chmod 600 vault.img
```

Now update ownership and permissions of the mount to permit your user writing to it. Substitute your user name.
```
sudo chmod 700 vault
sudo chown USERNAME:USERNAME
```

#### Setting up data directories
If you've already run verusd, stop it if it's already running. Then:
```
mv .komodo/VRSC vault/
```

Otherwise, if that directory doesn't exist:
```
mkdir vault/VRSC
```

Then link it up to its normal path:
```
ln -s ~/vault/VRSC .komodo/
```

Same process for the .verus directory for pbaas/veth config/data:
```
mv .verus/pbaas vault/
```
or, if it doesn't exist yet:
```
mkdir vault/pbaas
```

Thank link it:
```
ln -s ~/vault/pbaas .verus/pbaas
```

Now your entire Verus data directory and pbaas/gateway config/data directories are in your encrypted volume. If you like you can also create other directories and files in the encrypted volume.

### For option 2: encrypted home directory
The details of the user management will be distribution specific. And not all of this has been tested yet, apologies, if you find something that doesn't work please let me know and I'll provide support and make an update.

I would recommend having a user named "verus" and another with a username of your choosing. I'll assume for this process that you created the encrypted volume file in the home directory of a user named lex, and that we'll need to create the verus user and mount the encrypted volume as it's home.

We'll start by creating the mount point, the verus user's home directory:
```
sudo mkdir /home/verus
```

Now mount the encrypted volume there:
```
sudo mount /dev/mapper/vaultcrypt /home/verus
```

Now we'll create the Verus user:
```
sudo adduser verus
```

And we'll confirm ownership of the home directory:
```
sudo chown -R verus:verus /home/verus
sudo chmod 700 /home/verus
```

Now, when you log into the Verus user the entire home directory will be encrypted, so you can install and run Verus, bridgekeeper, and store any other files in the home directory and it will all be encrypted. If you log into this user account when the vault hasn't been mounted you'll find the home directory is empty, or you might have trouble logging in.

Also, rather than logging out and back in again, you can switch users using:
```
su - verus
```
or specify another username to use that user. If you are logged in as root you can do this without a password. Otherwise you'll enter the password for that user.

## General Usage
After you're all set up you'll have to know how to open/mount and unmount/close your volume for normal usage.

On system startup you'll run (using the mount point you chose in option A or B above, and logged in as the appropriate user):
```
sudo cryptsetup open --type luks vault.img vaultcrypt
sudo mount /dev/mapper/vaultcrypt <mount point>
```

If you'd like to unmount and close the volume while keeping your system running, do:
```
sudo umount vaultcrypt
sudo cryptsetup close vaultcrypt
```

When you shut down or restart your system gracefully this will be unmounted and closed gracefully for you.
