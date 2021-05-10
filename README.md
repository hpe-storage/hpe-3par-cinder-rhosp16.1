# HPE 3PAR and Primera Deployment Guide for RHOSP16.1

## Overview

This page provides detailed steps on how to enable the containerization of HPE 3PAR and Primera Cinder driver on top of the OSP Cinder images.
It also contains steps to deploy & configure HPE 3PAR and Primera backends for RHOSP16.1.

The custom Cinder container image contains following package:

python-3parclient 4.2.11

## Prerequisites

* Red Hat OpenStack Platform 16.1 with RHEL 8.2.

* HPE 3PAR array 3.3.1 OR HPE Primera array 4.2 or higher.

## Steps

### 1.	Prepare custom container

1.1	Create Dockerfile as described [here](https://github.com/hpe-storage/hpe-3par-cinder-rhosp16.1/blob/master/Dockerfile)

1.2	Login to Red Hat registry
```
sudo buildah login registry.redhat.io 
```

1.3	Build the podman image
```
sudo buildah bud --format docker --build-arg http_proxy=http://<proxy_ip>:8080 --build-arg https_proxy=http://<proxy_ip>:8080 . 
```
Notice the dot at the end.

1.4	Run podman images command to verify if the container got created successfully or not
```
sudo podman images
REPOSITORY                                           TAG                 IMAGE ID                       CREATED             SIZE
<none>                                               <none>              d599f4f06967                   21 seconds ago      1.18 GB
```

1.5	Add tag to the image created

In below command, cld13b4.ctlplane.set.rdlabs.hpecorp.net:8787 acts as local registry.

```
sudo podman tag <image id> cld13b4.ctlplane.set.rdlabs.hpecorp.net:8787/rhosp-rhel8/openstack-cinder-volume-hpe:latest
```

1.6	Run podman images command to verify the repository and tag is correctly updated to the docker image
```
sudo podman images
REPOSITORY                                                                                                            TAG                                IMAGE ID               CREATED                    SIZE
cld13b4.ctlplane.set.rdlabs.hpecorp.net:8787/rhosp-rhel8/openstack-cinder-volume-hpe                      latest                             d599f4f06967        2 minutes ago       1.18 GB
```

1.7	Push the container to a local registry
```
sudo openstack tripleo container image push --local cld13b4.ctlplane.set.rdlabs.hpecorp.net:8787/rhosp-rhel8/openstack-cinder-volume-hpe:latest
```

### 2.	Prepare the Environment Files for HPE 3PAR/Primera cinder-volume container and cinder backend

#### 2.1 Environment File for cinder-volume container

To use HPE 3PAR/Primera hardware as a Block Storage back end, cinder-volume container having python-3parclient needs to be deployed.

Procedure

Create a new container images file for your overcloud:

```
openstack tripleo container image prepare default \
    --local-push-destination \
    --output-env-file containers-prepare-parameter-hpe.yaml
```

Edit the containers-prepare-parameter-hpe.yaml file.

Add an exclude parameter to the strategy for the main Red Hat OpenStack Platform container images. 

```
parameter_defaults:
  ContainerImagePrepare:
    - push_destination: true
      excludes:
  	   - cinder-volume
      set:
        namespace: registry.redhat.io/rhosp-rhel8
        name_prefix: openstack-
        name_suffix: ''
        tag: 16.1
        ...
      tag_from_label: "{version}-{release}"
```

Add a new strategy to the ContainerImagePrepare parameter that includes the replacement container image for the HPE plugin:

```
parameter_defaults:
  ContainerImagePrepare:
    ...
    - push_destination: true
      includes:
        - cinder-volume
      set:
        namespace: registry.connect.redhat.com/hpe3parcinder
        name_prefix: openstack-
        name_suffix: -hpe3parcinder16-1
        tag: latest
        ...
```

Add the authentication details for the registry.connect.redhat.com registry to the ContainerImageRegistryCredentials parameter:

```
parameter_defaults:
  ContainerImageRegistryCredentials:
    registry.redhat.io:
      [service account username]: [service account password]
    registry.connect.redhat.com:
      [service account username]: [service account password]
```

Save the containers-prepare-parameter-hpe.yaml file.

Include the containers-prepare-parameter-hpe.yaml file with any deployment commands, such as as openstack overcloud deploy:

```
openstack overcloud deploy --templates
    ...
    -e containers-prepare-parameter-hpe.yaml
    ...
```

When director deploys the overcloud, the overcloud uses the HPE container image instead of the standard container image.

IMPORTANT
The containers-prepare-parameter-hpe.yaml file replaces the standard containers-prepare-parameter.yaml file in your overcloud deployment. Do not include the standard containers-prepare-parameter.yaml file in your overcloud deployment. Retain the standard containers-prepare-parameter.yaml file for your undercloud installation and updates.



#### 2.2 Environment File for cinder backend

The environment file is a OSP director environment file. The environment file contains the settings for each backend you want to define.

Create the environment file “cinder-hpe-[iscsi|fc].yaml” under /home/stack/templates/ with below parameters and other backend details.

```
parameter_defaults:
  CinderEnableIscsiBackend: false
  ControllerExtraConfig:
```

Sample files for both iSCSI and FC backends are available in [templates](https://github.com/traghavendra/hpe-3par-cinder-rhosp16.1/tree/main/templates) folder for reference.

#### Additional Help

For further details of HPE 3PAR and Primera cinder driver, kindly refer documentation [here](https://docs.openstack.org/cinder/victoria/configuration/block-storage/drivers/hpe-3par-driver.html)


### 3.	Deploy the overcloud and configured backends

After creating ```cinder-hpe-[iscsi|fc].yaml``` file with appropriate backends, deploy the backend configuration by running the openstack overcloud deploy command using the templates option.
Use the ```-e``` option to include the environment file ```cinder-hpe-[iscsi|fc].yaml```.

The order of the environment files (.yaml) is important as the parameters and resources defined in subsequent environment files take precedence.

```
openstack overcloud deploy --templates /usr/share/openstack-tripleo-heat-templates \
    -e /home/stack/templates/node-info.yaml \
    -e /home/stack/containers-prepare-parameter-hpe.yaml \
    -e /home/stack/custom_container/cinder-hpe-[iscsi|fc].yaml \
    --ntp-server <ntp_server_ip> \
    --debug
```

### 4.	Verify the configured changes

4.1	SSH to controller node from undercloud and check the docker process for cinder-volume
```
(overcloud) [heat-admin@overcloud-controller-0 ~]$ sudo podman ps | grep cinder
56baa616ae2c  cld13b4.ctlplane.set.rdlabs.hpecorp.net:8787/rhosp-rhel8/openstack-cinder-volume:16.1           /bin/bash /usr/lo...  2 weeks ago  Up 2 hours ago         openstack-cinder-volume-podman-0
11777e9848c1  cld13b4.ctlplane.set.rdlabs.hpecorp.net:8787/rhosp-rhel8/openstack-cinder-scheduler:16.1        kolla_start           2 weeks ago  Up 2 hours ago         cinder_scheduler
5d6d06cacc19  cld13b4.ctlplane.set.rdlabs.hpecorp.net:8787/rhosp-rhel8/openstack-cinder-api:16.1              kolla_start           2 weeks ago  Up 2 hours ago         cinder_api_cron
4043187451d2  cld13b4.ctlplane.set.rdlabs.hpecorp.net:8787/rhosp-rhel8/openstack-cinder-api:16.1              kolla_start           2 weeks ago  Up 2 hours ago         cinder_api
```

4.2.	Verify that the python-3parclient is present in the cinder-volume container
```
(overcloud) [heat-admin@overcloud-controller-0 ~]$ sudo podman exec -it 56baa616ae2c bash
()[root@overcloud-controller-0 /]# pip list | grep 3par
python-3parclient        4.2.11
```

4.3.	Verify that the backend details are visible in ```/etc/cinder/cinder.conf``` in the cinder-volume container

Given below is an example of FC backend details. Similar entries should be observed for iSCSI backend too.

```
[3parfc_1]
hpe3par_api_url=https://<3par_ip>:8080/api/v1
hpe3par_cpg=<cpg_name>
hpe3par_debug=True
hpe3par_password=<3par_username>
hpe3par_username=<3par_password>
san_ip=<3par_ip>
san_login=<3par_username>
san_password=<3par_password>
volume_backend_name=3parfc_1
volume_driver=cinder.volume.drivers.hpe.hpe_3par_fc.HPE3PARFCDriver
```

