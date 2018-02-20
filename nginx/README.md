# NGINX on Docker EE

- [NGINX on Docker EE](#nginx-on-docker-ee)
	- [Installation Pre-Requisites](#installation-pre-requisites)
	- [Routing Mesh Overview](#routing-mesh-overview)
	- [Enable the routing mesh in UCP](#enable-the-routing-mesh-in-ucp)
	- [Interlock/NGINX Manual Installation](#interlocknginx-manual-installation)
	- [Example Load Balancing](#example-load-balancing)

## Installation Pre-Requisites

- Docker version 17.06+ is required to use Interlock
- Docker must be running in Swarm mode
- Internet access (see Offline Installation for installing without internet access)

## Routing Mesh Overview

Docker Enterprise Edition has a routing mesh that allows you to make your services available to the outside world using a domain name. This is also known as a layer 7 load balancer.

![](/images/interlock-install-1.svg)

_Table 1\. Swarm routing mesh_

In this example, the WordPress service is being served on port 8000\. Users can access WordPress using the IP address of any node in the cluster and port 8000\. If WordPress is not running in that node, the request is redirected to a node that is.

Docker EE extends this and provides a routing mesh for application-layer load balancing. This allows you to access services with HTTP and HTTPS endpoints using a domain name instead of an IP.

![](/images/interlock-install-2.svg)

_Table 2\. http routing mesh_

In this example, the WordPress service listens on port 8000, but it is made available to the outside world as wordpress.example.org.

When users access wordpress.example.org, the HTTP routing mesh routes the request to the service running WordPress in a way that is transparent to them.

## Enable the routing mesh in UCP

The following To enable the HTTP routing mesh which will automatically provision the NGINX containers to your Swarm cluster.

Log in as an administrator, go to the UCP web UI, navigate to the Admin Settings page, and click the Routing Mesh option. Check the Enable routing mesh option.

![](/images/interlock-install-3.png)

_Table 2\. http routing mesh_

By default, the routing mesh service listens on port 80 for HTTP and port 8443 for HTTPS. Change the ports if you already have services that are using them.

View the Interlock services started by UCP running on your EE nodes.

```
$> docker service ls | grep interlock
1nwy84jpoxly        ucp-interlock-extension   replicated          1/1                 dockereng/ucp-interlock-extension:3.0.0-tp10
rth5tavyh0ke        ucp-interlock-proxy       replicated          2/2                 dockereng/ucp-interlock-proxy:3.0.0-tp10       *:80->80/tcp,*:8443->443/tcp
w72amyxljrp2        ucp-interlock             replicated          1/1                 dockereng/ucp-interlock:3.0.0-tp10
```

View the NGINX containers that were started by UCP running on your EE nodes.

```
$> docker ps -a | grep nginx
38a0821a7461   dockereng/ucp-interlock-proxy:3.0.0-tp10  "nginx -g 'daemon ..."  2 minutes ago  Up 2 minutes    80/tcp   swarm-0/ucp-interlock-proxy.2.9zwregnmn7k8h4pzl4v72v73u
ccc72c2e0a43   dockereng/ucp-interlock-proxy:3.0.0-tp10  "nginx -g 'daemon ..."  2 minutes ago  Up 2 minutes    80/tcp   swarm-1/ucp-interlock-proxy.1.qnjlsh998z2wpg6dqs7vc0v5r
```


## Interlock/NGINX Manual Installation

Interlock uses a configuration file for the core service. The following is an example config to get started. In order to utilize the deployment and recovery features in Swarm we will create a Docker Config object:

```
$> cat << EOF | docker config create service.interlock.conf -
ListenAddr = ":8080"
DockerURL = "unix:///var/run/docker.sock"
PollInterval = "3s"

[Extensions]
  [Extensions.default]
    Image = "interlockpreview/interlock-extension-nginx:2.0.0-preview"
    Args = ["-D"]
    ProxyImage = "nginx:alpine"
    ProxyArgs = []
    ProxyConfigPath = "/etc/nginx/nginx.conf"
    ServiceCluster = ""
    PublishMode = "ingress"
    PublishedPort = 80
    TargetPort = 80
    PublishedSSLPort = 443
    TargetSSLPort = 443
    [Extensions.default.Config]
      User = "nginx"
      PidPath = "/var/run/proxy.pid"
      WorkerProcesses = 1
      RlimitNoFile = 65535
      MaxConnections = 2048
EOF
oqkvv1asncf6p2axhx41vylgt
```

Next we will create a dedicated network for Interlock and the extensions:

$> docker network create -d overlay interlock
Now we can create the Interlock service. Note the requirement to constrain to a manager. The Interlock core service must have access to a Swarm manager, however the extension and proxy services are recommended to run on workers. See the Production section for more information on setting up for an production environment.

```
$> docker service create \
    --name interlock \
    --mount src=/var/run/docker.sock,dst=/var/run/docker.sock,type=bind \
    --network interlock \
    --constraint node.role==manager \
    --config src=service.interlock.conf,target=/config.toml \
    interlockpreview/interlock:2.0.0-preview -D run -c /config.toml
sjpgq7h621exno6svdnsvpv9z
```

There should be three (3) services created. One for the Interlock service, one for the extension service and one for the proxy service:

```
$> docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                                                       PORTS
lheajcskcbby        modest_raman        replicated          1/1                 nginx:alpine                                                *:80->80/tcp *:443->443/tcp
oxjvqc6gxf91        keen_clarke         replicated          1/1                 interlockpreview/interlock-extension-nginx:2.0.0-preview
sjpgq7h621ex        interlock           replicated          1/1                 interlockpreview/interlock:2.0.0-preview
```

The Interlock traffic layer is now deployed. 


## Example Load Balancing

Once Interlock has been deployed you are now ready to launch and publish applications. Using Service Labels the service is configured to publish itself to the load balancer.

Note: the examples below assume a DNS entry (or local hosts entry if you are testing local) exists for each of the applications.

To publish we will create a Docker Service using two labels:

- `com.docker.lb.hosts`
- `com.docker.lb.port`

The `com.docker.lb.hosts` label instructs Interlock where the service should be available. The `com.docker.lb.port` label instructs what port the proxy service should use to access the upstreams.

In this example we will publish a demo service to the `host demo.local`.

First we will create an overlay network so that service traffic is isolated and secure:

```
$> docker network create -d overlay demo
1ujq8o1fhvnj084eohejhmgur
```

Next we will deploy the application:

```
$> docker service create \
    --name demo \
    --network demo \
    --label com.docker.lb.hosts=demo.local \
    --label com.docker.lb.port=8080 \
    ehazlett/docker-demo
sq9ekdzyplchbjer1166320ca
```

Inspect the service `demo` created with the following:

```
docker service inspect demo
[
    {
        "ID": "sq9ekdzyplchbjer1166320ca",
        "Version": {
            "Index": 1818
        },
        "CreatedAt": "2018-02-20T19:52:00.078921041Z",
        "UpdatedAt": "2018-02-20T19:52:00.080925219Z",
        "Spec": {
            "Name": "demo",
            "Labels": {
                "com.docker.lb.hosts": "demo.local",
                "com.docker.lb.port": "8080",
                "com.docker.ucp.access.label": "/",
                "com.docker.ucp.collection": "swarm",
                "com.docker.ucp.collection.root": "true",
                "com.docker.ucp.collection.swarm": "true"
            },
            "TaskTemplate": {
                "ContainerSpec": {
                    "Image": "ehazlett/docker-demo:latest@sha256:2c63bd0546c8ff0471db37970b7b0d59ce45810d3351bc60658144c97453879a",
                    "Labels": {
                        "com.docker.ucp.access.label": "/",
                        "com.docker.ucp.collection": "swarm",
                        "com.docker.ucp.collection.root": "true",
                        "com.docker.ucp.collection.swarm": "true"
                    },
                    "StopGracePeriod": 10000000000,
                    "DNSConfig": {}
                },
                "Resources": {
                    "Limits": {},
                    "Reservations": {}
                },
                "RestartPolicy": {
                    "Condition": "any",
                    "Delay": 5000000000,
                    "MaxAttempts": 0
                },
                "Placement": {
                    "Constraints": [
                        "node.labels.com.docker.ucp.collection.swarm==true",
                        "node.labels.com.docker.ucp.orchestrator.swarm==true"
                    ],
                    "Platforms": [
                        {
                            "Architecture": "amd64",
                            "OS": "linux"
                        }
                    ]
                },
                "Networks": [
                    {
                        "Target": "1ujq8o1fhvnj084eohejhmgur"
                    }
                ],
                "ForceUpdate": 0,
                "Runtime": "container"
            },
            "Mode": {
                "Replicated": {
                    "Replicas": 1
                }
            },
            "UpdateConfig": {
                "Parallelism": 1,
                "FailureAction": "pause",
                "Monitor": 5000000000,
                "MaxFailureRatio": 0,
                "Order": "stop-first"
            },
            "RollbackConfig": {
                "Parallelism": 1,
                "FailureAction": "pause",
                "Monitor": 5000000000,
                "MaxFailureRatio": 0,
                "Order": "stop-first"
            },
            "EndpointSpec": {
                "Mode": "vip"
            }
        },
        "Endpoint": {
            "Spec": {
                "Mode": "vip"
            },
            "VirtualIPs": [
                {
                    "NetworkID": "1ujq8o1fhvnj084eohejhmgur",
                    "Addr": "10.0.1.2/24"
                }
            ]
        }
    }
]
```

Interlock will detect once the service is available and publish it. Once the tasks are running and the proxy service has been updated the application should be available via <http://demo.local>

```
$> curl -s -H "Host: demo.local" http://127.0.0.1/ping
{"instance":"b50025a35322","version":"0.1","request_id":"cf5c520ef77422f1390a6043f65e0ebe"}
```

To increase service capacity use the Docker Service Scale command:

```
$> docker service scale demo=4
demo scaled to 4
```

The four service replicas will be configured as upstreams. The load balancer will balance traffic across all service replicas.



