# Raspberry Pi Network Monitor
This repository includes instructions and configuration files for setting up a Raspberry Pi to monitor a network's connectivity status via a Grafana Dashboard through Prometheus and Telegraf data collection.

## Motivation for this Project
![Network Diagram](/network-diagram.png)

I recently moved to a more rural location in Minnesota that does not have buried coper or fiber internet infrastructure. Instead, our internet connection comes from a fixed wireless signal. Verizon has a good explanation of the technology [here](https://www.verizon.com/about/blog/fixed-wireless-access), if you are not familiar. 

The unique part about my location is that the house we live in does not have a direct line of sight with our ISP's closest basestation, but another building on the property does. So, our ISP placed their antenna (a Ubiquiti LiteBeam 60) that captures their external internet connection on that building. That signal is then passed into a switch where a MikroTik Cube Lite 60 repeats the signal back to our home. Another MikroTik Cube Lite 60 that is located on our house captures the repeated signal and passes it into our home network's router.

Fore the most part, our experience has been pretty good. Speeds are about 100Mbps down and 50Mbps up with ping times averaging right around 10ms. Keep in mind, we came from a city where we had buried copper that got 500Mbps down and 20Mbps up. We have run into some periods of high latency in our home network (Network 2 in the diagram above) and some intermittent connectivity drops that been reported by our Ubiquiti Dream Machine Pro. 

We wanted to begin collecting data about the health of our network to try and give our ISP data confirming our issues. However, our network set up has more points of failure than most home networks. In areas where buried fiber or copper lines are run into a house, the usual culprits of a bad internet connection are the router or modem and both are found right inside your home. This makes it fairly easy for you and/or your ISP to diagnose network issues. With our setup, the antenna that captures the external internet connection could be down. One or both of the repeaters could  be down. There could be some combination of both, or it could be our home router. These devices are scattered across two buildings.

Knowing whether or not there was an internet connection in the building 500ft away from our house (Network 1 in the diagram above) was going to give us the best information. We could immediately narrow a loss of connectivity in our house to an issue with the antenna capturing the broader internet or if there was an issue with one (or both) of our repeaters. The challenge is that these are two separate networks. We wanted a local device in the second building that we could access remotely to view historical and current ping times. This is where the Raspberry Pi comes in. We can use a Raspberry Pi that runs a local Grafana dashboard to display network statistics from data collected by locally running Prometheus and Telegraf services. Using Cloudflare Tunnels, we could securely make this dashboard available publicly to us. **All of this can be down without having to pay for any services**.

Now, I do realize that if the Raspberry Pi does not have an internet connection that we won't be able to reach our Raspberry Pi's Grafana dashboard via the Cloudflare Tunnel. That's fine. That will immediately tell us that we have an issue with the external antenna capturing our boarder internet signal. If we can reach the Raspberry Pi and it's reporting normal ping times, but we don't have an internet connection in our house - then we either have an issue with our repeaters or our local router. This can easily be tracked down since our Ubiquiti Dream Machine Pro will inform us whether it has an internet signal or not. The Raspberry Pi also gives us the flexibility to connect a red LED light that would get turned on if the Pi doesn't have an internet connection.

Don't feel like you can't use this project in your own home network if your house as a buried copper or fiber line. This could be a great project to set up in you want more data about your network and prove to your ISP that you are having issues.

## Warning
Since the Raspberry Pi will be sending pings 24/7, leaving it running will consume some amount of network data. Exactly how much data will depend on how many different external services you ping, how frequently you ping them, and how many packets you send with each ping session. If you ISP imposes data limits, be careful to ensure that your project won't tear through your data limit!

## Preparing the Raspberry Pi

We first need to get our Raspberry Pi up and running. That starts with flashing a MicroSD card with Raspberry Pi OS and booting into our machine.

### Installing Raspberry Pi OS

Raspberry Pi has some excellent documentation about all of the ways that you can get Raspberry Pi OS installed on your Pi. That documentation is found here: [https://www.raspberrypi.com/documentation/computers/getting-started.html#using-raspberry-pi-imager](https://www.raspberrypi.com/documentation/computers/getting-started.html#using-raspberry-pi-imager)

I used [Raspberry Pi Imager](https://www.raspberrypi.com/software/) to flash my MicroSD card and install Raspberry Pi OS. Once you download Raspberry Pi Imager:
- Select the OS version that you want to install.
  - **Note**: I chose to install the 64-Bit ARM version on my Raspberry Bi 3 B+.
- Select the MicroSD card that you want to flash.
- Click `Write`.

Don't worry about changing or enabling any of the advanced installation options, if you don't want to. We will configure all of the settings that we need in the next step. 

This process will probably take several minutes to complete.

### Configuring Raspberry Pi OS on First Boot

Once your MicroSD card is flashed with your selected version of Raspberry Pi OS, insert your MicroSD into your Raspberry Pi, connect a monitor, keyboard, mouse, and plug in your Raspberry Pi to boot it up!

During the first boot, you will need to do some last configuration of the OS. When prompted:
- If you are using a wireless keyboard and/or mouse, you will first be prompted to get those connected.
- Select your *correct* Country, Language, and Timezone.
- Enter in a Username and Password. This username and password will be used to log into the Pi.
  - **Note**: Your username can be whatever you like. Make sure that you write down what your username is as we will need it for a later step. My best recommendation is to keep it simple with all lowercase letters, no spaces, no special characters, and no numbers. If you really can't think of something, you could use the default username of `pi`. I would really not recommend this for security reasons though.
- Adjust the settings for your Desktop (if needed).
- If you are not using a wired Ethernet connection, you will be prompted to enter your Wi-Fi security credentials.
- Finally, you will be prompted to check for any Raspberry Pi OS Updates. **Be sure to do this.**

Once your configuration is complete, you will be taken to your Desktop. Congrats! You now have your own Raspberry Pi. 

There are just two settings that we need to enable. 

### Enabling Auto Login
To enable Auto Login, in the top-left corner of your Desktop menu bar find the Raspberry Pi Logo. Then:
- Select `Raspberry Pi Configuration` from the dropdown.
- Inside the window that opens, select the `System` tab.
- Toggle `Auto login` to `On`.

Enabling this setting is **not** required. The reason for enabling this setting is to prevent having to enter your username and password every time the Pi has to restart or boot up. If you are installing your Raspberry Pi within your own home, it is probably safe to enable this setting. If you are installing your Pi in any public place, I would leave this setting disabled. 

### Enabling SSH-ing
To enable SSH, in the top-left corner of your Desktop menu bar find the Raspberry Pi Logo. Then:
- Select `Raspberry Pi Configuration` from the dropdown.
- Inside the window that opens, select the `Interfaces` tab.
- Toggle `SSH` to `On`.

Again, enabling this setting is **not** required. The reason for enabling this setting is to be able to remote into your machine if there are ever any issues with it.

### Updating Raspberry Pi OS

The last thing that we need to do is run a quick quick check to ensure that all of the packets provided with Raspberry Pi OS are update to do. 

This is easily done by:
- Opening a terminal window.
- Input `sudo apt update` and press `Enter`.
  - **Note:** You will most likely be prompted for your password that you created during setup.
- Input `sudo full-upgrade` and press `Enter`.
  - **Note:** You will most likely be prompted for your password that you created during setup.

This process should take no longer than a few minutes to complete. 

We are now ready to begin to install and setup the infrastructure that is going to support your network monitoring dashboard!

## Setting Up Prometheus

**Prometheus Website:** [https://prometheus.io](https://prometheus.io)

**Github Repository:** [https://github.com/prometheus/prometheus/](https://github.com/prometheus/prometheus/)

Prometheus is a monitoring system that can collect data from specific targets at specific intervals. This is exactly how we are going to routinely collect the network data that Telegraf is going to provide. We can then use Prometheus as a source of data within our Grafana dashboard!

### Installing Prometheus

At the time of writing, the long-term supported version of Prometheus is `2.37.6`. That is the version of Prometheus that I will be installing. All of the `v2.37.6` downloads can be found at: [https://github.com/prometheus/prometheus/releases/tag/v2.37.6](https://github.com/prometheus/prometheus/releases/tag/v2.37.6).

If you are looking for a different release, all releases can be found at [https://github.com/prometheus/prometheus/releases](https://github.com/prometheus/prometheus/releases) or at [https://prometheus.io/download/](https://prometheus.io/download/).

Since I installed the 64-bit ARM version of Raspberry Pi OS, I will be installing the `linux-arm64` version of Prometheus. You will need to install the correct version of Prometheus based on the version and type of OS on your own Raspberry Pi.

Installing Prometheus can be done with a few simple commands via the command line:
- Open a new terminal window on your Raspberry Pi.
- Input `cd ~` and press `Enter`.
  - **Note:** This command will move you to your home directory. This is where we will install Prometheus.
- Input `wget https://github.com/prometheus/prometheus/releases/download/v2.37.6/prometheus-2.37.6.linux-arm64.tar.gz` and press `Enter`.
  - **Note:** This could take a minute or two to download - depending on your internet speeds.
- Input `tar xfz prometheus-2.37.6.linux-arm64.tar.gz` and press `Enter`.
- Input `mv prometheus-2.37.6.linux-arm64.tar.gz/ prometheus/` and press `Enter`.
- Input `rm prometheus-2.37.6.linux-arm64.tar.gz` and press `Enter`.

After running these commands, you should see a `prometheus` folder within your home directory. You can double check this by running `ls -l`. Within the list of folders and files, you should see a `prometheus` entry.

### Creating a Prometheus Service

We want Prometheus to start up as soon as Raspberry Pi boots up. We can do this easily by creating a service for Prometheus. You can learn more about Linux Services here: [https://manpages.ubuntu.com/manpages/bionic/man5/systemd.service.5.html](https://manpages.ubuntu.com/manpages/bionic/man5/systemd.service.5.html)

Creating a service can be done with the following commands on the command line:
- Open a new terminal window on your Raspberry Pi.
- Input `sudo nano /etc/systemd/system/prometheus.service` and press `Enter`.
  - **Note:** This will open a text editor within your command line.
- Paste the text below into your command line text editor. 
  - **Note:** Replace the 4 instances of `<your-user-name>` in the text content below with the username that you set up when configuring your Raspberry Pi on the first start up.
  - **Note:** The text content below can also be found within the `prometheus.service` file within this repository.
```
[Unit]
Description=Prometheus Server
Documentation=https://prometheus.io/docs/introduction/overview/
After=network-online.target

[Service]
User=<your-user-name>
Restart=on-failure

ExecStart=/home/<your-user-name>/prometheus/prometheus \
  --config.file=/home/<your-user-name>/prometheus/prometheus.yml \
  --storage.tsdb.path=/home/<your-user-name>/prometheus/data

[Install]
WantedBy=multi-user.target
```
- After you have pasted this text into your command line text editor and replaced `<your-user-name>` with your actual Pi's username, you are ready to close and save the file. To do this, press `CTRL + X`, then `Y`, then `ENTER`.
- On the command line, input `sudo systemctl enable prometheus` and press `Enter`.
- Input `sudo systemctl start prometheus` and press `Enter`.
- Input `sudo systemctl status prometheus` and press `Enter`.
  - If things were successful, after running `sudo systemctl status prometheus` you should see output that tells you that the `prometheus` services is `ACTIVE`.

**Anytime that you make changes to `~/prometheus/prometheus.yml`, you will want to restart your Prometheus service so the changes get picked up. To do this, you can run `sudo systemctl restart prometheus`.**

### Viewing the Web Interface for Prometheus

Now that the Prometheus service is running and active on your Raspberry Pi, you should be able to visit its dashboard. 

Within your Pi's web browser, attempt to visit `http://localhost:9090`. The page should load you into the Prometheus dashboard. 

Congrats! You just set up a Prometheus service to collect data. Now we need to collect data about our network and pass it into Prometheus.

## Setting Up Telegraf

**Telegraf Website:** [https://www.influxdata.com/time-series-platform/telegraf/](https://www.influxdata.com/time-series-platform/telegraf/)

**Github Repository:** [https://github.com/influxdata/telegraf](https://github.com/influxdata/telegraf)

Telegraf is an InfluxDB plugin that can be used to collect any number of system metrics. We are going to use one of its many plugins to collect ping times on our network. We can use another output plugin to easily pass our collected data out to Prometheus.

### Installing Telegraf

At the time of writing, the current version of Telegraf is `1.26.0`. That is the version of Prometheus that I will be installing. All of the `v1.26.0` downloads can be found at: [https://github.com/influxdata/telegraf/releases/tag/v1.26.0](https://github.com/influxdata/telegraf/releases/tag/v1.26.0).

If you are looking for a different release, all releases can be found at [https://github.com/influxdata/telegraf/releases](https://github.com/influxdata/telegraf/releases) or at [https://portal.influxdata.com/downloads/](https://portal.influxdata.com/downloads/).

Since I installed the 64-bit ARM version of Raspberry Pi OS, I am going to install the Ubuntu & Debian versions of Telegraf. This should work with most (if not all) Raspberry Pis. You can choose to install different version.

Telegraf has some great installation instructions ([https://portal.influxdata.com/downloads/](https://portal.influxdata.com/downloads/)) and that is what I will be following.

Installing Telegraf can be done with a few simple commands via the command line:
- Open a new terminal window on your Raspberry Pi.
- Input `cd ~` and press `Enter`.
- Copy the commands below, paste them into your command line, and press `Enter`.
```
# influxdata-archive_compat.key GPG Fingerprint: 9D539D90D3328DC7D6C8D3B9D8FF8E1F7DF8B07E
wget -q https://repos.influxdata.com/influxdata-archive_compat.key
echo '393e8779c89ac8d958f81f942f9ad7fb82a25e133faddaf92e15b16e6ac9ce4c influxdata-archive_compat.key' | sha256sum -c && cat influxdata-archive_compat.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg > /dev/null
echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive_compat.gpg] https://repos.influxdata.com/debian stable main' | sudo tee /etc/apt/sources.list.d/influxdata.list
sudo apt-get update && sudo apt-get install telegraf
```

This process could take a few minutes to complete - depending on your internet connection. 

Congrats! You've just installed Telegraf. Now we need to set a up a service for it just like we did for Prometheus.

### Configuring a Telegraf Service

We want Telegraf to start up as soon as Raspberry Pi boots up. We can do this easily by creating a service for Telegraf. You can learn more about Linux Services here: [https://manpages.ubuntu.com/manpages/bionic/man5/systemd.service.5.html](https://manpages.ubuntu.com/manpages/bionic/man5/systemd.service.5.html)

Creating a service can be done with the following commands on the command line:
- Open a new terminal window on your Raspberry Pi.
- Input `sudo nano /etc/telegraf/telegraf.conf` and press `Enter`.
  - **Note:** This will open a text editor within your command line.
- Paste the text below into your command line text editor. 
  - **Note:** The text content below can also be found within the `telegraf.conf` file within this repository.
  - This is where you might want to make changes. The configuration file says that we are going to ping `1.1.1.1` (Cloudflare's public DNS) 1 time every 15 seconds and our ping command will timeout after 5 seconds. We will then make the data available in a Prometheus format on port `9091`.
```
# Input plugins

# Ping plugin
[[inputs.ping]]
urls = ["1.1.1.1"]
count = 1
ping_interval = 15.0
timeout = 5.0

# Output format plugins
[[outputs.prometheus_client]]
  listen = ":9091"
  metric_version = 2
```
- After you have pasted this text into your command line text editor, you are ready to close and save the file. To do this, press `CTRL + X`, then `Y`, then `ENTER`.
- On the command line, input `sudo systemctl enable telegraf.service` and press `Enter`.
- Input `sudo systemctl start telegraf.service` and press `Enter`.
- Input `sudo systemctl status telegraf.service` and press `Enter`.
  - If things were successful, after running `sudo systemctl status telegraf.service` you should see output that tells you that the `telegraf` services is `ACTIVE`.

**Anytime that you make changes to `/etc/telegraf/telegraf.conf`, you will want to restart your Telegraf service so the changes get picked up. To do this, you can run `sudo systemctl restart telegraf.service`.**

Congrats! You just set up a Telegraf service to collect network ping data. Now we need to tell Prometheus that there is a new set of data available for collection on port `9091`.

### Pointing Prometheus to Telegraf's Data

Now that we have Telegraf service running to collect our network ping times, we need to inform Prometheus about this new source of data. To do this, we need to update Prometheus's configuration file and then restart the Prometheus service to have the change be picked up.

First we need to update our `prometheus.yml` file. This can be done via the command line:
- Open a new command line window.
- Input `sudo nano ~/prometheus/prometheus.yml` and press `Enter`.
- Underneath the `scrape_configs` section, add the following:
  - **Note:** An example Prometheus configuration file can be found within the `prometheus.yml` file within this repository.
```
- job_name: "telegraf-agent"
  static_configs:
    - targets: ["localhost:9091"]
```
  - This tells Prometheus that it has another source of data it should scrape and that the data to scrape is found at `localhost:9091`. This port number should match the port that is listed under `outputs.prometheus_client` within your `telegraf.conf` file.
- After you have added this information, you are ready to close and save the file. To do this, press `CTRL + X`, then `Y`, then `ENTER`.

Now that our Prometheus configuration has been updated, we need to restart our Prometheus service for the changes to be picked up. To do this, run `sudo systemctl restart prometheus` within the command line. To do this:
- On the command line, input `sudo systemctl restart prometheus` and press `Enter`.
- On your Raspberry Pi, open a web browser and visit `localhost:9090/targets`.
  - If things are set up correctly, you should see two targets listed: `prometheus` and `telegraf-agent`. 
  - **Note:** The names of these targets match the `job_name` property within the `scrape_configs` section of the `prometheus.yml` configuration file.

## Setting Up Grafana

**Grafana Website:** [https://grafana.com](https://grafana.com)

**Github Repository:** [https://github.com/grafana/grafana](https://github.com/grafana/grafana)

Grafana is an open-source tool that can be used to visual data from a number of different sources. We are going to use it to visualize the data that our Prometheus service is collecting from Telegraf.

### Installing Grafana

Grafana has some great installation instructions ([https://grafana.com/tutorials/install-grafana-on-raspberry-pi/](https://grafana.com/tutorials/install-grafana-on-raspberry-pi/)) and that is what I will be following.

Installing Grafana can be done with a few simple commands via the command line:
- Open a new terminal window on your Raspberry Pi.
- Input `wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -` and press `Enter`.
- Input `echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list` and press `Enter`.
- Input `sudo apt-get update` and press `Enter`.
- Input `sudo apt-get install -y grafana` and press `Enter`.

The process could take a few minutes - depending on your internet connection. Grafana is now installed, but we need to create a service for it just like we did with Prometheus and Telegraf.

### Configuring a Grafana Service

We want Grafana to start up as soon as Raspberry Pi boots up. We can do this easily by creating a service for Grafana. You can learn more about Linux Services here: [https://manpages.ubuntu.com/manpages/bionic/man5/systemd.service.5.html](https://manpages.ubuntu.com/manpages/bionic/man5/systemd.service.5.html)

Creating a service can be done with the following commands on the command line:
- Open a new terminal window on your Raspberry Pi.
- Input `sudo /bin/systemctl enable grafana-server` and press `Enter`.
- Input `sudo /bin/systemctl start grafana-server` and press `Enter`.

### Viewing the Web Interface for Grafana

If everything has gone right, we should not be able to log into Grafana and start dashboarding data! On your Raspberry Pi, open a web browser and attempt to visit `http://localhost:3000`.

You should see the Grafana login page. You should be able to login with the following default credentials of:
```
username: admin
password: admin
```

**Please, change the password when you are prompted.**

If you were able to log in - we are ready to start viewing our network health data!

### Adding Prometheus as a Data Source to Grafana

Now that we have Grafana to dashboard our data, we need to add our Prometheus service to Grafana as a potential data source. To do this:
- Open a web browser on your Raspberry Pi and visit `localhost:3000/`.
- Log into your Grafana instance.
- From the left-hand sidebar menu, hover over the `⚙️` (Settings) icon and select `Data sources` from the flyout menu.
- Once the page loads, click the blue `Add new data source` button.
- Select the `Prometheus` option from the page. 
  - **Note:** You may need to search for it on the page.
- Enter a name for your datasource in the `Name` field at the top of the page.
  - If you aren't sure about a name, I would go with something simple like `Prometheus`.
- Within the `HTTP` section enter `localhost:9090` in the `URL` field. 
  - **Note:** If you used a different port for your Prometheus server, use that port instead.
- Scroll down to the bottom of the page and click the blue `Save & test` button.

If things are working correctly, you should see a green `Data source is working` expandable pop up below the blue `Save & test` button. 

If you need more assistance or have questions, Grafana has some excellent documentation about adding a Prometheus datasource. That documentation can be found at: [https://grafana.com/docs/grafana/latest/datasources/prometheus/](https://grafana.com/docs/grafana/latest/datasources/prometheus/)

### Creating our Dashboard & Panel in Grafana

Now that Grafana has access to the data that we are collecting about our network's health, we can finally create our network monitoring dashboard! 

![Grafana Dashboard](/grafana-dashboard.png)

You can either make your own custom dashboard from scratch, or you can import my dashboard via the `grafana-dashboard.json` file that is included within this repository. Grafana has a great explanation of how to import a dashboard at: [https://grafana.com/docs/grafana/latest/dashboards/manage-dashboards/#import-a-dashboard](https://grafana.com/docs/grafana/latest/dashboards/manage-dashboards/#import-a-dashboard)

This is not a comprehensive Grafana tutorial. Grafana is extremely powerful! If you are looking for more tutorials, Grafana has some great ones here: [https://grafana.com/tutorials/](https://grafana.com/tutorials/). A quick Google search should also get you some great answers!

## Expanding this Project
There are numerous ways that this project could be expanded or customized.
- A Cloudflare Tunnel ([https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)) could be used to make your local Grafana dashboard available to the public internet in a secure manner. This would allow you to check on your network's health from anywhere in the world (as long as your Pi had a network connection).
  - Cloudflare offers this service for **free**. You could even buy a cheap custom domain to tunnel into your Pi!
- Telegraf has *many* plugins that can be used to collect data. Those plugins can be found at [https://docs.influxdata.com/telegraf/v1.26/plugins/](https://docs.influxdata.com/telegraf/v1.26/plugins/). You could find a new plugin to use to collect another source of data.
- You could modify the existing `telegraf.conf` file to ping more websites or change how frequently pings are done.
- Make some complex visualizations in Grafana. This tutorial only scratched the surface of what is possible with Grafana. Go crazy and make some cool, dynamic panels!

## Acknowledgements
- This article ([https://mrkaran.dev/posts/isp-monitoring/](https://mrkaran.dev/posts/isp-monitoring/)) served as the primary inspiration for this project. Great work! 