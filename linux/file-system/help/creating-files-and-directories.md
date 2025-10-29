# Creating files and directories

touch -> creates an empty file

{% hint style="info" %}
touch filename
{% endhint %}

You can create multiple files at once using:

{% hint style="info" %}
touch filename1 filename2 filename3
{% endhint %}

vi -> editor mode (I'll further discuss it later)

{% hint style="info" %}
vi filename
{% endhint %}

mkdir -> creates a new directory

{% hint style="info" %}
mkdir directoryname
{% endhint %}

cp -> copy file and rename it

{% hint style="info" %}
cp filename newfilename
{% endhint %}

this copies notes.txt into the Documents folder.

{% hint style="info" %}
cp notes.txt /home/user/Documents/
{% endhint %}

This copies the entire myFolder (directory) and all its contents into the Backups folder. The -R is for recursive (copies directories and their contents).

{% hint style="info" %}
cp -R myFolder /home/user/Backups/
{% endhint %}



**Command Syntax:**

{% hint style="info" %}
command \[options] \[arguments]
{% endhint %}

Here the directory /boot is being listed:

{% hint style="info" %}
ls -l /boot
{% endhint %}

Removing file (-f for forceful deletion)

{% hint style="info" %}
rm -f file
{% endhint %}

Removing directory and everything in it:

{% hint style="info" %}
rm -r directory
{% endhint %}



