
# 1. Introduction
Mesos is built using the same principles as the Linux kernel, only at a different level of abstraction. The Mesos
kernel runs on every machine and provides applications (e.g., Hadoop, Spark, Kafka, Elasticsearch) with API’s for
resource management and scheduling across entire datacenter and cloud environments. Ref: http://mesos.apache.org/ 

# 2. Problem statement
Mesos containerized infrastructure has lowest elements as Application (group of tasks) and Pods. There is a need to address networking segment (which includes network assignment, load balancing and security) for seamless connectivity among pods and apps.

# 3. Proposed solution
Currently Mesos Marathon framework leverages custom network provider for differnt network providers. We we use this to insert contrail as networking components in the traditional mesos infrastructure. Contrail will bring in its features like virtual ips, load balancer, network isolation, security and policy based routing.

### 3.1 Contrail as Network and IPAM provider
Contrail will assign IP's to task and pods also release when required. Contrail will plumb task and pods into its networking components and would help route packets according to 

### 3.2 Task/Pod network assignment
As per the the task request Task or Pod will be assigned to the designated network and will attach provided ip or ip from the subnet range.

```yaml
  "ipAddress": {
    "networkName": "contrail-cni-plugin",
    "labels": {
      "networks": "blue-network",
      "network-ip": "5.5.5.0/24",
      "project-name": "default-project",
      "domain-name": "default-domain",
    }
  }
```

### 3.3 Public/floating ip
As per the request a task or pod can be allocated a public ip from the public network you can also mention which ip to allocate from the public ip network subnet range.
```yaml
  "ipAddress": {
    "networkName": "contrail-cni-plugin",
    "labels": {
      "networks": "blue-network",
      "network-subnets": "5.5.5.0/24",
      "project-name": "default-project",
      "domain-name": "default-domain",
      "floating-ips": "default-domain:default-project:__public__:__fip_pool_public__(10.66.77.123),default-domain:default-project:__public__:__fip_pool_public2__(10.33.44.11)",
    }
  }
```  

### 3.4 Assigning security group on interface
An interface can be assign with a requested security group.
```yaml
  "ipAddress": {
    "networkName": "contrail-cni-plugin",
    "labels": {
      "networks": "blue-network",
      "network-subnets": "5.5.5.0/24",
      "project-name": "default-project",
      "domain-name": "default-domain",
      "security-groups": "default-domain:default-project:security_groups_mesos"
    }
  }
```  

# 4. Implementation
## 4.1 Work items
#### Describe changes needed for different components such as Controller, Analytics, Agent, UI. 
#### Add subsections as needed.

# 5. Performance and scaling impact
## 5.1 API and control plane
#### Scaling and performance for API and control plane

## 5.2 Forwarding performance
#### Scaling and performance for API and forwarding

# 6. Upgrade
#### Describe upgrade impact of the feature
#### Schema migration/transition

# 7. Deprecations
#### If this feature deprecates any older feature or API then list it here.

# 8. Dependencies
#### Describe dependent features or components.

# 9. Testing
## 9.1 Unit tests
## 9.2 Dev tests
## 9.3 System tests

# 10. Documentation Impact

# 11. References
