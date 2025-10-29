# Creating & Managing Groups

### Create a group

{% hint style="info" %}
sudo groupadd developers
{% endhint %}

### Add user to group

{% hint style="info" %}
sudo usermod -aG developers alice
{% endhint %}

### Remove user from group

{% hint style="info" %}
sudo gpasswd -d alice developers
{% endhint %}

### Delete group

{% hint style="info" %}
sudo groupdel developers
{% endhint %}

### **Shared Writable Directory for a Group:**

{% hint style="info" %}
sudo mkdir /shared

\
sudo chown :developers /shared

\
sudo chmod 2775 /shared
{% endhint %}

* `chown :developers` → group owns folder
* `chmod 2775` → setgid bit makes new files inherit group

test:

su - alice\
cd /shared\
touch test.txt

