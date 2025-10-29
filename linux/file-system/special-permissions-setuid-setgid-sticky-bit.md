# Special Permissions — setuid, setgid, sticky bit

Control what users or groups can do, especially in **shared folders or scripts**.

setuid — Run file with owner's privileges:

{% hint style="info" %}
chmod u+s file
{% endhint %}

setgid — New files inherit group:

{% hint style="info" %}
chmod g+s folder
{% endhint %}

To remove:

{% hint style="info" %}
chmod u-s file\
chmod g-s file
{% endhint %}

In shared folders like /home/projects/**design**, files stay with the **group design** even if others add content.
