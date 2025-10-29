# Process / Service Commands

Helps monitor the system, find processes, and kill suspicious ones (useful in incident response).

{% columns %}
{% column width="41.66666666666667%" %}
command

ps -e

ps -ef

ps -u username

kill PID

TTY



PID


{% endcolumn %}

{% column %}
what it does

Shows all running processes&#x20;

Full detailed list of processes

Shows only that user’s processes

Kills a process by ID

Shows terminal attached to each process

Process ID (used for targeting processes)
{% endcolumn %}
{% endcolumns %}

