## DCOS setup:

For DCOS you need minimum of 3 nodes (1 boot, 1 master, 1 agent)

### Reimge your servers

```bash
/cs-shared/server-manager/client/server-manager reimage --server_id <server_id> centos-7.5
```

### Setup boot node
```bash
localectl set-locale LANG=en_US.utf8
mkdir genconf

cat > ./genconf/config.yaml <<EOF
---
cluster_name: dcos-contrail
bootstrap_url: http://<boot-ip>
exhibitor_storage_backend: static
master_discovery: static
master_list:
- <master-ip>
resolvers:
- 8.8.8.8
superuser_username: admin
superuser_password_hash: "$6$rounds=656000$123o/Qz.InhbkbsO$kn5IkpWm5CplEorQo7jG/27LkyDgWrml36lLxDtckZkCxu22uihAJ4DOJVVnNbsz/Y5MCK3B1InquE6E7Jmh30"
ssh_port: 22
ssh_user: root
agent_list:
- <agent-ip1>
- <agent-ip2>
public_agent_list:
- <public-agent-ip1>
- <public-agent-ip2>
check_time: false
exhibitor_zk_hosts: <boot-ip>:2181
EOF

cat > ./genconf/ip-detect <<EOF
#!/usr/bin/env bash
set -o errexit
set -o nounset
set -o pipefail
echo $(/usr/sbin/ip route show to match 192.168.65.90 | grep -Eo '[0-9]{1,3}.[0-9]{1,3}.[0-9]{1,3}.[0-9]{1,3}' | tail -1)
EOF

```

Configure parameters in you genconf/config.yaml then run following:
```bash
curl -O https://downloads.dcos.io/dcos/stable/dcos_generate_config.sh 
sudo bash dcos_generate_config.sh    
sudo docker run -d -p 80:80 -v $PWD/genconf/serve:/usr/share/nginx/html:ro nginx
```

### Setup node environment both master and agent
```bash
cat > run.sh <<EOF
sudo echo 'overlay' >> /etc/modules-load.d/overlay.conf && sudo modprobe overlay

sudo yum update --exclude=docker-engine,docker-engine-selinux,centos-release* --assumeyes --tolerant

yum install yum-utils -y


sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo && sudo yum install docker-ce -y && \
sudo systemctl start docker && sudo systemctl enable docker

sudo systemctl stop firewalld && sudo systemctl disable firewalld

localectl set-locale LANG=en_US.utf8

sudo yum install -y tar xz unzip curl ipset bind-utils

sudo sed -i s/SELINUX=enforcing/SELINUX=permissive/g /etc/selinux/config &&
sudo groupadd nogroup &&
sudo groupadd docker &&
sudo reboot

EOF

chmod +x run.sh
./run.sh
```

### Setup master

```bash 
mkdir /tmp/dcos && cd /tmp/dcos
curl -O http://<boot_ip>:80/dcos_install.sh
sudo bash dcos_install.sh master
```

### Setup node/agent

```bash 
mkdir /tmp/dcos && cd /tmp/dcos
curl -O http://<boot_ip>:80/dcos_install.sh
sudo bash dcos_install.sh slave_public
```


#### Some troubleshooting cmds:
```bash
curl -fsSL http://localhost:8181/exhibitor/v1/cluster/status | python -m json.tool
-b gives you logs from boot time.
journalctl -flu dcos-net
journalctl -flu dcos-mesos-dns
journalctl -flu dcos-mesos-master
journalctl -u dcos-adminrouter -b
journalctl -u dcos-marathon -b
journalctl -u dcos-exhibitor -b
journalctl -u dcos-mesos-dns -b
journalctl -flu dcos-spartan
```




