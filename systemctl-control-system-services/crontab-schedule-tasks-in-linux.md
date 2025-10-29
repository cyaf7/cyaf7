# CRONTAB — Schedule Tasks in Linux

Automate tasks like running backups, checking logs, or sending alerts.

{% hint style="info" %}
crontab -e # Edit cron jobs\
crontab -l # List cron jobs\
crontab -r # Remove cron jobs
{% endhint %}

Crontab Syntax:

{% hint style="info" %}
minute hour day month day-of-week command
{% endhint %}

Run a script every day at 3:30 PM:

{% hint style="info" %}
30 15 \* \* \* /path/to/script.sh
{% endhint %}

in /etc/cron.daily/script

Run a one-time task:

{% hint style="info" %}
echo "echo Hello" | crontab -
{% endhint %}

You can also place scripts in:

/etc/cron.hourly/

/etc/cron.daily/

/etc/cron.weekly/

/etc/cron.monthly/

Example:

**30 15 24 \* \* echo "task" -> runs at 3:30 PM on the 24th of each month**

Run daily:

Put your script in /etc/cron.daily

