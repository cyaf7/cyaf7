---
description: 'Here I''ll leave the basic commands to navigate:'
---

# File system

{% hint style="info" %}
cd                      #change directory
{% endhint %}

{% hint style="info" %}
pwd                  # Print working directory
{% endhint %}

{% hint style="info" %}
ls                      # List files and directories
{% endhint %}

{% hint style="info" %}
ls -l                   # List with details
{% endhint %}

{% hint style="info" %}
ls -ltr                 # List by time (oldest to newest)
{% endhint %}



### **File system structure**

/boot -> Contains files used by the bootloader (e.g. grub.cfg)

/root -> Home directory for the root user

/dev -> System devices (disks, CD-ROM, USB, keyboard, etc.)

/etc -> Configuration files (app configs — always backup!)

/tmp -> Temporary files directory

/home -> User home directories

/var -> System logs

/run -> Temporary runtime files like PID files

/mnt -> Mount external filesystems (e.g. NFS)

/media -> Auto-mounted devices like CD-ROMs

/boot -> Boot config directory

/sbin -> System binaries (commands)

/bin -> User commands

/opt -> Optional add-on software (not part of core OS)

/proc -> Virtual filesystem showing running processes

/lib -> C programming libraries used by apps/commands



### **Absolute vs Relative Paths**

Absolute path: starts from root /

Ex: /var/log/samba

Relative path: starts from current position

Ex: cd ../log or cd log



### Linux File Types (with ls -l)

{% columns %}
{% column %}
**Symbol**

**-**

**d**

**l**

**c**

**s**

**p**

**b**
{% endcolumn %}

{% column %}
**Type**

Regular file (text, image, etc).

Directory

Symbolic link

Character device file

Socket

Named pipe (FIFO)

Block device
{% endcolumn %}
{% endcolumns %}
