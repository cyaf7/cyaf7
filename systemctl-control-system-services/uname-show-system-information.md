# uname - Show System Information

**Basic usage**

{% hint style="info" %}
uname
{% endhint %}

Show All Information:

{% hint style="info" %}
uname -a
{% endhint %}

Example output:

{% hint style="info" %}
Linux aoki 5.15.0-86-generic #96-Ubuntu SMP x86\_64 GNU/Linux
{% endhint %}

Breakdown:

* **Linux** → kernel name
* **aoki** → hostname
* **5.15.0-86-generic** → kernel version
* **#96-Ubuntu SMP** → build info
* **x86\_64** → architecture
* **GNU/Linux** → OS type

| Option | Meaning               | Example output     |
| ------ | --------------------- | ------------------ |
| `-s`   | Kernel name           | Linux              |
| `-n`   | Network hostname      | aoki               |
| `-r`   | Kernel release        | 5.15.0-86-generic  |
| `-v`   | Kernel version        | #96-Ubuntu SMP ... |
| `-m`   | Machine hardware name | x86\_64            |
| `-o`   | Operating system      | GNU/Linux          |
| `-p`   | Processor type        | x86\_64            |
| `-i`   | Hardware platform     | x86\_64            |
