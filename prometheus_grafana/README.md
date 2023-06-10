# Install `Prometheus` on Ubuntu 20.04 - 22.04

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

* Check the targets section `http://<ip>:9090/targets`, you should see `active target`.

* You also can check your `node_exporter` target metrics `http://<ip>:9100/metrics`

## Install `Grafana` on Ubuntu 20.04 - 22.04

*You can install `grafana` on dedicated server or container for distributed highload environment*

* To visualize metrics we can use `Grafana`. There are many different data sources that `Grafana` supports, one of them is Prometheus.

> Install dependencies for `Grafana`

```sh
sudo apt-get install -y apt-transport-https software-properties-common
```

> Add gpg key for verifications

```sh
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
```

> Add repo for stable releases

```sh
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```

> Update and install `Grafana`

```sh
sudo apt-get update
sudo apt-get -y install grafana
```

* If you have any issues, you can download linux package from [Home page](https://grafana.com/grafana/download)

> Download and install grafana packages

```sh
sudo apt-get install -y adduser libfontconfig1
wget https://dl.grafana.com/enterprise/release/grafana-enterprise_9.5.3_amd64.deb
sudo dpkg -i grafana-enterprise_9.5.3_amd64.deb
```

* Start and enable Grafana service.

```sh
sudo systemctl enable grafana-server

sudo systemctl start grafana-server
```

> Check 

```sh
sudo systemctl status grafana-server
```

* Go to http://<ip>:3000 and log in to the Grafana using default credentials. The username is `admin`, and the password is `admin` as well.

* When you log in for the first time, you get the option to change the password.

* To visualize metrics, you need to add a data source first. Click Add data source and select Prometheus. For the URL, enter http://localhost:9090 and click Save and test. You can see Data source is working.

> Best way to store all data source configurations add data source file as code.

```sh
sudo vim /etc/grafana/provisioning/datasources/datasources.yaml
```

> `datasources.yaml`

```sh
apiVersion: 1

datasources:
  - namer: Prometheus
    type: Prometheus
    url: http://localhost:9090
    isDefault: true
```

> Restart `Grafana` services

```sh
sudo systemctl restart grafana-server
```

* Go back to Grafana and refresh the page. You should see the Prometheus data source.

* We can import existing Grafana dashboards or create your own. Let's create a simple graph.

* Go back to the Prometheus, and let's explore what metrics we have. Start typing `scrape_duration_seconds` and click `Execute`. This metric will show you the duration of the scrape of each Prometheus target. At this point, we have `node_exporter` and `prometheus` targets. We're going to use this metric to create a simple graph in Grafana.

* Go to Grafana and click `create Dashboard` and then add a `new panel`.

* Give a title Scrape Duration and paste `scrape_duration_seconds` metric. You can also reduce the time interval to 1 hour.

* For the legend, we can use the `job` label and for the unit - seconds. There are a lot of configuration `parameters` that you can use. Let's keep it simple and click `apply` and save dashboard as Prometheus.

* Since we already have Node Exporter, we can import an open-source dashboard to visualize CPU, Memory, Network, and a bunch of other metrics. You can search for node exporter on the Grafana website https://grafana.com/grafana/dashboards/.

* Copy 1860 ID to Clipboard.

* Now, in Grafana, you can click Import and paste this ID. Then load the dashboard. Select Prometheus datasource and click import.

* You have all sorts of metrics here that come from node exporter.



