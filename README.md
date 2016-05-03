# BOSH Release for ClusterHQ Flocker
![sample1](https://docs.clusterhq.com/en/latest/_static/images/logo@2x.png)

## Purpose
ClusterHQ Flocker is a very nice tool to manage STATEFULL Docker container. It enables the persistent volumes provisioning for containers, and is able to relocate this volume between different host.
Its most usefull when associated with a container scheduler, like Docker Swarm, Mesos, or Kubernetes.

Bosh is the general tool for Iaas Management, used for Cloudfoundry and most cloudfoundry-community provisioning and management.

A generic Docker container management Bosh release is available. It can provision single containers, swarm managed containers, and on demand Cloudfoundry integrated services (CF Service Broker).
https://github.com/cloudfoundry-community/docker-boshrelease

This gives a very nice integration:
- enables on the shelf docker image reuse for bosh operators
- keep benefits of Bosh brilliant features (lifecycle management, health monitoring, versioned reproductible deployment)
- the swarm integration can provision stateless containers to be scheduled on multiple VMs Bosh deployment  

However, by design, Bosh manages a fixed number of external disk per VM. The above feature are not usable with statefull containers (eg: mongo/mysql/ ...)

The purpose of this release is to raise this storage limitations, providing an integration of ClusterHQ Flocker with Bosh Docker Release.


## Design
The design is an bosh context adaption of the clusterHQ documentation https://docs.clusterhq.com/en/latest/flocker-standalone/manual-install.html
This release must be used with the docker bosh release. https://github.com/cloudfoundry-community/docker-boshrelease

It enables on the fly persistent volume creation for docker containers
On OpenStack, it creates dedicated volumes for docker containers, enabling HA and Docker Swarm clustering while keeping container persistent data
https://docs.clusterhq.com/en/latest/flocker-features/openstack-configuration.html


### implementation and status

This is a very early stage prototype
The bosh release has been created with the nice bosh-gen tool https://github.com/cloudfoundry-community/bosh-gen
To ease development, flocker services are themselves deployed as docker containers.
The images are the clustehq images, initially designed for CoreOs Flocker prototyping. https://docs.clusterhq.com/en/latest/supported/flockercontainers.html#flocker-containers 

The flocker containers services need high level of control of the bosh vms (host networking, priviliged access to create the device on node vm)

Tested on :
- bosh 255
- openstack Juno Iaas
- docker bosh release 23
- stemcell 3215


### configuration
To generate the necessary certificates for the flocker deployment, use flocker-ca command see
- https://docs.clusterhq.com/en/latest/docker-integration/configuring-authentication.html
- https://docs.clusterhq.com/en/latest/docker-integration/generate-api-certificates.html
- https://docs.clusterhq.com/en/latest/docker-integration/generate-api-plugin.html 

creates a bosh deployment, including docker-bosh-release and flocker-boshrelease

- the template flocker_control provisions the necessary configuration for flocker_control service
- the template flocker_node provisions the required configuration for node level service (flocker_dataset_agent / ...)
- configure flocker_control container, mounting /var/vcap/jobs/flocker_control/config as /etc/flocker, for master vm
- configure flocker_node containers, mounting /var/vcap/jobs/flocker_node/config as /etc/flocker, for node vm


Here is a sample of a flocked enabled deployment. Must set 2 templates do provision the certificates, and associate with docker bosh release containers template

```
releases:
  - {name: docker, version: latest }
  - {name: flocker, version: latest }

...

jobs:
  - name: flocker-control
    templates:
      - {release: docker, name: docker}
      - {release: docker, name: containers}
      - {release: flocker, name: flocker_control}
    instances: 1
...
  - name: flocker-node
    templates:
      - {release: docker, name: docker}
      - {release: docker, name: containers}
      - {release: flocker, name: flocker_node}
    instances: 2

...


```

The flocker certs must be created with flocker-ca, and be set as properties


```
properties:
  flocker: 
    cluster_crt: |
      -----BEGIN CERTIFICATE-----
      MIIFdjCCA14CEQD4g6AbRTjrIKYvDy0y8W2bMA0GCSqGSIb3DQEBCwUAMEMxLTAr
      a0G0K8OenfR2gw==                                                
      -----END CERTIFICATE-----                                       
    cluster_key: |                                                    
      -----BEGIN PRIVATE KEY-----                                             
      MIIJQgIBADANBgkqhkiG9w0BAQEFAASCCSwwggkoAgEAAoICAQCYEn+mw9AhXV9/        
      Yper6nXKYXk0H6ku1DCkX2Vydorl8g==                                        
      -----END PRIVATE KEY-----                                               
    control_crt: |                                                            
      -----BEGIN CERTIFICATE-----                                             
      MIIFOTCCAyECEAoHuLLhDkCW/59vxSy720AwDQYJKoZIhvcNAQELBQAwQzEtMCsG        
      E08zi4lomI65YCScSfv+lfaHAdIXiPq9gcMC70tRT7ZHWDMYD28MLI5aQrvd            
      -----END CERTIFICATE-----                                               
                                                                              
    control_key: |                                                            
      -----BEGIN PRIVATE KEY-----                                                         
      MIIJRAIBADANBgkqhkiG9w0BAQEFAASCCS4wggkqAgEAAoICAQC4zSXaAI4pVhU5                    
      KQfYgMzL/g8I49Oeo3cONhFg6plsWcUn                                                    
      -----END PRIVATE KEY-----                                                           
                                                                                          
    client_crt: |                                                                         
      -----BEGIN CERTIFICATE-----                                                         
      MIIFMTCCAxkCEQCzYEfkFFn6BYEf7PGTQGZ1MA0GCSqGSIb3DQEBCwUAMEMxLTAr                    
      icQXlXVk4eDRe1JLT08RsfflTUWsbUDdbgXk+YNBWyV2Q6OLaA==                                
      -----END CERTIFICATE-----                                                           

    client_key: |
      -----BEGIN PRIVATE KEY-----                                                    
      MIIJQwIBADANBgkqhkiG9w0BAQEFAASCCS0wggkpAgEAAoICAQDhOwpOoSSQa7j6               
      9Mhn32fEZNeMVhBCvvZTBL1q9byM4Hg=                                               
      -----END PRIVATE KEY-----                                                      
                                                                                     
    plugin_crt: |                                                                    
      -----BEGIN CERTIFICATE-----                                                    
      MIIFKTCCAxECEQCePlOpibiHb6P9/kMjQVzFMA0GCSqGSIb3DQEBCwUAMEMxLTAr               
      tAlM6oIVaxIyWJdZGbtfUQ/pFJlFumvNU5fC7/4=                                       
      -----END CERTIFICATE-----                                                      

    plugin_key: |
      -----BEGIN PRIVATE KEY-----                                            
      MIIJRAIBADANBgkqhkiG9w0BAQEFAASCCS4wggkqAgEAAoICAQDU9yMv83Ufdce8       
      tqByDl2I6Me9cbvQd1ohciKO7rdP7AvL                                       
      -----END PRIVATE KEY-----                                              
                                                                             
                                                                             
    node:                                                                    
      cert: |                                                                
        -----BEGIN CERTIFICATE-----                                          
        MIIFLjCCAxYCEQCM6a2b4nGWHdO2yt0k0yQOMA0GCSqGSIb3DQEBCwUAMEMxLTAr     
        KguBdn9C1MK6iko04I3PSayRF/ZzuZVTpT4ZFvrumZRzDw==                     
        -----END CERTIFICATE-----                                            
      key: |                                                                 
        -----BEGIN PRIVATE KEY-----                                          
        MIIJQQIBADANBgkqhkiG9w0BAQEFAASCCSswggknAgEAAoICAQC9T1oyGaSJ3xqx     
        eLbwXt5BelA/Tqmp1DJcnC/SlcZD                                         
        -----END PRIVATE KEY-----                                            
```

You must configure flocker storage backend (agent.yml)

```
properties:
  flocker:
    node:                                                                             
      agent:
        control_service:
          hostname: <flocker control service address>
        dataset:
          backend: "openstack"
          region: "REG"
          auth_plugin: "v2password"
          auth_url: <https openstack identity v2 url>  #<--- Replace with OpenStack Identity API endpoint
          tenant_name: <tenant> # <--- Replace with OpenStack tenant name
          username: <user> # <--- Replace with OpenStack username
          password: <password> # <--- Replace with OpenStack password

```

Configure the flocker containers.
On flocker_control job

```
    properties:
      containers:
        # docker run --name=flocker-control-volume -v /var/lib/flocker clusterhq/flocker-control-service true
        name: flocker_control_volume 
          image: "clusterhq/flocker-control-service"
          restart: no
          bind-volumes:
          - "/var/lib/flocker"
          command: "true"

      # docker run --restart=always -d --net=host -v /etc/flocker:/etc/flocker --volumes-from=flocker-control-volume --name=flocker-control-service clusterhq/flocker-control-service
        - name: flocker_control_service 
          image: "clusterhq/flocker-control-service"
          net: host
          volumes:
           - "/var/vcap/jobs/flocker_control/config:/etc/flocker"
          volumes_from:
          - "flocker_control_volume"

```


On flocker_node job
```
      containers:
        #docker run --restart=always -d --net=host --privileged -v /flocker:/flocker -v /:/host -v /etc/flocker:/etc/flocker -v /dev:/dev --name=flocker-dataset-agent clusterhq/flocker-dataset-agent
        - name: flocker_dataset_agent
          image: "clusterhq/flocker-dataset-agent:1.10.2"
          net: host
          privileged: true
          volumes:
          - "/flocker:/flocker" 
          - "/:/host"
          - "/var/vcap/jobs/flocker_node/config:/etc/flocker"
          - "/dev:/dev"

        # docker run --restart=always -d --net=host --privileged -v /etc/flocker:/etc/flocker -v /var/run/docker.sock:/var/run/docker.sock --name=flocker-container-agent clusterhq/flocker-container-agent
        - name: flocker_container_agent
          image:  clusterhq/flocker-container-agent:1.10.2
          net: host
          privileged: true
          volumes:
          - "/var/vcap/jobs/flocker_node/config:/etc/flocker"
          - "/var/vcap/sys/run/docker/docker.sock:/var/run/docker.sock"  #target the docker boshrelease sock file location


          #docker run --restart=always -d --net=host -e FLOCKER_CONTROL_SERVICE_BASE_URL=<Control-Service-Host-DNS-or-IP>:4523/v1 -e MY_NETWORK_IDENTITY=<Host-IP-Address> -v /etc/flocker:/etc/flocker -v /run/docker:/run/docker --name=flocker-docker-plugin clusterhq/flocker-docker-plugin
        - name: flocker_docker_plugin
          image: clusterhq/flocker-docker-plugin:1.10.2
          net: host
          volumes:
          - "/var/vcap/jobs/flocker_node/config:/etc/flocker"
          - "/run/docker:/run/docker"
          env_vars:
            - "FLOCKER_CONTROL_SERVICE_BASE_URL=https://<flocker_control_address>:4523/v1" #<Control-Service-Host-DNS-or-IP>
            - "MY_NETWORK_IDENTITY=<node host IP>"  # <Host-IP-Address> 

```


Finally, configure your statefull container, so it leverages flocker volume-driver plugin

```
        # container with flocker docker volume driver
        - name: xxx
          image: "xxx:2"
          bind_ports:
            - "3206:3206"
          volumes:
          - "volumename:/var/lib/yyyy"
          #this activates the flocker docker plugin
          volume_driver: flocker

```


## Usage

To use this bosh release, first upload it to your bosh:

```
bosh target BOSH_HOST
git clone https://github.com/cloudfoundry-community/flocker-boshrelease.git
cd flocker-boshrelease
bosh upload release releases/flocker-1.yml
```

For [bosh-lite](https://github.com/cloudfoundry/bosh-lite), you can quickly create a deployment manifest & deploy a cluster:

```
templates/make_manifest warden
bosh -n deploy
```

For AWS EC2, create a single VM:

```
templates/make_manifest aws-ec2
bosh -n deploy
```

### Override security groups

For AWS & Openstack, the default deployment assumes there is a `default` security group. If you wish to use a different security group(s) then you can pass in additional configuration when running `make_manifest` above.

Create a file `my-networking.yml`:

``` yaml
---
networks:
  - name: flocker1
    type: dynamic
    cloud_properties:
      security_groups:
        - flocker
```

Where `- flocker` means you wish to use an existing security group called `flocker`.

You now suffix this file path to the `make_manifest` command:

```
templates/make_manifest openstack-nova my-networking.yml
bosh -n deploy
```

### Development

As a developer of this release, create new releases and upload them:

```
bosh create release --force && bosh -n upload release
```

### Final releases

To share final releases:

```
bosh create release --final
```

By default the version number will be bumped to the next major number. You can specify alternate versions:


```
bosh create release --final --version 2.1
```

After the first release you need to contact [Dmitriy Kalinin](mailto://dkalinin@pivotal.io) to request your project is added to https://bosh.io/releases (as mentioned in README above).
