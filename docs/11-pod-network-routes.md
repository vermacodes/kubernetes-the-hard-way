# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://cloud.google.com/compute/docs/vpc/routes).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VPC network.

Print the internal IP address and Pod CIDR range for each worker instance:

```bash
{
  SERVER_IP=$(grep server machines.txt | cut -d " " -f 1)
  NODE_0_IP=$(grep node-0 machines.txt | cut -d " " -f 1)
  NODE_0_SUBNET=$(grep node-0 machines.txt | cut -d " " -f 4)
  NODE_1_IP=$(grep node-1 machines.txt | cut -d " " -f 1)
  NODE_1_SUBNET=$(grep node-1 machines.txt | cut -d " " -f 4)
  NODE_2_IP=$(grep node-2 machines.txt | cut -d " " -f 1)
  NODE_2_SUBNET=$(grep node-2 machines.txt | cut -d " " -f 4)
}
```

```bash
ssh root@server <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
  ip route add ${NODE_2_SUBNET} via ${NODE_2_IP}
EOF
```

```bash
ssh root@node-0 <<EOF
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
  ip route add ${NODE_2_SUBNET} via ${NODE_2_IP}
EOF
```

```bash
ssh root@node-1 <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
  ip route add ${NODE_2_SUBNET} via ${NODE_2_IP}
EOF
```

```bash
ssh root@node-2 <<EOF
  ip route add ${NODE_0_SUBNET} via ${NODE_0_IP}
  ip route add ${NODE_1_SUBNET} via ${NODE_1_IP}
EOF
```

## Verification 

```bash
ssh root@server ip route
```

```text
default via XXX.XXX.XXX.XXX dev ens160 
10.200.0.0/24 via XXX.XXX.XXX.XXX dev ens160 
10.200.1.0/24 via XXX.XXX.XXX.XXX dev ens160 
10.200.2.0/24 via XXX.XXX.XXX.XXX dev ens160 
XXX.XXX.XXX.0/24 dev ens160 proto kernel scope link src XXX.XXX.XXX.XXX 
```

```bash
ssh root@node-0 ip route
```

```text
default via XXX.XXX.XXX.XXX dev ens160 
10.200.1.0/24 via XXX.XXX.XXX.XXX dev ens160 
10.200.2.0/24 via XXX.XXX.XXX.XXX dev ens160 
XXX.XXX.XXX.0/24 dev ens160 proto kernel scope link src XXX.XXX.XXX.XXX 
```

```bash
ssh root@node-1 ip route
```

```text
default via XXX.XXX.XXX.XXX dev ens160 
10.200.0.0/24 via XXX.XXX.XXX.XXX dev ens160 
10.200.2.0/24 via XXX.XXX.XXX.XXX dev ens160 
XXX.XXX.XXX.0/24 dev ens160 proto kernel scope link src XXX.XXX.XXX.XXX 
```

```bash
ssh root@node-2 ip route
```

```text
default via XXX.XXX.XXX.XXX dev ens160 
10.200.0.0/24 via XXX.XXX.XXX.XXX dev ens160 
10.200.1.0/24 via XXX.XXX.XXX.XXX dev ens160 
XXX.XXX.XXX.0/24 dev ens160 proto kernel scope link src XXX.XXX.XXX.XXX 
```

## Making Routes Persistent

The routes created above will not persist across reboots. To make them persistent, create a systemd service that runs after the network is ready.

Create the route configuration script and systemd service on each machine:

```bash
ssh root@server <<EOF
cat > /usr/local/bin/configure-kube-routes.sh <<'SCRIPT'
#!/bin/bash
LOGFILE="/var/log/kube-routes.log"

log() {
  echo "\$(date '+%Y-%m-%d %H:%M:%S') - \$1" | tee -a \$LOGFILE
}

log "Starting Kubernetes route configuration"

if ip route add 10.200.0.0/24 via ${NODE_0_IP} 2>&1 | tee -a \$LOGFILE; then
  log "Added route: 10.200.0.0/24 via ${NODE_0_IP}"
else
  log "Route already exists or failed: 10.200.0.0/24 via ${NODE_0_IP}"
fi

if ip route add 10.200.1.0/24 via ${NODE_1_IP} 2>&1 | tee -a \$LOGFILE; then
  log "Added route: 10.200.1.0/24 via ${NODE_1_IP}"
else
  log "Route already exists or failed: 10.200.1.0/24 via ${NODE_1_IP}"
fi

if ip route add 10.200.2.0/24 via ${NODE_2_IP} 2>&1 | tee -a \$LOGFILE; then
  log "Added route: 10.200.2.0/24 via ${NODE_2_IP}"
else
  log "Route already exists or failed: 10.200.2.0/24 via ${NODE_2_IP}"
fi

log "Kubernetes route configuration completed"
SCRIPT
chmod +x /usr/local/bin/configure-kube-routes.sh

cat > /etc/systemd/system/kube-routes.service <<'SERVICE'
[Unit]
Description=Configure Kubernetes pod network routes
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/configure-kube-routes.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
SERVICE

systemctl daemon-reload
systemctl enable kube-routes.service
systemctl start kube-routes.service
EOF
```

```bash
ssh root@node-0 <<EOF
cat > /usr/local/bin/configure-kube-routes.sh <<'SCRIPT'
#!/bin/bash
LOGFILE="/var/log/kube-routes.log"

log() {
  echo "\$(date '+%Y-%m-%d %H:%M:%S') - \$1" | tee -a \$LOGFILE
}

log "Starting Kubernetes route configuration"

if ip route add 10.200.1.0/24 via ${NODE_1_IP} 2>&1 | tee -a \$LOGFILE; then
  log "Added route: 10.200.1.0/24 via ${NODE_1_IP}"
else
  log "Route already exists or failed: 10.200.1.0/24 via ${NODE_1_IP}"
fi

if ip route add 10.200.2.0/24 via ${NODE_2_IP} 2>&1 | tee -a \$LOGFILE; then
  log "Added route: 10.200.2.0/24 via ${NODE_2_IP}"
else
  log "Route already exists or failed: 10.200.2.0/24 via ${NODE_2_IP}"
fi

log "Kubernetes route configuration completed"
SCRIPT
chmod +x /usr/local/bin/configure-kube-routes.sh

cat > /etc/systemd/system/kube-routes.service <<'SERVICE'
[Unit]
Description=Configure Kubernetes pod network routes
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/configure-kube-routes.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
SERVICE

systemctl daemon-reload
systemctl enable kube-routes.service
systemctl start kube-routes.service
EOF
```

```bash
ssh root@node-1 <<EOF
cat > /usr/local/bin/configure-kube-routes.sh <<'SCRIPT'
#!/bin/bash
LOGFILE="/var/log/kube-routes.log"

log() {
  echo "\$(date '+%Y-%m-%d %H:%M:%S') - \$1" | tee -a \$LOGFILE
}

log "Starting Kubernetes route configuration"

if ip route add 10.200.0.0/24 via ${NODE_0_IP} 2>&1 | tee -a \$LOGFILE; then
  log "Added route: 10.200.0.0/24 via ${NODE_0_IP}"
else
  log "Route already exists or failed: 10.200.0.0/24 via ${NODE_0_IP}"
fi

if ip route add 10.200.2.0/24 via ${NODE_2_IP} 2>&1 | tee -a \$LOGFILE; then
  log "Added route: 10.200.2.0/24 via ${NODE_2_IP}"
else
  log "Route already exists or failed: 10.200.2.0/24 via ${NODE_2_IP}"
fi

log "Kubernetes route configuration completed"
SCRIPT
chmod +x /usr/local/bin/configure-kube-routes.sh

cat > /etc/systemd/system/kube-routes.service <<'SERVICE'
[Unit]
Description=Configure Kubernetes pod network routes
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/configure-kube-routes.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
SERVICE

systemctl daemon-reload
systemctl enable kube-routes.service
systemctl start kube-routes.service
EOF
```

```bash
ssh root@node-2 <<EOF
cat > /usr/local/bin/configure-kube-routes.sh <<'SCRIPT'
#!/bin/bash
LOGFILE="/var/log/kube-routes.log"

log() {
  echo "\$(date '+%Y-%m-%d %H:%M:%S') - \$1" | tee -a \$LOGFILE
}

log "Starting Kubernetes route configuration"

if ip route add 10.200.0.0/24 via ${NODE_0_IP} 2>&1 | tee -a \$LOGFILE; then
  log "Added route: 10.200.0.0/24 via ${NODE_0_IP}"
else
  log "Route already exists or failed: 10.200.0.0/24 via ${NODE_0_IP}"
fi

if ip route add 10.200.1.0/24 via ${NODE_1_IP} 2>&1 | tee -a \$LOGFILE; then
  log "Added route: 10.200.1.0/24 via ${NODE_1_IP}"
else
  log "Route already exists or failed: 10.200.1.0/24 via ${NODE_1_IP}"
fi

log "Kubernetes route configuration completed"
SCRIPT
chmod +x /usr/local/bin/configure-kube-routes.sh

cat > /etc/systemd/system/kube-routes.service <<'SERVICE'
[Unit]
Description=Configure Kubernetes pod network routes
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/configure-kube-routes.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
SERVICE

systemctl daemon-reload
systemctl enable kube-routes.service
systemctl start kube-routes.service
EOF
```

Verify the systemd service was created and started:

```bash
for host in server node-0 node-1 node-2; do
  echo "=== ${host} ==="
  ssh root@${host} systemctl status kube-routes.service
done
```

Check the logs to ensure routes were configured successfully:

```bash
for host in server node-0 node-1 node-2; do
  echo "=== ${host} ==="
  ssh root@${host} cat /var/log/kube-routes.log
done
```

The routes will now be automatically configured on each boot after the network is online.

Next: [Smoke Test](12-smoke-test.md)
