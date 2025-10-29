# Changing Ownership

### Change user owner

{% hint style="info" %}
sudo chown bob file.txt
{% endhint %}

### Change group owner

{% hint style="info" %}
sudo chgrp developers file.txt
{% endhint %}

### Change both

{% hint style="info" %}
sudo chown bob:developers file.txt
{% endhint %}

### Recursive change

{% hint style="info" %}
sudo chown -R bob:developers /project
{% endhint %}

to root:

{% hint style="info" %}
sudo chown root:root file.txt
{% endhint %}

### **Giving Sudo Access:**

#### Debian/Ubuntu

{% hint style="info" %}
sudo usermod -aG sudo alice
{% endhint %}

#### RHEL/CentOS/Fedora

{% hint style="info" %}
sudo usermod -aG wheel alice
{% endhint %}
