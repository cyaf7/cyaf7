# systemctl — Control system services

To **start, stop, enable, disable**, and **check the status** of system services (like firewall, SSH, cron).

{% hint style="info" %}
systemctl start firewalld.service    # Start a service\
systemctl stop firewalld.service    # Stop a service\
systemctl status firewalld.service    # Check its status\
systemctl enable sshd.service     # Start at boot\
systemctl restart sshd.service    # Restart the service\
systemctl list-units --all    # List all services
{% endhint %}

To add a service manually, create a unit file in:

{% hint style="info" %}
/etc/systemd/system/servicename.service
{% endhint %}

To reboot or shut down:

{% hint style="info" %}
systemctl reboot\
systemctl poweroff
{% endhint %}

