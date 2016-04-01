# BOSH Release for flocker



## Design
This bosh release is to be used with the docker bosh release.
It enables on the fly persistent volume creation for docker containers
On OpenStack, it creates dedicated volumes for docker containers, enabling HA and Docker Swarm clustering while keeping container persistent data
https://docs.clusterhq.com/en/latest/flocker-features/openstack-configuration.html


### implementation and status

This is a very early stage prototype
The bosh release has been created with the nice bosh-gen tool https://github.com/cloudfoundry-community/bosh-gen
To ease development, flocker services are themselves deployed as docker containers.
The images are the clustehq images, initially designed for CoreOs Flocker prototyping. https://docs.clusterhq.com/en/latest/supported/flockercontainers.html#flocker-containers 

The flocker containers services need high level of control of the bosh vms (host networking, priviliged access to create the device on node vm)


### prerequisite
generates necessary certificates for the flocker deployment
flocker-ca command see 

creates a bosh deployment, based on docker-bosh-release

the template flocker_control provisions the necessary configuration for flocker_control service
the template flocker_node provisions the required configuration for node level service (flocker_dataset_agent / ...)

configure flocker_control container, mounting /var/vcap/jobs/flocker_control/config as /etc/flocker, for master vm
configure flocker_node containers, mounting /var/vcap/jobs/flocker_node/config as /etc/flocker, for node vm


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

The flocker certs must be created elsewhere, and be set as properties


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
