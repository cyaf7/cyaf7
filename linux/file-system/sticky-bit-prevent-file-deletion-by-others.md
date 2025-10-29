# Sticky Bit — Prevent file deletion by others

Prevents users from deleting each other’s files in shared folders like /tmp.

{% hint style="info" %}
chmod +t /shared-folder
{% endhint %}

Only the file owner or root can delete the file now, /tmp has the sticky bit to avoid users deleting others’ temp files.
