# Install Prometheus on Ubuntu 20.04 - 22.04

* First of all, let create a dedicated Linux user or sometimes called a system account for Prometheus. Having individual users for each service serves two main purposes:

> It is a security measure to reduce the impact in case of an incident with the service.

> It simplifies administration as it becomes easier to track down what resources belong to which service.

* To create a system user or system account, run the following command:

```sh
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false prometheus
```

--system - Will create a system account.

--no-create-home - We don't need a home directory for Prometheus or any other system accounts in our case.

--shell /bin/false - It prevents logging in as a Prometheus user.

--prometheus - Will create Prometheus user and a group with the exact same name.

* You can use the `curl` or `wget` command to download Prometheus, or the latest version of Prometheus you can download from the [home page](https://prometheus.io/download/)

```sh
wget https://github.com/prometheus/prometheus/releases/download/v2.32.1/prometheus-2.32.1.linux-amd64.tar.gz
```

* Extract all Prometheus files from the archive.

```sh
tar -xvf prometheus-2.44.0.linux-amd64.tar.gz
```

* You would have a disk mounted to the data directory. For this tutorial, I will simply create a /data directory. Also, you need a folder for Prometheus configuration files.

```sh
sudo mkdir -p /data /etc/prometheus
```

* Change the directory to Prometheus and move some files.

```sh
cd prometheus-2.44.0.linux-amd64
```

* Move the prometheus binary and a promtool to the /usr/local/bin/. promtool is used to check configuration files and Prometheus rules.

```sh
sudo mv prometheus promtool /usr/local/bin/
```

* We can move console libraries to the Prometheus configuration directory. Console templates allow for the creation of arbitrary consoles using the Go templating language. You don't need to worry about it if you're just getting started.

```sh
sudo mv consoles/ console_libraries/ /etc/prometheus/
```

* Move the example of the main prometheus configuration file.

```sh
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
```

* Set correct ownership for the /etc/prometheus/ and data directory.

```sh
sudo chown -R prometheus:prometheus /etc/prometheus /data
```

* Verify that you can execute the Prometheus binary by running the following command:

```sh
prometheus --version
```

* To get more information and configuration options, run Prometheus help.

```sh
prometheus --help
```

* We're going to use systemd, which is a system and service manager for Linux operating systems. For that, we need to create a systemd unit configuration file.

```sh
sudo vim /etc/systemd/system/prometheus.service
```

> [prometheus.service](prometheus.service)

```sh
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

* To automatically start the Prometheus after reboot, run enable and start.

```sh
sudo systemctl enable prometheus

sudo systemctl start prometheus
```

* Check running service

```sh
sudo systemctl status prometheus
```

* If you have any issues, check logs and journal

```sh
journalctl -u prometheus -f --no-pager
```

* Now we can try to access it via browser. I'm going to be using the IP address of the Ubuntu server. You need to append port `9090` to the IP. For example `http://<ip>:9090`.

## Install Node Exporter on Ubuntu 20.04 - 22.04

* Node Exporter uses to collect Linux system metrics like CPU load and disk I/O. Node Exporter will expose these as Prometheus-style metrics. Since the installation process is very similar, I'm not going to cover as deep as Prometheus.

* Create a system user for Node Exporter by running the following command:

```sh
sudo user add \
  --no-create-home \
  --system \
  --shell /bin/false node_exporter
```

* You can download Node Exporter from the same [page](https://prometheus.io/download/), or use `wget`.

```sh
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.0/node_exporter-1.6.0.linux-amd64.tar.gz
```

* Extract node exporter from the archive.

```sh
tar -xvf node_exporter-1.6.0.linux-amd64.tar.gz
```

* Move binary to the /usr/local/bin.

```sh
sudo mv \
  node_exporter-1.3.1.linux-amd64/node_exporter \
  /usr/local/bin/
```

* Clean up, delete node_exporter archive and a folder.

```sh
rm -rf node_exporter*
```

* Verify that you can run the binary and get all help options.

```sh
node_exporter --version

node_exporter --help
```

* Next, create similar systemd unit [file](node_exporter.service).

```sh
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter \
    --collector.logind

[Install]
WantedBy=multi-user.target
```

* To automatically start the Node Exporter after reboot, enable the service and start.

```sh
sudo systemctl enable node_exporter

sudo systemctl start node_exporter
```

* Check the status of Node Exporter with the following command:

```sh
sudo systemctl status node_exporter
```

* At this point, we have only a single target in our Prometheus. There are many different service discovery mechanisms built into Prometheus. For example, Prometheus can dynamically discover targets in AWS, GCP, and other clouds based on the labels.

* To create a static target, you need to add `job_name` with `static_configs` in `prometheus.yml` file.

```sh
...
  - job_name: node_export
    static_configs:
      - targets: ["localhost:9100"]
```

* Before, restarting check if the config is valid.

```sh
promtool check config /etc/prometheus/prometheus.yml
```

* You can use a POST request to reload the config.

```sh
curl -X POST http://localhost:9090/-/reload
```

* Check the targets section http://<ip>:9090/targets, you should see `active target`.

## Install Grafana on Ubuntu 20.04 - 22.04