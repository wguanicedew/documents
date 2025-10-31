# Apache ActiveMQ Artemis Installation Guide (AlmaLinux 9)

**Date:** 2025-10-31  

This guide walks through installing and configuring **Apache ActiveMQ Artemis 2.43.0** on **AlmaLinux 9**.

---

## 1. Install Java

Artemis requires Java 11+ (we will use Java 17):

```bash
sudo dnf install java-17-openjdk java-17-openjdk-devel -y
java --version
````

Ensure the output shows OpenJDK 17 or newer.

---

## 2. Prepare Installation Directory

We will install Artemis under `/opt/artemis`.

```bash
cd /opt
sudo mkdir artemis
cd artemis
```

---

## 3. Download and Extract Artemis

Download the latest release (2.43.0 at the time of writing):

```bash
sudo wget https://dlcdn.apache.org/activemq/activemq-artemis/2.43.0/apache-artemis-2.43.0-bin.tar.gz
sudo tar xzf apache-artemis-2.43.0-bin.tar.gz
sudo mv apache-artemis-2.43.0 artemis
```

Your directory structure should now look like this:

```python
/opt/artemis/
    apache-artemis-2.43.0-bin.tar.gz
    artemis/
       bin/
       examples/
       lib/
        ...
```

---

##  4. Create a Broker Instance

create a broker instance in `/opt/artemis/artemis-broker`.

```bash
sudo /opt/artemis/artemis/bin/artemis create /opt/artemis/artemis-broker \
  --user admin \
  --password admin \
  --allow-anonymous
```

This creates a runnable Artemis broker and configuration directory.

---

## 5. Create a Systemd Service

Create the systemd unit file `/etc/systemd/system/artemis.service`:

```bash
sudo vi /etc/systemd/system/artemis.service
```

Paste in the following content:

```ini
[Unit]
Description=Apache ActiveMQ Artemis Broker
After=network.target

[Service]
Type=forking
User=artemis
ExecStart=/opt/artemis/artemis-broker/bin/artemis-service start
ExecStop=/opt/artemis/artemis-broker/bin/artemis-service stop
Restart=on-abort
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
```

---

## 6. Create a Dedicated User and Set Permissions

```bash
sudo useradd --no-create-home --shell /sbin/nologin artemis
sudo chown -R artemis:artemis /opt/artemis/artemis-broker/
```

---

## 7. Reload and Start the Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now artemis
sudo systemctl status artemis
```

Check that the status shows **active (running)**.

---

## 8. (Optional) Firewall Configuration

If `firewalld` is active, open the Artemis ports:

```bash
sudo firewall-cmd --add-port=61616/tcp --permanent   # Core/AMQP port
sudo firewall-cmd --add-port=8161/tcp --permanent    # Web console port
sudo firewall-cmd --reload
```

---

## 9. Test the Broker

Once the service is running, test message send/receive:

```bash
/opt/artemis/artemis-broker/bin/artemis producer
/opt/artemis/artemis-broker/bin/artemis consumer
```

You should see messages successfully produced and consumed.

---

## 10. Access the Web Console

Open a browser and visit:

**http\://<your-server-ip>:8161/**

Login with:

```
Username: admin
Password: admin
```

---

## 11. configure java heap size

```
vi /opt/artemis/artemis-broker/etc/artemis.profile
JAVA_ARGS="$JAVA_ARGS -Xms2G -Xmx4G"
```
