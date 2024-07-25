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

After this you can proceed with installing the stacks with the given docker compose files