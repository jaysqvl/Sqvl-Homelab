# Setting up the RPi4

## Getting Ready
```
sudo apt-get update && sudo apt-get upgrade -y
```

## Install Tailscale

Download it
```
curl -fsSL https://tailscale.com/install.sh | sh
```

Give it admin permissions
```
sudo tailscale up --operator=$USER
```

## Install Docker
Download the installtion script
```
curl -fsSL https://get.docker.com -o get-docker.sh
```

Run it
```
sudo sh get-docker.sh
```

Give docker permissions (so you don't have to run sudo everytime)
```
sudo usermod -aG docker $USER
```

Exit the ssh session and re-connect to let the permissions propagate
```
exit
```

## Install portainer
Create a volume for portainer
```
docker volume create portainer_data
```
Install it into docker
```
docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
```

## Create a macvlan to separate Nginx Proxy Manager and Pi-Hole

We do this in order to give each container dedicated MAC addresses and IPs on the host network and simplify DHCP reservation
```
sudo docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.254 \
  -o parent=eth0 br0
```

## Make sure your network interfaces are in promiscuous mode
Ensure the network interfaces are in promiscuous mode so that multiple MAC addresses can send data out of the same port
```
ip link set eth0 promisc on
```
```
ip link set docker0 promisc on
```
```
ip link set tailscale0 promisc on
```

To make sure that these settings stick on reboot
You can do this more advanced set-up

Place this script here for convenience
```
sudo nano /usr/local/sbin/setup-promisc.sh
```

Add these lines to the script (Ctrl+V) then (Ctrl+S) and (Ctrl+X)
```
#!/bin/bash

# Wait until eth0 is available
while ! ip link show eth0 >/dev/null 2>&1; do sleep 1; done
# Enable promiscuous mode on eth0
ip link set eth0 promisc on

# Wait until docker0 is available
while ! ip link show docker0 >/dev/null 2>&1; do sleep 1; done
# Enable promiscuous mode on docker0
ip link set docker0 promisc on

# Wait until tailscale0 is available
while ! ip link show tailscale0 >/dev/null 2>&1; do sleep 1; done
# Enable promiscuous mode on tailscale0
ip link set tailscale0 promisc on
```

Make it executable
```
sudo nano /usr/local/bin/set_promisc_mode.sh
```

Create a new systemd service file
```
sudo nano /etc/systemd/system/promisc_mode.service
```

Add these lines to it (Ctrl+V) then (Ctrl+S) and (Ctrl+X)
```
[Unit]
Description=Set network interfaces to promiscuous mode
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/set_promisc_mode.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable and restart the service
```
sudo systemctl daemon-reload
sudo systemctl enable promisc_mode.service
sudo systemctl start promisc_mode.service
```

After this you can proceed with installing the stacks with the given docker compose files