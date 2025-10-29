---
description: >-
  This section explains how to create, modify, and control users and groups so
  you can enforce security policies, share resources, and troubleshoot
  permissions.
---

# user/group management

In Linux:

* **User** = an account that can log into the system.
* **Group** = a collection of users that share permissions.
* Each file/directory has:
  * **User owner**
  * **Group owner**
  * **Permission set** for owner, group, and others.

Important files:

* /etc/passwd → stores users and their UIDs
* /etc/group → stores groups and their GIDs
* /etc/shadow → stores password hashes (only root can read)

### Creating Users

{% hint style="info" %}
##

## <sup><sub>Create a user with a home directory<sub></sup>

**sudo useradd -m alice**

## <sup><sub>create with a specific shell<sub></sup>

**sudo useradd -m -s /bin/bash alice**

## <sup><sub>Create with a primary group<sub></sup>

**sudo useradd -m -s /bin/bash -g developers alice**
{% endhint %}

-m is important — it creates /home/Alice automatically.\
Without it, the user will not have a writable home directory.

**To set a password:**

{% hint style="info" %}
sudo passwd alice
{% endhint %}

If you forgot -m, you'll have to create the home directory with

{% hint style="info" %}
sudo mkdir /home/alice\
sudo chown alice:alice /home/alice\
sudo chmod 700 /home/alice
{% endhint %}

### Modifying users:

#### Change username

{% hint style="info" %}
sudo usermod -l newname oldname
{% endhint %}

#### Change shell

{% hint style="info" %}
sudo usermod -s /bin/zsh alice
{% endhint %}

#### Lock / unlock account

{% hint style="info" %}
sudo usermod -L alice sudo usermod -U alice
{% endhint %}

### **Changing User ID (UID):**

**check it with :**

{% hint style="info" %}
id username
{% endhint %}

Change UID:

{% hint style="info" %}
sudo usermod -u 1050 alice
{% endhint %}

