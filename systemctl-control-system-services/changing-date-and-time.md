# Changing Date & Time

There are two main ways:&#x20;

Using the **date** command (quick, temporary)&#x20;

Using **timedatectl** (systemd-based, permanent & recommended)



to list a date:

{% hint style="info" %}
date -d "1999-06-7"
{% endhint %}

to list a calendar:

{% hint style="info" %}
cal 1999
{% endhint %}

### Change Date & Time with date (Temporary):

{% hint style="info" %}
sudo date -s "2025-08-09 15:30:00"
{% endhint %}

* `-s` → set date/time
* Format: `"YYYY-MM-DD HH:MM:SS"`

check with: date

This change lasts until the next reboot unless you also sync it to the hardware clock:

{% hint style="info" %}
sudo hwclock --systohc
{% endhint %}

### **Change Date & Time with `timedatectl` (Permanent):**

#### View current date/time and settings

{% hint style="info" %}
timedatectl
{% endhint %}

#### Set date

{% hint style="info" %}
sudo timedatectl set-time "2025-08-09"
{% endhint %}

#### Set time

{% hint style="info" %}
sudo timedatectl set-time "15:30:00"
{% endhint %}

#### Set both date & time

{% hint style="info" %}
sudo timedatectl set-time "2025-08-09 15:30:00"
{% endhint %}

Change Time Zone:

#### List all time zones

{% hint style="info" %}
timedatectl list-timezones
{% endhint %}

#### Set time zone

{% hint style="info" %}
sudo timedatectl set-timezone Europe/London
{% endhint %}
