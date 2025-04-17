# Dell iDRAC Monitoring with Telegraf, InfluxDB, and Grafana

This guide provides step-by-step instructions for setting up a monitoring system for Dell iDRAC interfaces using Telegraf, InfluxDB, and Grafana on Rocky Linux.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Setting Up InfluxDB](#setting-up-influxdb)
3. [Setting Up Telegraf](#setting-up-telegraf)
4. [Setting Up Grafana](#setting-up-grafana)
5. [Configuring SNMP on iDRAC](#configuring-snmp-on-idrac)
6. [Verification and Troubleshooting](#verification-and-troubleshooting)

## Prerequisites

- Rocky Linux VM
- Dell iDRAC(s) with network access
- Root or sudo access on the Rocky Linux VM
- iDRAC admin credentials

## Setting Up InfluxDB

### Install InfluxDB
```bash
# Download the InfluxDB RPM
wget https://dl.influxdata.com/influxdb/releases/influxdb-1.8.10.x86_64.rpm

# Install with rpm
sudo rpm -ivh --nodeps ./influxdb-1.8.10.x86_64.rpm
```

### Create the systemd service file
```bash
sudo tee /etc/systemd/system/influxdb.service > /dev/null << EOF
[Unit]
Description=InfluxDB is an open-source, distributed, time series database
Documentation=https://docs.influxdata.com/influxdb/
After=network-online.target

[Service]
User=influxdb
Group=influxdb
LimitNOFILE=65536
ExecStart=/usr/bin/influxd -config /etc/influxdb/influxdb.conf
KillMode=control-group
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

### Create configuration directory and file if not already present
```bash
sudo mkdir -p /etc/influxdb
```

If the configuration file doesn't exist, create it:
```bash
sudo tee /etc/influxdb/influxdb.conf > /dev/null << EOF
[meta]
  dir = "/var/lib/influxdb/meta"

[data]
  dir = "/var/lib/influxdb/data"
  wal-dir = "/var/lib/influxdb/wal"

[http]
  enabled = true
  bind-address = ":8086"
EOF
```

### Create the influxdb user and directories
```bash
sudo useradd -r -d /var/lib/influxdb -s /bin/false influxdb
sudo mkdir -p /var/lib/influxdb/{meta,data,wal}
sudo chown -R influxdb:influxdb /var/lib/influxdb
```

### Start and enable InfluxDB
```bash
sudo systemctl daemon-reload
sudo systemctl start influxdb
sudo systemctl enable influxdb
sudo systemctl status influxdb
```

### Create the database for iDRAC monitoring
```bash
influx -execute "CREATE DATABASE idrac_monitoring"
```
*Note: Alternatively, you can edit directly in telegraf.conf file by adding host IP address, port, database name, user, and password.*

## Setting Up Telegraf

### Install Telegraf
```bash
# Install the SNMP utilities first
sudo dnf install -y net-snmp net-snmp-utils

# Download and install Telegraf
wget https://dl.influxdata.com/telegraf/releases/telegraf-1.24.0-1.x86_64.rpm
sudo rpm -ivh telegraf-1.24.0-1.x86_64.rpm
```

### Configure Telegraf
Create the iDRAC input configuration file:
```bash
sudo mkdir -p /etc/telegraf/telegraf.d
sudo vi /etc/telegraf/telegraf.d/idrac-input.conf
```

Add the following configuration:
```toml
[[processors.regex]]
  [[processors.regex.fields]]
    key = "log-dates"
    pattern = "^(?P<YYYY>\\d{4})(?P<MM>\\d{2})(?P<DD>\\d{2})(?P<HH>\\d{2})(?P<mm>\\d{2})(?P<ss>\\d{2})\\.(?P<SSSSSS>\\d{6})(?P<ZZ>[-+]\\d{3,4})$"
    replacement = "${YYYY}-${MM}-${DD} ${HH}:${mm}:${ss}"

[[inputs.snmp]]
  agents = [ "idracURL1:161" , "idracURL2:161" , "idracURL3:161" ] # REPLACE WITH YOUR IDRAC IP ADDRESSES
  version = 1
  community = "public"
  name = "idrac-hosts"

  [[inputs.snmp.field]]
     name = "system-name"
     oid  = ".1.3.6.1.2.1.1.5.0"
     is_tag = true

  [[inputs.snmp.field]]
     name = "system-osname"
     oid  = ".1.3.6.1.4.1.674.10892.5.1.3.6.0"

  [[inputs.snmp.field]]
     name = "system-osversion"
     oid  = ".1.3.6.1.4.1.674.10892.5.1.3.14.0"

  [[inputs.snmp.field]]
     name = "system-model"
     oid  = ".1.3.6.1.4.1.674.10892.5.1.3.12.0"

  [[inputs.snmp.field]]
     name = "idrac-url"
     oid  = ".1.3.6.1.4.1.674.10892.5.1.1.6.0"

  [[inputs.snmp.field]]
     name = "power-state"
     oid  = ".1.3.6.1.4.1.674.10892.5.2.4.0"

  [[inputs.snmp.field]]
     name = "system-uptime"
     oid  = ".1.3.6.1.4.1.674.10892.5.2.5.0"

  [[inputs.snmp.field]]
     name = "system-servicetag"
     oid  = ".1.3.6.1.4.1.674.10892.5.1.3.2.0"

  [[inputs.snmp.field]]
     name = "system-globalstatus"
     oid  = ".1.3.6.1.4.1.674.10892.5.2.1.0"

  [[inputs.snmp.table]]
     name = "idrac-hosts"
     inherit_tags = [ "system-name" , "disks-name" ]

    [[inputs.snmp.table.field]]
       name = "bios-version"
       oid = ".1.3.6.1.4.1.674.10892.5.4.300.50.1.8"

    [[inputs.snmp.table.field]]
       name = "raid-batterystate"
       oid = ".1.3.6.1.4.1.674.10892.5.5.1.20.130.15.1.4"

    [[inputs.snmp.table.field]]
       name = "intrusion-sensor"
       oid = ".1.3.6.1.4.1.674.10892.5.4.300.70.1.6"

    [[inputs.snmp.table.field]]
       name = "disks-mediatype"
       oid = ".1.3.6.1.4.1.674.10892.5.5.1.20.130.4.1.35"

    [[inputs.snmp.table.field]]
       name = "disks-state"
       oid = ".1.3.6.1.4.1.674.10892.5.5.1.20.130.4.1.4"

    [[inputs.snmp.table.field]]
       name = "disks-predictivefail"
       oid = ".1.3.6.1.4.1.674.10892.5.5.1.20.130.4.1.31"

    [[inputs.snmp.table.field]]
       name = "disks-capacity"
       oid = ".1.3.6.1.4.1.674.10892.5.5.1.20.130.4.1.11"

    [[inputs.snmp.table.field]]
       name = "disks-name"
       oid = ".1.3.6.1.4.1.674.10892.5.5.1.20.130.4.1.2"
       is_tag = true

    [[inputs.snmp.table.field]]
       name = "memory-status"
       oid = ".1.3.6.1.4.1.674.10892.5.4.200.10.1.27"

    [[inputs.snmp.table.field]]
       name = "storage-status"
       oid = ".1.3.6.1.4.1.674.10892.5.2.3"

    [[inputs.snmp.table.field]]
       name = "temp-status"
       oid = ".1.3.6.1.4.1.674.10892.5.4.200.10.1.63"

    [[inputs.snmp.table.field]]
       name = "psu-status"
       oid = ".1.3.6.1.4.1.674.10892.5.4.200.10.1.9"

    [[inputs.snmp.table.field]]
       name = "log-dates"
       oid = ".1.3.6.1.4.1.674.10892.5.4.300.40.1.8"

    [[inputs.snmp.table.field]]
       name = "log-entry"
       oid = ".1.3.6.1.4.1.674.10892.5.4.300.40.1.5"

    [[inputs.snmp.table.field]]
       name = "log-severity"
       oid = ".1.3.6.1.4.1.674.10892.5.4.300.40.1.7"

    [[inputs.snmp.table.field]]
       name = "log-number"
       oid = ".1.3.6.1.4.1.674.10892.5.4.300.40.1.2"
       is_tag = true

    [[inputs.snmp.table.field]]
       name = "nic-name"
       oid = ".1.3.6.1.4.1.674.10892.5.4.1100.90.1.30"
       is_tag = true

    [[inputs.snmp.table.field]]
       name = "nic-vendor"
       oid = ".1.3.6.1.4.1.674.10892.5.4.1100.90.1.7"

    [[inputs.snmp.table.field]]
       name = "nic-status"
       oid = ".1.3.6.1.4.1.674.10892.5.4.1100.90.1.4"

    [[inputs.snmp.table.field]]
       name = "nic-current_mac"
       oid = ".1.3.6.1.4.1.674.10892.5.4.1100.90.1.15"
       conversion = "hwaddr"

  [[inputs.snmp.field]]
     name = "fan1-speed"
     oid  = ".1.3.6.1.4.1.674.10892.5.4.700.12.1.6.1.1"

  [[inputs.snmp.field]]
     name = "fan2-speed"
     oid  = ".1.3.6.1.4.1.674.10892.5.4.700.12.1.6.1.2"

  [[inputs.snmp.field]]
     name = "fan3-speed"
     oid  = ".1.3.6.1.4.1.674.10892.5.4.700.12.1.6.1.3"

  [[inputs.snmp.field]]
     name = "fan4-speed"
     oid  = ".1.3.6.1.4.1.674.10892.5.4.700.12.1.6.1.4"

  [[inputs.snmp.field]]
     name = "fan5-speed"
     oid  = ".1.3.6.1.4.1.674.10892.5.4.700.12.1.6.1.5"

  [[inputs.snmp.field]]
     name = "fan6-speed"
     oid  = ".1.3.6.1.4.1.674.10892.5.4.700.12.1.6.1.6"
 
  [[inputs.snmp.field]]
     name = "inlet-temp"
     oid  = ".1.3.6.1.4.1.674.10892.5.4.700.20.1.6.1.1"

  [[inputs.snmp.field]]
     name = "exhaust-temp"
     oid  = ".1.3.6.1.4.1.674.10892.5.4.700.20.1.6.1.2"

  [[inputs.snmp.field]]
     name = "cpu1-temp"
     oid  = ".1.3.6.1.4.1.674.10892.5.4.700.20.1.6.1.3"

  [[inputs.snmp.field]]
     name = "cpu2-temp"
     oid  = ".1.3.6.1.4.1.674.10892.5.4.700.20.1.6.1.4"

  [[inputs.snmp.field]]
     name = "cmos-batterystate"
     oid  = ".1.3.6.1.4.1.674.10892.5.4.600.50.1.6.1.1"

  [[inputs.snmp.field]]
     name = "system-watts"
     oid  = ".1.3.6.1.4.1.674.10892.5.4.600.30.1.6.1.3"
```

Configure the main Telegraf configuration:
```bash
sudo vi /etc/telegraf/telegraf.conf
```

Find the `[[outputs.influxdb]]` section and update it:
```toml
[[outputs.influxdb]]
  urls = ["http://localhost:8086"]
  database = "idrac_monitoring"
  # No need for username/password for the default setup
```

Start and enable Telegraf:
```bash
sudo systemctl start telegraf
sudo systemctl enable telegraf
```

## Setting Up Grafana

### Install Grafana
```bash
# Add Grafana repository
sudo tee /etc/yum.repos.d/grafana.repo << EOF
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF

# Install Grafana
sudo dnf install -y grafana

# Start Grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

### Configure Firewall to Allow Grafana Web Access
```bash
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload
```

### Initial Grafana Setup
1. Access Grafana by navigating to `http://your-rocky-linux-ip:3000` in your browser
2. Default login credentials are admin/admin
3. You will be prompted to change the password on first login

### Add InfluxDB as a Data Source
1. Go to Configuration > Data Sources > Add data source
2. Select InfluxDB
3. Set the following settings:
   - Name: iDRAC Monitoring
   - URL: http://localhost:8086
   - Database: idrac_monitoring
4. Click "Save & Test"

### Import the Dashboard
1. Go to Create > Import
2. You can either:
   - Enter a Grafana Dashboard ID if available
   - Upload a JSON file if you have a custom dashboard
   - Create a new dashboard

## Configuring SNMP on iDRAC

1. Log in to your iDRAC web interface (e.g., https://192.168.0.11)
2. Navigate to Network > Services
3. Find the SNMP section
4. Enable SNMP Agent
5. Set Community Name to "public" (or customize it and update your Telegraf config)
6. Configure SNMP settings as needed
7. Apply the changes

Repeat for each iDRAC you want to monitor.

## Verification and Troubleshooting

### Verify SNMP Connectivity
```bash
# Test SNMP connectivity to your iDRAC
snmpwalk -v1 -c public 192.168.0.11
```

### Check Telegraf Status
```bash
sudo systemctl status telegraf
sudo journalctl -u telegraf
```

### Verify Data in InfluxDB
```bash
# Connect to InfluxDB
influx

# Switch to the iDRAC monitoring database
> USE idrac_monitoring

# Check if data is being collected
> SHOW MEASUREMENTS
> SELECT * FROM "idrac-hosts" LIMIT 10
```

### Grafana Dashboard Troubleshooting
If your dashboard is not showing data:
1. Verify data exists in InfluxDB
2. Check your query in Grafana
3. Adjust the time range in Grafana
4. Verify the data source connection
