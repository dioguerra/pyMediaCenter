# pyMediaCenter
A RPi4 media center helm chart based on opensource for your home


# Playing media with JellyFin
## Hardware Requirements and Compatibility
I started this project with a RaspberryPi 4 (4GB) ram model.
Depending on the format of your files, the user-end device you use (Computer, Smartphone or TV),
and your connection bandwidth you might require to use transcoding.

In case you need transcoding, which is needed if the format is not compatible with the WebPlayer,
or target devices, you will need:
* Good power source for your RPi4 (3A should be enough)
* Active heat dissipation on your RPi4
    * I'm using this that i got from ebay https://ebay.to/3kQ5VXy

As of this moment, this release is tested on the following platforms:

| Device              | Computer | SmartPhone | SmartTV |
|---------------------|----------|------------|---------|
| RPi4 4GB Raspbian64 |     X    |            |         |

TODO: SetUp options to enable playing on onther devices (config. revProxy)

## Software Requirements and Comaptibility
As of this moment, i do have HW transcoding but only for decoding.
There is a problem with the current OS I am using that does not have the currect libs to do the
video encoding. I think this might be possible do be compiled, but i still didnt got around to that.
What I do is, I select the movie, leave it around some 10 minutes to transcode (I am limited by
client BW) and then play it.
I also still dont have the active heat dissipation on, so more updates on that later.

TODO: Add encoding libs to the base OS or wait for upstream:
* https://github.com/jellyfin/jellyfin/issues/4023
* https://www.reddit.com/r/jellyfin/comments/ei6ew6/rpi4_hardware_acceleration_guide/
* https://github.com/raspberrypi/Raspberry-Pi-OS-64bit/issues/98
* https://www.raspberrypi.org/forums/viewtopic.php?t=268356

If you find problems, or you would like to read the jellyfin documentation on how to setup HVA use
this link https://jellyfin.org/docs/general/administration/hardware-acceleration.html#raspberry-pi-3-and-4
NOTE: Currently i only have decode enabled and encode is CPU handled (toasty times)


# Install
This section presents how to setup everything without the bits and bobs of specific configurations
During my initial try-out configuration I was heavily based on this video
https://www.youtube.com/watch?v=momNnMYkmtQ

## Requirements
### 0. disable swap - dont use swap on flash :(
```
sudo dphys-swapfile swapoff && \
sudo systemctl disable dphys-swapfile && \
sudo dphys-swapfile uninstall
sudo apt install fail2ban -y
```

### 1. create a new user to be responsible for your media-center
```
sudo useradd -u 1001 media-center
sudo usermod -a -G video media-center
sudo usermod -a -G render media-center
```

### 2. setup your external storage (big volume) under /srv/media-center
Edit your fstab file and add
```
storinator.lan:/volumeUSB1/usbshare /srv/media-center nfs,username=USER,password=PASS 0 0
```
TODO: Fix fstab. For now do manually:
```
sudo mount -t nfs -O user=USER,pass=PASS,gid=media-center,uid=media-center storinator.lan:/volumeUSB1/usbshare /srv/media-center
```

### 3. accessibility and routing
* Make your server accessible from anywhere:
https://www.noip.com/

* Setup port-forward from your router to your server as follows:
```
forward external-ip:80->raspberryPi:80
forward external-ip:443->raspberryPi:443
```

Because we are using 1 Cluster RPI, we use the RPI as a node too.
We need to untaint the node as master only with:
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### 4. create a kubernetes cluster
I used kubeadm instead of k3s because i found some problems that i could not easily solve.
This is mostly relative to the lack of support that the RaspbianOS 64bit image had at the time.
I followed this https://opensource.com/article/20/6/kubernetes-raspberry-pi tutorial and i made it work.
Also this might be helpfull https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

### 5. install helm
Add this line to .bashrc
```
export PATH=$PATH:$HOME/.local/bin
```
and run the commands bellow
```
wget https://get.helm.sh/helm-v3.6.1-linux-arm64.tar.gz
tar zxvf helm-*-linux-arm64.tar.gz
mkdir $HOME/.local/bin
mv linux-arm64/helm $HOME/.local/bin/helm
rm -rf helm-v* linux-arm64
```

## PreInstall
```
TODO: Needs reference on sops, base-auth setup and other stuff
```

## Install
```
git clone https://github.com/dioguerra/pyMediaCenter.git
cd pyMediaCenter
helm dependency update
kubectl create ns media-center
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager --namespace kube-system --set installCRDs=true
helm upgrade -i media-center ../pyMediaCenter --namespace media-center --values <OVERWRITE VALUES HERE>
```

### Add missing bits
sudo mkdir -p /srv/.config/{radarr,sonarr,transmission,jackett,jellyfin} && sudo chown media-center:media-center /srv/.config -R

TODO: Because the nginx path routes to the different components, we need to setup or provide a
base configuration for the servers

TODO: cluster dns not working for cluster services

TODO: Add proxy service and proxy rules to use for downloads

## Map of the exposed services
```
. 80 -> 433
├── /radarr
├── /sonarr
├── /transmission
├── /jackett (still no)
└── jellyfin
    ├──/
    └──/jellyfin
```

## File Tree
```
.
├── Chart.lock
├── charts
│   ├── ingress-nginx-3.22.0.tgz
│   ├── jellyfin-5.1.0.tgz
│   ├── radarr-9.2.0.tgz
│   ├── sonarr-9.2.0.tgz
│   └── transmission-2.1.0.tgz
├── Chart.yaml
├── LICENSE
├── README.md
├── templates
│   ├── cert-manager-issuer.yaml
│   └── nginx-basic-auth.yaml
└── values.yaml
```
