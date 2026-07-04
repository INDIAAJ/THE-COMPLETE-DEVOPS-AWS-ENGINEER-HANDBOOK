CHAPTER 6: PROCESS MANAGEMENT, SERVICES, systemctl, journalctl, cron

6.1 Theory

A process is a running instance of a program. Every process has a unique Process ID (PID), a parent process (PPID), and exists in a particular state (running, sleeping, stopped, zombie). Modern Linux distributions use systemd as the init system — the very first process (PID 1) that starts at boot and manages all other services thereafter.

6.2 Why It Exists

Servers need to run background services reliably — a web server (Nginx) must start automatically on boot, restart if it crashes, and be easy to stop/start/check status on demand. systemd and its command systemctl standardize this across virtually all modern distros (replacing older, inconsistent init systems like SysVinit and Upstart).

6.3 Process Management Commands

bashps aux                    # Show ALL running processes, all users, detailed format
ps -ef                    # Alternative full-format process listing
ps aux | grep nginx        # Find a specific process
pstree                     # Show processes as a parent-child tree
kill <PID>                 # Gracefully terminate a process (sends SIGTERM)
kill -9 <PID>               # Force kill a process (sends SIGKILL, cannot be ignored)
killall nginx                # Kill all processes by name
pkill -f "python app.py"     # Kill processes matching a command pattern
nice -n 10 command            # Start a process with lower priority
renice -n 5 -p <PID>           # Change priority of a running process
jobs                          # List background jobs in current shell
bg                            # Resume a job in the background
fg                            # Bring a background job to foreground
command &                     # Run a command in the background immediately
nohup command &                # Run a command immune to hangups (survives terminal close)

Sample Output of ps aux:

USER   PID  %CPU %MEM    VSZ   RSS TTY   STAT START   TIME COMMAND
root     1   0.0  0.1 168000 11200 ?     Ss   09:00   0:02 /sbin/init
www-data 892 0.1  0.5  55000  8900 ?     S    09:05   0:01 nginx: worker process
ubuntu  1203 0.0  0.0  17800  3300 pts/0 R+   10:22   0:00 ps aux

Key columns: PID (process ID), %CPU/%MEM (resource usage), STAT (S=sleeping, R=running, Z=zombie, T=stopped), COMMAND (what's running).

6.4 Understanding Signals

SignalNumberMeaningSIGTERM15Default kill signal — polite request to terminate, allows cleanupSIGKILL9Force kill — cannot be caught or ignored, no cleanupSIGHUP1Hang up — often used to tell a service to reload its configSIGSTOP19Pause a processSIGCONT18Resume a paused process

6.5 systemctl — Managing Services

systemctl is your primary tool for controlling services (also called "units") on any systemd-based distro (Ubuntu 16.04+, RHEL 7+, Amazon Linux 2+).

bashsudo systemctl start nginx        # Start a service now
sudo systemctl stop nginx          # Stop a service now
sudo systemctl restart nginx        # Stop then start (brief downtime)
sudo systemctl reload nginx         # Reload config WITHOUT dropping connections (if supported)
sudo systemctl status nginx         # Check current status, recent logs, PID
sudo systemctl enable nginx         # Auto-start this service on every future boot
sudo systemctl disable nginx        # Do NOT auto-start on boot
sudo systemctl enable --now nginx   # Enable AND start in one command
systemctl is-active nginx            # Quick check: active or not
systemctl is-enabled nginx           # Quick check: enabled on boot or not
systemctl list-units --type=service  # List all currently loaded services
systemctl daemon-reload              # Reload systemd config after editing a .service file

Sample Output of systemctl status nginx:

● nginx.service - A high performance web server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2026-07-01 09:05:12 UTC; 2h 10min ago
       Docs: man:nginx(8)
   Main PID: 892 (nginx)
      Tasks: 3 (limit: 1137)
     Memory: 5.2M
        CPU: 156ms
     CGroup: /system.slice/nginx.service
             ├─892 nginx: master process /usr/sbin/nginx
             └─894 nginx: worker process

The green ● and "active (running)" confirm the service is healthy. This exact command is what you'd run in the scenario from Chapter 1: "check why Nginx is down."

6.6 Creating Your Own systemd Service

This is a critical real-world DevOps skill — running your own application (e.g., a Python/Node app) as a proper managed service.

bashsudo nano /etc/systemd/system/myapp.service

ini[Unit]
Description=My Custom Application
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/myapp
ExecStart=/usr/bin/python3 /home/ubuntu/myapp/app.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target

bashsudo systemctl daemon-reload
sudo systemctl enable --now myapp
sudo systemctl status myapp

6.7 journalctl — Viewing systemd Logs

journalctl reads the systemd journal, a centralized binary log covering the kernel and every systemd-managed service.

bashjournalctl -u nginx               # Logs for a specific service (unit)
journalctl -u nginx -f             # Follow logs in real time (like tail -f)
journalctl -u nginx --since today   # Logs since midnight today
journalctl -u nginx --since "1 hour ago"
journalctl -p err                   # Only show error-priority and above
journalctl -b                       # Logs since last boot
journalctl -k                       # Kernel messages only
journalctl --disk-usage             # How much space the journal is using
sudo journalctl --vacuum-time=7d     # Delete logs older than 7 days

6.8 cron — Task Scheduling

cron runs commands automatically on a schedule — essential for backups, cleanup scripts, health checks, and automated deployments.

bashcrontab -e            # Edit YOUR crontab (opens in default editor)
crontab -l              # List your current cron jobs
crontab -r              # Remove all your cron jobs
sudo crontab -e -u www-data   # Edit another user's crontab (as root)

Cron Syntax:

 ┌───────────── minute (0 - 59)
 │ ┌───────────── hour (0 - 23)
 │ │ ┌───────────── day of month (1 - 31)
 │ │ │ ┌───────────── month (1 - 12)
 │ │ │ │ ┌───────────── day of week (0 - 6) (Sunday=0)
 │ │ │ │ │
 * * * * *  command-to-execute

Real Examples:

bash# Run a backup script every day at 2:30 AM
30 2 * * * /home/ubuntu/scripts/backup.sh

# Restart an app every Monday at 4 AM
0 4 * * 1 systemctl restart myapp

# Run every 15 minutes
*/15 * * * * /home/ubuntu/scripts/healthcheck.sh

# Run every Sunday at midnight
0 0 * * 0 /home/ubuntu/scripts/weekly_cleanup.sh

# Run at system reboot
@reboot /home/ubuntu/scripts/startup.sh

Always redirect output for debugging:

bash30 2 * * * /home/ubuntu/scripts/backup.sh >> /home/ubuntu/logs/backup.log 2>&1

6.9 Architecture Diagram — systemd Boot & Service Flow

   Power On
      │
      ▼
   BIOS/UEFI → GRUB Bootloader → Kernel loads
      │
      ▼
   systemd (PID 1) starts
      │
      ├──► Reads /etc/systemd/system/*.service (enabled units)
      │
      ├──► Starts network.target
      │
      ├──► Starts nginx.service ──► journalctl logs everything
      │
      ├──► Starts docker.service
      │
      └──► Starts sshd.service ──► SSH now accepts connections

6.10 Best Practices


Always use systemctl enable --now for services that must survive a reboot (extremely common EC2 gotcha: service works, then instance reboots, and the app is "down" because it wasn't enabled).
Use Restart=always in custom .service files so your app self-heals after a crash.
Redirect cron job output to a log file — silent cron failures are one of the most common production issues.
Use journalctl -u <service> -f as your first troubleshooting step for any "service won't start" issue.


6.11 Common Mistakes


Starting a service with systemctl start but forgetting systemctl enable, so it doesn't come back after a reboot.
Writing cron schedules with typos and not testing them (use crontab.guru-style tools or simply test the command manually first).
Assuming kill (SIGTERM) always works — some misbehaving processes need kill -9 (SIGKILL).
Forgetting sudo systemctl daemon-reload after editing/creating a .service file — systemd won't pick up changes otherwise.


6.12 Interview Questions


What is the difference between systemctl restart and systemctl reload?
What does systemctl enable actually do, versus systemctl start?
How would you view the last hour of logs for a specific service?
Explain the difference between SIGTERM and SIGKILL.
Write a cron expression to run a script every day at 3:45 AM.
How do you create a custom systemd service for your own application?
What is a zombie process, and how would you identify one?


6.13 Scenario-Based Question

Scenario: You deployed your Flask application on an EC2 instance using a systemd service. It works fine, but after rebooting the instance for a security patch, the application is unreachable. Diagnose and fix this.

(Approach: systemctl status myapp → likely shows "inactive (dead)" → check if it was enabled: systemctl is-enabled myapp → if disabled, run sudo systemctl enable --now myapp → verify with curl localhost:5000 and check journalctl -u myapp for startup errors.)

6.14 Troubleshooting


Issue: Service shows "failed" status.
Fix: journalctl -u <service> -n 50 --no-pager to see the last 50 log lines and identify the actual error (missing dependency, wrong file path, port already in use, etc.).
Issue: Cron job doesn't run.
Fix: Check /var/log/syslog (Ubuntu) or /var/log/cron (RHEL) for cron execution logs; verify absolute paths are used inside the script/command; check crontab -l for typos.


6.15 Cheat Sheet

bashps aux | grep <name>              # Find a process
kill -9 <PID>                      # Force kill
systemctl start|stop|restart <svc> # Control a service
systemctl enable --now <svc>       # Enable + start
systemctl status <svc>             # Check status
journalctl -u <svc> -f              # Follow live logs
journalctl -u <svc> --since today   # Today's logs
crontab -e                          # Edit cron jobs
crontab -l                          # List cron jobs
* * * * * command                   # min hour day month weekday

6.16 Summary

Process and service management is where "Linux knowledge" becomes "production DevOps skill." systemctl controls the lifecycle of every service on your server, journalctl is your primary debugging lens into what went wrong, and cron automates recurring tasks. Expect hands-on questions about all three in nearly every DevOps interview.
