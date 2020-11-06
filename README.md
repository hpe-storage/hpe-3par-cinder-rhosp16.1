# HPE 3PAR and Primera Cinder volume custom container for RHOSP16.1

## Overview

This page provides detailed steps on how to enable the containerization of HPE 3PAR and Primera Cinder driver on top of the OSP Cinder images.

The custom Cinder container image contains following package:

python-3parclient 4.2.11

## Prerequisites

* Red Hat OpenStack Platform 16.1.

* HPE 3PAR array 3.3.1 OR HPE Primera array 4.0 or higher.

## Steps

### 1.	Prepare custom container

1.1	Create Dockerfile as described [here](https://github.com/hpe-storage/hpe-3par-cinder-rhosp16.1/blob/master/Dockerfile)

1.2	Login to Red Hat registry
```
sudo buildah login registry.redhat.io 
```

1.3	Build the podman image
```
sudo buildah bud --format docker --build-arg http_proxy=http://16.85.88.10:8080 --build-arg https_proxy=http://16.85.88.10:8080 . 
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

### 2.	Prepare custom environment yaml

Create new env file “custom_container_[iscsi|fc].yaml” under /home/stack/custom_container/ with only the custom container parameter and other backend details. Sample files are available in [custom_container](https://github.com/hpe-storage/hpe-3par-cinder-rhosp16.1/blob/master/custom_container) folder for reference
```
parameter_defaults:
    DockerCinderVolumeImage: cld13b4.ctlplane.set.rdlabs.hpecorp.net:8787/rhosp-rhel8/openstack-cinder-volume-hpe:latest
```

### 3.	Deploy the overcloud
```
openstack overcloud deploy --templates /usr/share/openstack-tripleo-heat-templates \
    -e /home/stack/templates/node-info.yaml \
    -e /home/stack/containers-prepare-parameter.yaml \
    -e /home/stack/custom_container/custom_container_[iscsi|fc].yaml \
    --ntp-server 16.110.135.123 \
    --debug
```

The order of the environment files (.yaml) is important as the parameters and resources defined in subsequent environment files take precedence.
The custom_container_[iscsi|fc].yaml is mentioned after containers-prepare-parameter.yaml so that custom container can be used instead of the default one.

### 4.	Verify the custom container

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
