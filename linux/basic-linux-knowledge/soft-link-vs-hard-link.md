# Soft link vs Hard link

inode: internal number that identifies files (computers don’t use names, only inodes)

**Soft Link (symbolic):**&#x20;

* Acts like a shortcut.
* If the original file is deleted, the link is broken.

**Hard Link:**

* Points directly to the file's data.
* Deleting the original file doesn’t break the link.

{% hint style="info" %}
ln fileA fileB                       # Create hard link
{% endhint %}

{% hint style="info" %}
ln -s fileA fileB                 # Create soft (symbolic) link
{% endhint %}

* you **can't create a hard or soft link with the same name** in the same directory.
* That’s why links are often created inside /tmp.
* hard links must stay within the same filesystem.

example:

{% hint style="info" %}
echo "hello from hardlink" > /hardfile.txt

ln /hardfile.txt /home/tmp/myhardlink.txt
{% endhint %}

then:

{% hint style="info" %}
cat /home/tmp/myhardlink.txt
{% endhint %}

and you'll have your output even if you remove the original file.

<figure><img src="../../.gitbook/assets/image (3).png" alt="" width="375"><figcaption></figcaption></figure>

#### **To remove Soft/Hard link:**

{% hint style="info" %}
rm /home/tmp/myhardlink.txt
{% endhint %}



