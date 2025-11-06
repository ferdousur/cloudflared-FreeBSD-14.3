# cloudflared-FreeBSD-14.3
Document how to run a Cloudflare Tunnel (cloudflared) on FreeBSD. Includes a ready rc.d script (root-only) and recommended best practices.
Cloudflared Tunnel Setup Guide (FreeBSD)
Step 1: Install Cloudflared
pkg update
pkg install cloudflared

Step 2: Login and Create Tunnel
cloudflared login
cloudflared tunnel create <Tunnel name>
After successful login, Cloudflared will:
    • Automatically create a certificate file (cert.pem)
    • Generate a tunnel JSON credentials file
Both will be stored in:


Step 3: Create Configuration File
Create and edit the config file to following directories : 
cd .cloudflared
create config.yml 
nano config.yml
Paste the following content (replace placeholders accordingly):
tunnel: <Tunnel Name> # add here tunnel name please
credentials-file: /root/.cloudflared/<Tunnel ID>.json #add here tunnel id please
origincert: /root/.cloudflared/cert.pem

ingress:
  - hostname: domain.com #your domain
    service: http://localhost:80 #your service on localhost
  - service: http_status:404





Step 4: Setup DNS Route
cloudflared tunnel route dns <Tunnelid> domain.com

Step 5: Manual Run (Test Tunnel)
You can manually test the tunnel with:
cloudflared --config /root/.cloudflared/config.yml tunnel run testtunnel
If it works properly, proceed to the auto-start configuration.

Step 6: Create Service Script
Go to the rc.d directory:
cd /usr/local/etc/rc.d
nano cloudflared
Paste the following full script:
#!/bin/sh
#
# PROVIDE: cloudflared
# REQUIRE: LOGIN
# KEYWORD: shutdown

. /etc/rc.subr

name="cloudflared"
rcvar="${name}_enable"

# Default values
: ${cloudflared_enable:="NO"}
: ${cloudflared_user:="root"}                       # run as root
: ${cloudflared_bin:="/usr/local/bin/cloudflared"}
: ${cloudflared_config:=/root/.cloudflared/config.yml}  # point to your config (adjust if needed)
: ${cloudflared_pidfile:="/var/run/cloudflared/cloudflared.pid"}
: ${cloudflared_logfile:="/var/log/cloudflared.log"}

start_precmd()
{
    # Check binary and config
    [ -x "${cloudflared_bin}" ] || { echo "cloudflared binary not found"; return 1; }
    [ -f "${cloudflared_config}" ] || { echo "cloudflared config not found: ${cloudflared_config}"; return 1; }

    # Ensure directories exist (owned by root)
    mkdir -p /var/run/cloudflared
    chown root:wheel /var/run/cloudflared
    chmod 0755 /var/run/cloudflared

    mkdir -p $(dirname "${cloudflared_logfile}")
    touch "${cloudflared_logfile}"
    chown root:wheel "${cloudflared_logfile}"
    chmod 0640 "${cloudflared_logfile}"
}

start_cmd="${name}_start"
stop_cmd="${name}_stop"
status_cmd="${name}_status"

cloudflared_start()
{
    start_precmd || return 1
    echo "Starting cloudflared in background ..."
    /usr/sbin/daemon -f -u ${cloudflared_user} \
        -p ${cloudflared_pidfile} \
        -o ${cloudflared_logfile} \
        ${cloudflared_bin} --config ${cloudflared_config} tunnel run <Tunnel Name>   #add you tunnel name here please
}

cloudflared_stop()
{
    if [ -f "${cloudflared_pidfile}" ]; then
        echo "Stopping cloudflared..."
        kill "$(cat ${cloudflared_pidfile})" 2>/dev/null || true
        rm -f "${cloudflared_pidfile}"
    else
        echo "cloudflared not running"
    fi
}

cloudflared_status()
{
    if [ -f "${cloudflared_pidfile}" ]; then
        echo "cloudflared running, PID=$(cat ${cloudflared_pidfile})"
    else
        echo "cloudflared not running"
    fi
}

load_rc_config "$name"
run_rc_command "$1"

Save and exit (CTRL + O, then CTRL + X).




Step 7: Enable and Start Service
Now execute these commands one by one:



sysrc cloudflared_enable="YES"
service cloudflared start

Step 8: Manage the Service
    • Start Cloudflared:
      service cloudflared start
    • Stop Cloudflared:
      service cloudflared stop
    • Check Status:
      service cloudflared status
    • View Live http log:
      cloudflared tail <tunnel id>

