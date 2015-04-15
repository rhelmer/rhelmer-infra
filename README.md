Terraform + Atlas Playground
----------------------------

First, you need to hook up Terraform and Atlas. You should already have 
an Atlas account and token. This token needs to be set whenever you want
to work with your Atlas account:

```
export ATLAS_TOKEN=...
```

Setting up the Consul cluster
=============================

First, hook the ```consul-cluster``` directory up to the
```${USER}/consul-cluster``` environment, and push the local state and
configuration up to Atlas:

```
pushd consul-cluster
terraform remote config -backend-config="name=${USER}/consul-cluster"
terraform push -name="${USER}/consul-cluster"
popd
```

Now review this environment in Atlas, you should see a Run that "NEEDS
USER ACTION": 

https://atlas.hashicorp.com/${USER}/environments/consul-cluster/changes

Click on that run and review the ```terraform plan``` output. If you like
what you see, click the "Confirm & Apply" button.

You need to log in and set up the cluster. Look in the AWS consul or use
the aws-cli to find the nodes spun up by your consul ASG, and ssh into each
one.

Put this into ```/etc/consul/server.json```, making sure to fill in the
```atlas_token``` with your actual token:

```
{
    "server": true,
    "bootstrap_expect": 3,
    "atlas_join": true,
    "atlas_token": "..."
    "atlas_infrastructure": "${USER}/consul-cluster"
}
```

Then set the hostname and reset consul. Setting a hostname is optional,
but makes things much clearer on the Atlas dashoard:

```
sudo hostname consul${n}
sudo systemctl restart consul
```

Now your consul servers should automatically discover each other via
Atlas.

Go to https://atlas.hashicorp.com/${USER}/environments/consul-cluster
and you should now see "Your infrastructure is healthy".

Setting up the Socorro Collector app
====================================

As above, set up Atlas as a terraform remote for the socorro-collector app:

```
pushd socorro-collector
terraform remote config -backend-config="name=${USER}/socorro-collector"
terraform push -name="${USER}/socorro-collector"
popd
```

FIXME You need to log into the box and join it to the consul cluster,
this should be made automatic but there's no provisioning in this repo yet:

```
# you can find the IP you need be looking at the Atlas e.g.:
# https://atlas.hashicorp.com/${USER}/environments/consul-cluster/dc1/nodes/consul1
consul join <any_consul_server_ip>
```

```consul members``` should show one client and 3 servers.

FIXME You can now bring up the app, as again there is no provisioning in this
repo yet you need to do it by hand:

```
sudo yum install nginx
sudo systemctl start nginx socorro-collector
```

You should be able to submit crashes and get a CrashID back:

```
curl -H 'Host: crash-reports' -F 'ProductName: Fake' -F 'Version=1.0' localhost/submit
```
