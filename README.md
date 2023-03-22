# Raspberry Pi Network Monitor
This repository includes instructions and configuration files for setting up a Raspberry Pi to monitor a network's connectivity status via a Grafana Dashboard through Prometheus and Telegraf data collection.

## Motivation for this Project
![Network Diagram](/network-diagram.png)

I recently moved to a more rural location in Minnesota that does not have buried coper or fiber internet infrastructure. Instead, our internet connection comes from a fixed wireless signal. Verizon has a good explanation of the technology [here](https://www.verizon.com/about/blog/fixed-wireless-access), if you are not familiar. 

The unique part about my location is that the house we live in does not have a direct line of sight with our ISP's closest base station, but another building on the property does. So, our ISP placed their antenna (a Ubiquiti LiteBeam 60) that captures their external internet connection on that building. That signal is then passed into a switch where a MikroTik Cube Lite 60 repeats the signal back to our home. Another MikroTik Cube Lite 60 that is located on our house captures the repeated signal and passes it into our home network's router.

Fore the most part, our experience has been pretty good. Speeds are about 100Mbps down and 50Mbps up with ping times averaging right around 10ms. Keep in mind, we came from a city where we had buried copper that got 500Mbps down and 20Mpbs up. We have run into some periods of high latency in our home network (Network 2 in the diagram above) and some intermittent connectivity drops that been reported by our Ubiquiti Dream Machine Pro. 

We wanted to begin collecting data about the health of our network to try and give our ISP data confirming our issues. However, our network set up has more points of failure than more home networks. In areas where buried fiber or copper lines are run into a house, the usual culprits of a bad internet connection are the router or modem and both are found right inside your home. This makes it fairly easy for you and/or your ISP to diagnose network issues. With our setup, the antenna that captures the external internet connection could be down. One or both of the repeaters could  be down. There could be some combination of both, or it could be our home router. These devices are scattered across two buildings.

Knowing whether or not there was an internet connection in the building 500ft away from our house (Network 1 in the diagram above) was going to give us the best information. We could immediately narrow a loss of connectivity in our house to an issue with the antenna capturing the broader internet or if there was an issue with one (or both) of our repeaters. The challenge is that these are two separate networks. We wanted a local device in the second building that we could access remotely to view historical and current ping times. This is where the Raspberry Pi comes in. We can use a Raspberry Pi that runs a local Grafana dashboard to display network statistics from data collected by locally running Prometheus and Telegraf services. Using Cloudflare Tunnels, we could securely make this dashboard available publicly to us. **All of this can be down without having to pay for any services**.

Now, I do realize that if the Raspberry Pi does not have an internet connection that we won't be able to reach our Raspberry Pi's Grafana dashboard via the Cloudflare Tunnel. That's fine. That will immediately tell us that we have an issue with the external antenna capturing our boarder internet signal. If we can reach the Raspberry Pi and it's reporting normal ping times, but we don't have an internet connection in our house - then we either have an issue with our repeaters or our local router. This can easily be tracked down since our Ubiquiti Dream Machine Pro will inform us whether it has an internet signal or not. The Raspberry Pi also gives us the flexibility to connect a red LED light that would get turned on if the Pi doesn't have an internet connection.

Don't feel like you can't use this project in your own home network if your house as a buried copper or fiber line. This could be a great project to set up in you want more data about your network and prove to your ISP that you are having issues.

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

## Creating a Prometheus Service

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

### Viewing the Web Interface for Prometheus

Now that the Prometheus service is running and active on your Raspberry Pi, you should be able to visit its dashboard. 

Within your Pi's web browser, attempt to visit `localhost:9090`. The page should load you into the Prometheus dashboard. 

Congrats! You just set up a Prometheus service to collect data. Now we need to collect data about our network and pass it into Prometheus.

## Establishing a Telegraf Service to Collect Network Information

To do.

### Installing Telegraf

To do.

### Configuring Telegraf

To do.

### Pointing Prometheus to Telegraf's Data

To do.

## Dashboarding our Network's Health

To do.

### Installing Grafana

To do.

### Adding Prometheus as a Data Source to Grafana

To do.

### Creating our First Panel in Grafana

To do.

## Tunneling our Local Grafana Dashboard to a Public Domain Name

To do.