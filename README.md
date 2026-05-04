
# Cloudflared Tunnel Setup Guide (FreeBSD)

This guide explains how to install and configure Cloudflared Tunnel on FreeBSD to securely expose a local service to the internet.

---

## 📦 Installation

Update package list and install Cloudflared:

```bash
pkg update
pkg install cloudflared
```

---

## 🔐 Login & Create Tunnel

Authenticate Cloudflared and create a new tunnel:

```bash
cloudflared login
cloudflared tunnel create <Tunnel-Name>
```

After successful login:

* `cert.pem` will be created
* Tunnel credentials (`<Tunnel-ID>.json`) will be generated

📁 Stored in:

```
/root/.cloudflared/
```

---

## ⚙️ Configuration

Navigate to Cloudflared directory:

```bash
cd /root/.cloudflared
nano config.yml
```

Paste the following configuration:

```yaml
tunnel: <Tunnel-Name>
credentials-file: /root/.cloudflared/<Tunnel-ID>.json
origincert: /root/.cloudflared/cert.pem

ingress:
  - hostname: domain.com
    service: http://localhost:80
  - service: http_status:404
```

Replace:

* `<Tunnel-Name>` with your tunnel name
* `<Tunnel-ID>` with your generated tunnel ID
* `domain.com` with your actual domain

---

## 🌐 DNS Routing

Route your domain to the tunnel:

```bash
cloudflared tunnel route dns <Tunnel-ID> domain.com
```

---

## 🧪 Test the Tunnel

Run the tunnel manually to verify:

```bash
cloudflared --config /root/.cloudflared/config.yml tunnel run <Tunnel-Name>
```

If everything works, proceed to service setup.

---

## 🔧 Create rc.d Service

Navigate to rc.d directory:

```bash
cd /usr/local/etc/rc.d
nano cloudflared
```

Paste the following script:

```sh
#!/bin/sh
#
# PROVIDE: cloudflared
# REQUIRE: LOGIN
# KEYWORD: shutdown

. /etc/rc.subr

name="cloudflared"
rcvar="${name}_enable"

: ${cloudflared_enable:="NO"}
: ${cloudflared_user:="root"}
: ${cloudflared_bin:="/usr/local/bin/cloudflared"}
: ${cloudflared_config:="/root/.cloudflared/config.yml"}
: ${cloudflared_pidfile:="/var/run/cloudflared/cloudflared.pid"}
: ${cloudflared_logfile:="/var/log/cloudflared.log"}

start_precmd()
{
    [ -x "${cloudflared_bin}" ] || { echo "cloudflared binary not found"; return 1; }
    [ -f "${cloudflared_config}" ] || { echo "config not found"; return 1; }

    mkdir -p /var/run/cloudflared
    chown root:wheel /var/run/cloudflared
    chmod 0755 /var/run/cloudflared

    touch "${cloudflared_logfile}"
    chown root:wheel "${cloudflared_logfile}"
    chmod 0640 "${cloudflared_logfile}"
}

start_cmd="${name}_start"
stop_cmd="${name}_stop"
status_cmd="${name}_status"

cloudflared_start()
{
    echo "Starting cloudflared..."
    /usr/sbin/daemon -f -u ${cloudflared_user} \
        -p ${cloudflared_pidfile} \
        -o ${cloudflared_logfile} \
        ${cloudflared_bin} --config ${cloudflared_config} tunnel run <Tunnel-Name>
}

cloudflared_stop()
{
    if [ -f "${cloudflared_pidfile}" ]; then
        kill "$(cat ${cloudflared_pidfile})"
        rm -f "${cloudflared_pidfile}"
        echo "Stopped."
    else
        echo "Not running."
    fi
}

cloudflared_status()
{
    if [ -f "${cloudflared_pidfile}" ]; then
        echo "Running PID=$(cat ${cloudflared_pidfile})"
    else
        echo "Not running."
    fi
}

load_rc_config "$name"
run_rc_command "$1"
```

Make it executable:

```bash
chmod +x /usr/local/etc/rc.d/cloudflared
```

---

## 🚀 Enable & Start Service

```bash
sysrc cloudflared_enable="YES"
service cloudflared start
```

---

## 🛠️ Service Management

* Start:

```bash
service cloudflared start
```

* Stop:

```bash
service cloudflared stop
```

* Status:

```bash
service cloudflared status
```

---

## 📊 View Logs

Check live logs:

```bash
tail -f /var/log/cloudflared.log
```

Or use:

```bash
cloudflared tail <Tunnel-ID>
```

---

## ⚠️ Notes

* Ensure your domain is managed by Cloudflare
* Port `80` must be accessible locally
* Verify firewall rules if connection fails

---

## 📌 Summary

This setup allows you to:

* Expose local services securely
* Run Cloudflared as a background service
* Automatically start tunnel on system boot

---

## 📄 License

Free to use for educational and production purposes.
