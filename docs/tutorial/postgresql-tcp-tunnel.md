# Tutorial: Tunnel a private Postgresql database

In this tutorial we will tunnel Postgresql over inlets PRO to a remote machine. From there you can expose it to the Internet, or bind it to the local network for private VPN-like access.

> You can [subscribe to inlets for personal or commercial use via Gumroad](https://inlets.dev/blog/2021/07/27/monthly-subscription.html)

## Setup your exit node

Provision a cloud VM on DigitalOcean or another IaaS provider using [inletsctl](https://github.com/inlets/inletsctl):

```bash
inletsctl create \
 --provider digitalocean \
 --region lon1 \
 --pro
```

Note the `--url` and `TOKEN` given to you in this step.

## Run Postgresql on your private server

We can run a Postgresql instance using Docker:

```bash
head -c 16 /dev/urandom |shasum 
8cb3efe58df984d3ab89bcf4566b31b49b2b79b9

export PASSWORD="8cb3efe58df984d3ab89bcf4566b31b49b2b79b9"

docker run --rm --name postgres -p 5432:5432 -e POSTGRES_PASSWORD=8cb3efe58df984d3ab89bcf4566b31b49b2b79b9 -ti postgres:latest
```

## Connect the inlets PRO client

Fill in the below with the outputs you received from `inletsctl create`.

Note that `UPSTREAM="localhost"` can be changed to point at a host or IP address accessible from your client. The choice of `localhost` is suitable when you are running Postgresql in Docker on the same computer as the inlets PRO client.

The client will look for your license in `$HOME/.inlets/LICENSE`, but you can also use the `--license/--license-file` flag if you wish.

```bash
export EXIT_IP="134.209.21.155"
export TCP_PORTS="5432"
export LICENSE_FILE="$HOME/LICENSE.txt"
export TOKEN="KXJ5Iq1Z5Cc8GjFXdXJrqNhUzoScXnZXOSRKeh8x3f6tdGq1ijdENWQ2IfzdCg4U"
export UPSTREAM="localhost"

inlets-pro tcp client --connect "wss://$EXIT_IP:8123/connect" \
  --token "$TOKEN" \
  --upstream $UPSTREAM \
  --ports $TCP_PORTS
```

## Connect to your private Postgresql server from the Internet

You can run this command from anywhere, since your exit-server has a public IP:

```bash
export PASSWORD="8cb3efe58df984d3ab89bcf4566b31b49b2b79b9"
export EXIT_IP="209.97.141.140"

docker run -it -e PGPORT=5432 -e PGPASSWORD=$PASSWORD --rm postgres:latest psql -U postgres -h $EXIT_IP
```

Try a command such as `CREATE database` or `\dt`.

## Treat the database as private - like a VPN

[A split data and control-plane mean that tunnels do not need to be exposed on the Internet and can replace a VPN or a bespoke solution with SSH tunnels](https://raw.githubusercontent.com/inlets/inlets-pro/master/docs/images/inlets-pro-split-plane.png)

> A split data and control-plane mean that tunnels do not need to be exposed on the Internet and can replace a VPN or a bespoke solution with SSH tunnels

If you would like to keep the database service and port private, you can run the exit-server as a Pod in a Kubernetes cluster, or add an iptables rule to block access from external IPs.

Log into your exit-server and update `/etc/systemd/system/inlets-pro.service`

To listen on loopback, add: `--listen-data=127.0.0.1:`
To listen on a private adapter such as `10.1.0.10`, add: `--listen-data=10.1.0.10:`

Restart the service, and you'll now find that the database port `5432` can only be accessed from within the network you specified in `--listen-data`

Other databases such as Cassandra, MongoDB and Mysql/MariaDB also work exactly the same. Just change the port from `5432` to the port of your database.
