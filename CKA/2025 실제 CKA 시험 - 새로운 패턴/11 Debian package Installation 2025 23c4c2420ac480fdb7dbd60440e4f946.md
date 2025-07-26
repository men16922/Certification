# 11. Debian package Installation 2025

Weightage: 10%

Complete these tasks to prepare the system for Kubernetes:
Set up cri-dockerd:

Install the Debian package:
~/cri-dockerd_0.3.9.3-0.ubuntu-focal_amd64.deb

Start the cri-dockerd service.

Enable and start the systemd service for cri-dockerd.

Configure these system parameters:

Set net.bridge.bridge-nf-call-iptables to 1.

Set net.ipv6.conf.all.forwarding to 1.

Set net.ipv4.ip_forward to 1.

- [https://kubernetes.io/docs/setup/production-environment/container-runtimes/](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

```bash
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

# or
vi /etc/sysctl.d/k8s.conf

# Apply sysctl params without reboot
sudo sysctl --system

sysctl net.ipv4.ip_forward
sysctl net.ipv6.conf.all.forwarding
sysctl net.bridge.bridge-nf-call-iptables

cat /etc/sysctl.conf

apt update
curl -LO https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.9/cri-dockerd_0.3.9.3-0.ubuntu-focal_amd64.deb
dpkg -i ~/cri-dockerd_0.3.9.3-0.ubuntu-focal_amd64.deb

ls /etc/systemd/system/ | grep cri-dockerd
sudo systemctl status cri-docker.socket
systemctl list-units | grep cri

sudo systemctl start cri-docker.service
sudo systemctl enable cri-docker.service
sudo systemctl status cri-docker.service

```