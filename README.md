# 1. Introduction
Mesos is built using the same principles as the Linux kernel, only at a different level of abstraction.
The Mesos kernel runs on every machine and provides applications (e.g., Hadoop, Spark, Kafka,
Elasticsearch) with APIs for resource management and scheduling across entire datacenter and cloud
 environments. Ref: http://mesos.apache.org/

# 2. Problem statement
Mesos containerized infrastructure has lowest elements as Application (group of tasks) and Pods.
There is a need to address networking segment (which includes network assignment, load balancing
and security) for seamless connectivity among pods and apps.

# 3. Proposed solution
Currently Mesos Marathon framework leverages custom network provider for different network providers.
We we use this to insert Contrail as networking components in the traditional Mesos infrastructure.
Contrail will bring in its features like virtual IPs, load balancer, network isolation, security and
policy based routing.

### Contrail as Network and IPAM provider
Contrail will assign IP to task & pod. Contrail will also attach them into its networking components
and would help route packets according to policies and route rules.

### 3.1 Task/Pod network assignment
As per the the task request Task or Pod will be assigned to the designated network and will attach
provided IP or IP from the subnet range.

```yaml
  "ipAddress": {
    "networkName": "contrail-cni-plugin",
    "labels": {
      "networks": "blue-network",
      "project-name": "default-project",
      "domain-name": "default-domain",
    }
  }
```

### 3.2 Public/floating IP
As per the request a task or pod can be allocated a public IP from the public network you can also
mention which IP to allocate from the public IP network subnet range.
```yaml
  "ipAddress": {
    "networkName": "contrail-cni-plugin",
    "labels": {
      "networks": "blue-network",
      "project-name": "default-project",
      "domain-name": "default-domain",
      "floating-ips": "default-domain:default-project:__public__:__fip_pool_public__(10.66.77.123),default-domain:default-project:__public__:__fip_pool_public2__(10.33.44.11)",
    }
  }
```

### 3.3 Assigning security group on interface
An interface can be assigned with a requested security group.
```yaml
  "ipAddress": {
    "networkName": "contrail-cni-plugin",
    "labels": {
      "networks": "blue-network",
      "project-name": "default-project",
      "domain-name": "default-domain",
      "security-groups": "default-domain:default-project:security_groups_mesos"
    }
  }
```

### 3.4 Load balancer and DNS support
Extending Contrail IP-fabric features Contrail will be able to support Mesos-DNS and Marathon-lb &/ dcos-layer4-lb.

### 3.5 A sample marathon input file:
You can also mention network.
```yaml
{
  "id": "/app-no-1",
  "acceptedResourceRoles": [
    "slave_public"
  ],
  "backoffFactor": 1.15,
  "backoffSeconds": 1,
  "container": {
    "type": "MESOS",
    "volumes": [],
    "docker": {
      "image": "ubuntu-upstart",
      "forcePullImage": true,
      "parameters": []
    }
  },
  "cpus": 0.1,
  "disk": 0,
  "instances": 1,
  "maxLaunchDelaySeconds": 3600,
  "mem": 128,
  "gpus": 0,
  "networks": [
    {
      "name": "contrail-cni-plugin",
      "mode": "container"
    }
  ],
  "requirePorts": false,
  "upgradeStrategy": {
    "maximumOverCapacity": 1,
    "minimumHealthCapacity": 1
  },
  "killSelection": "YOUNGEST_FIRST",
  "unreachableStrategy": {
    "inactiveAfterSeconds": 0,
    "expungeAfterSeconds": 0
  },
  "healthChecks": [],
  "fetch": [],
  "constraints": []
}
```
# 4. Implementation


## Architecture Diagram

```
+---------------------------------------------------+                +---------------------------------------------------+
|                                                   |                |                                                   |
|                                                   |                |                                                   |
|                                                   |                |                                                   |
|                                                   |                |                                                   |
|  +---------------+            +---------------+   |                |                          +-------------------+    |
|  |               | Updates    |               |   |                |                          |                   |    |
|  |               | via VNCAPIs|               |   |                |                          |  Mesos Executor   |    |
|  | Mesos Manager |            |    Contrail   |   |                |                          |                   |    |
|  |               |            |   Controller  +------------------------+                      |                   |    |
|  |               +----------> |               |   |                |   | Updates task IP      +--------+----------+    |
|  |               |            |               |   |                |   | info to agent                 | Invokes CNI   |
|  |               |            |               |   |                |   |                               | when task is  |
|  |               |            |               |   |                |   |                               v spawned       |
|  +------+--------+            +---------------+   |                |   |                                               |
|         ^                                         |                |   |                      +-------------------+    |
|         | Get task/pod                            |                |   |          Polls Agent |                   |    |
|         | info from API/Events                    |                |   |          for task info       CNI         |    |
|         |                                         |                |   |         +------------+                   |    |
|  +------+--------+                                |                |   v         v            |                   |    |
|  |               |                                |                |                          +-------------------+    |
|  |               |                                |                |  +--------------+                                 |
|  |               |                                |                |  |              |                                 |
|  |   Marathon    |                                |                |  |Contrail Agent|       DCOS/Mesos slave          |
|  |               |         DCOS/Mesos master      |                |  |              |       components are running    |
|  |               |         components are running |                |  |              |                                 |
|  |               |                                |                |  |              |                                 |
|  |               |                                |                |  |              |                                 |
|  +---------------+                                |                |  +------+-------+                                 |
|                                                   |                |         | Updates routing         "Slave Node"    |
|                                                   |                |         |                                         |
|                                                   |                +---------v-----------------------------------------+
|                                                   |                |                                                   |
|                               "Master Node"       |                | vRouter kernel module                "Kernel"     |
|                                                   |                |                                                   |
+---------------------------------------------------+                +---------------------------------------------------+

```
## Setup information : 
Setup is done in two parts DCOS installation and contrail installation. For DCOS setup you can follow \
this website : https://dcos.io/install/. For contrail installation follow : 
https://github.com/Juniper/contrail-ansible-deployer make sure you fill out inventory file and set \
orchestrator as mesos. 

Master Node consist of :
+ DCOS master components (https://docs.mesosphere.com/1.11/overview/architecture/components/)
+ Contrail master (Controller, Analytics, Config, UI)
+ Mesos Master

Slave/Agent Node consist of :
+ Contrail Agent
+ Contrail vRouter kernel module
+ Contrail CNI
+ DCOS slave components (https://docs.mesosphere.com/1.11/overview/architecture/components/)

## Components : 

### 4.1 Contrail controller
Contrail controller is the brain of contrail which does the decision making. You will find \
config management, analytics, UI and control place components for network virtualization. \
You can find more information at https://github.com/Juniper/contrail-controller. Contrail expose \
API for creating configuartion and updating virtual network components. In Mesos, mesos manager will \
update all information regarding task (universal docker) to Contrail Contraoller via API server. 
All Contrail controller components are micro service docker.

### 4.2 Mesos Manager
Mesos manager consist of two sub module :
 a. VNC server 
 b. Marathon API 

```
                           +----------------+
                           |                |
                           |                |
                           |                |
Interacts with             | Mesos Manager  |             Talks to Contrail VNC
marathon API & <-----------+                +-----------> API server
Server Side Events         |                |
                           |                |
                           |                |
                           |                |
                           |                |
                           +----------------+

```
 Mesos manager app runs inside a docker on master. Mesos manager app when its started it first tries to connect to \
 Marathon API server (master-ip:8080) and pulls all current running task. It parses only those tasks \
 which are registered as network "contrail-cni-plugin" and status as "TASK_RUNNING".
 More info on api at https://docs.mesosphere.com/1.11/deploying-services/marathon-api.
 Once it has all tasks data it check against its VNC cached DB and updates/deleted task info which is stale.
 
 ```
     [
      {
        "id": "string",
        "containers": [
          {
            "name": "string",
            ....
          }
        ],
        ....
        "networks": [
          {
            "name": "contrail-cni-plugin",
            "mode": "container",
            "labels": {
              "additionalProp1": "string",
              "additionalProp2": "string",
              "additionalProp3": "string"
            }
          }
        ],
        ...
      }
    ]
 ```
 Now it subscribe to Server Side Events from Marathon which is a event stream. More info at \
 https://mesosphere.github.io/marathon/docs/event-bus.html. We should be only subscribe to \
 status_update_event for task and specifically checking on taskStatus with "TASK_RUNNING", \
 "TASK_FINISHED", "TASK_FAILED", "TASK_KILLED" and for pods it would be pod_created_event, \
 pod_updated_event, pod_deleted_event. Filtering and subscribtion works as follow: 
 ```
 curl -H "Accept: text/event-stream"  <MARATHON_HOST>:<MARATHON_PORT>/v2/events?event_type=status_update_event\&event_type=pod_created_event\&event_type=pod_updated_event\&event_type=pod_deleted_event
 ```

### 4.1 Contrail CNI
Mesos agent would invoke Contrail CNI when custom/host network provider is mentioned in the task
description. CNI would parse all argument provided and pass required info to contrail's mesos manager.
CNI would then poll contrail agent for IP address and mac info and create a tap interface in container.
Loction is CNI is /opt/mesosphere/active/cni/contrail-cni-plugin and config is
/opt/mesosphere/etc/dcos/network/cni/contrail-cni-plugin.conf


### 4.3 DNS and load balancer
Mesos DNS and Mesos marathon lb would running as part of Contrail network so that resolved IP
address can be reached out via Contrail vRouter. Minuteman, spartan, Mesos DNS and Navstar should
would be configured to make them working.

# 5. Performance and scaling impact
Nothing so far.

# 6. Upgrade
Not applicable.

# 7. Deprecations
Not applicable.

# 8. Dependencies

# 9. Debugging

Curl to master IP to get status of all pods and apps.
```shell
curl http://{MASTER_IP}/marathon/v2/apps {/pods}
{
  "apps": [
    {
      "id": "/app-no-1",
      "acceptedResourceRoles": [
        "slave_public"
      ],
      "backoffFactor": 1.15,
      "backoffSeconds": 1,
      "container": {
        "type": "MESOS",
        "docker": {
          "forcePullImage": true,
          "image": "ubuntu-upstart",
          "parameters": [],
          "privileged": false
        },
        "volumes": [],
        "portMappings": [
          {
            "containerPort": 0,
            "labels": {},
            "name": "default",
            "protocol": "tcp",
            "servicePort": 10000
          }
        ]
      },
      "cpus": 0.1,
      "disk": 0,
      "executor": "",
      "instances": 1,
      "labels": {},
      "maxLaunchDelaySeconds": 3600,
      "mem": 128,
      "gpus": 0,
      "networks": [
        {
          "name": "contrail-cni-plugin",
          "mode": "container"
        }
      ],
      "requirePorts": false,
      "upgradeStrategy": {
        "maximumOverCapacity": 1,
        "minimumHealthCapacity": 1
      },
      "version": "2018-09-27T00:37:18.286Z",
      "versionInfo": {
        "lastScalingAt": "2018-09-27T00:37:18.286Z",
        "lastConfigChangeAt": "2018-09-27T00:37:18.286Z"
      },
      "killSelection": "YOUNGEST_FIRST",
      "unreachableStrategy": {
        "inactiveAfterSeconds": 0,
        "expungeAfterSeconds": 0
      },
      "tasksStaged": 0,
      "tasksRunning": 0,
      "tasksHealthy": 0,
      "tasksUnhealthy": 0,
      "deployments": [
        {
          "id": "7a72867d-1ad6-49c5-8321-c141f010466b"
        }
      ]
    }
  ]
}

```

You can also use dcos cli to retrive status.
```shell
dcos marathon app list --json

[
  {
    "acceptedResourceRoles": [
      "slave_public"
    ],
    "backoffFactor": 1.15,
    "backoffSeconds": 1,
    "container": {
      "docker": {
        "forcePullImage": true,
        "image": "ubuntu-upstart",
        "parameters": [],
        "privileged": false
      },
      "portMappings": [
        {
          "containerPort": 0,
          "labels": {},
          "name": "default",
          "protocol": "tcp",
          "servicePort": 10000
        }
      ],
      "type": "MESOS",
      "volumes": []
    },
    "cpus": 0.1,
    "deployments": [
      {
        "id": "7a72867d-1ad6-49c5-8321-c141f010466b"
      }
    ],
    "disk": 0,
    "executor": "",
    "gpus": 0,
    "id": "/app-no-1",
    "instances": 1,
    "killSelection": "YOUNGEST_FIRST",
    "labels": {},
    "maxLaunchDelaySeconds": 3600,
    "mem": 128,
    "networks": [
      {
        "mode": "container",
        "name": "contrail-cni-plugin"
      }
    ],
    "requirePorts": false,
    "tasksHealthy": 0,
    "tasksRunning": 0,
    "tasksStaged": 0,
    "tasksUnhealthy": 0,
    "unreachableStrategy": {
      "expungeAfterSeconds": 0,
      "inactiveAfterSeconds": 0
    },
    "upgradeStrategy": {
      "maximumOverCapacity": 1,
      "minimumHealthCapacity": 1
    },
    "version": "2018-09-27T00:37:18.286Z",
    "versionInfo": {
      "lastConfigChangeAt": "2018-09-27T00:37:18.286Z",
      "lastScalingAt": "2018-09-27T00:37:18.286Z"
    }
  }
]

```

CNI logs:
```shell
/var/log/contrail/cni/opencontrail.log
```

Mesos manager logs (inside container):
```shell
/var/log/contrail/contrail-mesos-manager.log
 ```

# 10. Testing
## 10.1 Unit tests
## 10.2 Dev tests
## 10.3 System tests

# 11. Documentation Impact

# 12. References
