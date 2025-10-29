# Switch Users & Sudo Access

Give a user **root-like access** securely or switch between users.

**Switch to another user:**

{% hint style="info" %}
su - username
{% endhint %}

**Use root privileges:**

{% hint style="info" %}
sudo command
{% endhint %}

**Give a user sudo access:**

{% hint style="info" %}
usermod -aG wheel username
{% endhint %}

**Check with:**

{% hint style="info" %}
grep wheel /etc/group
{% endhint %}

