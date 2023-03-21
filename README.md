# Raspberry Pi Network Monitor
This repository includes instructions and configuration files for setting up a Raspberry Pi to monitor a network's connectivity status via a Grafana Dashboard through Prometheus and Telegraf data collection.

## Motivation for this Project
![Network Diagram](/network-diagram.png)

I recently moved to a more rural location in Minnesota that does not have buried coper or fiber internet infrastructure. Instead, our internet connection comes from a fixed wireless signal. Verizon has a good explanation of the technology [here](https://www.verizon.com/about/blog/fixed-wireless-access), if you are not familiar. 

The unique part about my location is that the house we live in does not have a direct line of sight with our ISP's closest base station, but another building on the property does. So, our ISP placed their antenna (a Ubiquiti LiteBeam 60) that captures their external internet connection on that building. That signal is then passed into a switch where a MikroTik Cube Lite 60 repeats the signal back to our home. Another MikroTik Cube Lite 60 that is located on our house captures the repeated signal and passes it into our home network's router.

Fore the most part, our experience has been pretty good. Speeds are about 100Mbps down and 50Mbps up with ping times averaging right around 10ms. Keep in mind, we came from a city where we had buired copper that got 500Mbps down and 20Mpbs up. We have run into some periods of high latency in our home network (Network 2 in the diagram above) and some intermittent connectivity drops that been reported by our Ubiquiti Dream Machine Pro. 

We wanted to begin collecting data about the health of our network to try and give our ISP data confirming our issues. However, our network set up has more points of failure than more home networks. In areas where buried fiber or copper lines are run into a house, the usual culprits of a bad internet connection are the router or modem and both are found right inside your home. This makes it fairly easy for you and/or your ISP to diagnose network issues. With our setup, the antenna that captures the external internet connection could be down. One or both of the repeaters could  be down. There could be some combination of both, or it could be our home router. These devices are scattered across two buildings.

Knowing whether or not there was an internet connection in the building 500ft away from our house (Network 1 in the diagram above) was going to give us the best information. We could immediately narrow a loss of connectivity in our house to an issue with the antenna capturing the broader internet or if there was an issue with one (or both) of our repeaters. The challenge is that these are two separate networks. We wanted a local device in the second building that we could access remotely to view historical and current ping times. This is where the Raspberry Pi comes in. We can use a Raspberry Pi that runs a local Grafana dashboard to display network statistics from data collected by locally running Prometheus and Telegraf services. Using Cloudflare Tunnels, we could securely make this dashboard available publicly to us. **All of this can be down without having to pay for any services**.

Now, I do realize that if the Raspberry Pi does not have an internet connection that we won't be able to reach our Raspberry Pi's Grafana dashboard via the Cloudflare Tunnel. That's fine. That will immediately tell us that we have an issue with the external antenna capturing our boarder internet signal. If we can reach the Raspberry Pi and it's reporting normal ping times, but we don't have an internet connection in our house - then we either have an issue with our repeaters or our local router. This can easily be tracked down since our Ubiquiti Dream Machine Pro will inform us whether it has an internet signal or not. The Raspberry Pi also gives us the flexibility to connect a red LED light that would get turned on if the Pi doesn't have an internet connection.

Don't feel like you can't use this project in your own home network if your house as a buried copper or fiber line. This could be a great project to set up in you want more data about your network and prove to your ISP that you are having issues.

## Preparing the Raspberry Pi

To do.

### Installing Raspberry Pi OS

To do.

### Configuring Raspberry Pi OS

To do.

### Updating Raspberry Pi OS

To do.

## Establishing a Prometheus Data Scraper

To do.

### Installing Prometheus

To do.

### Configuring Prometheus

To do.

### Viewing the Web Interface for Prometheus

To do.

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