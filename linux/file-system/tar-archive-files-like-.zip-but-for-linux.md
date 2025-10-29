# tar — Archive files (like .zip but for Linux)

Puts many files/folders together into one `.tar` file (doesn’t compress unless used with gzip).

you create the tar.zip file wherever directory you're inside.

{% hint style="info" %}
sudo tar -cvf camilly.tar.zip /home/camilly/fileIwanttocompress .

sudo tar czvf camilly.tar.zip fileIwanttocompress&#x20;
{% endhint %}

&#x20;C -> Create\
V -> Verbose (show process)\
F -> File name&#x20;

Move it to a new folder:

{% hint style="info" %}
mkdir dir\
mv camilly.tar dir
{% endhint %}

Extract the archive:

{% hint style="info" %}
tar xvf camilly.tar
{% endhint %}

X -> extract
