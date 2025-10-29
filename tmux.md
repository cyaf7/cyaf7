---
description: >-
  stands for terminal multiplexer — it lets you have multiple terminal sessions
  inside a single window. Think of it as a window manager for your terminal.
---

# tmux

With tmux you can:

* Split your terminal into multiple **panes** (see them all at once)
* Switch between panes with simple shortcuts
* Resize panes
* Leave a session running in the background and **reattach later**
* Organize your work without opening multiple terminal windows

to instal it:

{% hint style="info" %}
sudo apt update\
sudo apt install tmux
{% endhint %}

To check if it’s installed:

{% hint style="info" %}
tmux -V
{% endhint %}

To start it, just type:

{% hint style="info" %}
tmux
{% endhint %}

You’ll notice a **green status bar** at the bottom — you’re now inside a `tmux` session.

| Action           | Keys                  |
| ---------------- | --------------------- |
| Start tmux       | `tmux`                |
| Vertical split   | `Ctrl+b %`            |
| Horizontal split | `Ctrl+b "`            |
| Switch panes     | `Ctrl+b` + Arrow keys |
| Resize pane      | `Ctrl+b` + hold Arrow |
| Close pane       | `Ctrl+b x`            |
| Detach session   | `Ctrl+b d`            |
| Reattach session | `tmux attach`         |
| List sessions    | `tmux ls`             |

In each pane, run:

* Pane 1: `top`
* Pane 2: `ping google.com`
* Pane 3: `tail -f /var/log/syslog`

Now you have a **real-time dashboard** in one terminal window
