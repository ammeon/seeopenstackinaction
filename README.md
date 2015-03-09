# See OpenStack in Action

##[See Openstack in Action Presentation](http://www.slideshare.net/AmmeonHR/ammeon-see-openstack-in-action) 

## Before running packstack
We are running Centos 7 on a tower server with 8 cores, 32 GB Ram and 2x 500GB HDDs
We have a single nic that we have configured with a vlan to provides our floating IP connectivity.
![](/home/keith/openstackinaction/images/server.png)

We installed Centos using lvm on the first disk and created the  volume group **cinder-volumes** on the second disk.
![](/home/keith/openstackinaction/images/disks.png) 


####Install Juno and Epel repos
```
yum install epel-release
yum install -y https://rdo.fedorapeople.org/rdo-release.rpm
```

#### Snapshotting
Before running packstack we snapshotted the system using lvm
```
lvcreate -s -n prepackstack -L 100G centos/root
```
To revert after something goes wrong
```
lvconvert --merge /dev/centos/prepackstack
reboot 
```

## Running packstack

```bash
packstack --answer-file=answer.txt
```
![](/home/keith/openstackinaction/images/packstack.png)


# Some post installation stepsNow

#### Sourcing keytstonerc_admin, we identify ourselves as admin for Openstack:
```bash
cd ~
. keystonerc_admin
```

####Starting with the configuration, we need an external network that allows us to connect to the instances, lets call it public
```bash
neutron net-create public --router:external=True
```
```
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | daf6f233-c3ce-43b3-ab1d-c29451c0b63f |
| name                      | public                               |
| provider:network_type     | vxlan                                |
| provider:physical_network |                                      |
| provider:segmentation_id  | 12                                   |
| router:external           | True                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 0d22b722258d4fdab1321fed13522b33     |
+---------------------------+--------------------------------------+
```

####Now, we define a subnet inside that network, where we will get our floating ips
```bash
neutron subnet-create --name public_subnet --enable_dhcp=False --allocation_pool start=10.248.22.230,end=10.248.22.250 --gateway=10.248.22.1 public 10.248.22.0/24
```
```
Created a new subnet:
+-------------------+----------------------------------------------------+
| Field             | Value                                              |
+-------------------+----------------------------------------------------+
| allocation_pools  | {"start": "10.248.22.230", "end": "10.248.22.250"} |
| cidr              | 10.248.22.0/24                                     |
| dns_nameservers   |                                                    |
| enable_dhcp       | False                                              |
| gateway_ip        | 10.248.22.1                                        |
| host_routes       |                                                    |
| id                | 15c57f4d-e2d3-4851-9d56-a8aa8af490b2               |
| ip_version        | 4                                                  |
| ipv6_address_mode |                                                    |
| ipv6_ra_mode      |                                                    |
| name              | public_subnet                                      |
| network_id        | 1f87a239-05f4-4229-ade9-2760785045bf               |
| tenant_id         | 0d22b722258d4fdab1321fed13522b33                   |
+-------------------+----------------------------------------------------+
```

####This is an optional step to have glance use swift.
```bash
PASS=$(grep ^admin_password /etc/glance/glance-api.conf | sed s/admin_password=//)
HOST=$(grep ^auth_host /etc/glance/glance-api.conf | sed s/auth_host=//)
echo "
default_store = swift
stores=glance.store.filesystem.Store,
       glance.store.http.Store,
       glance.store.swift.Store
swift_store_auth_address = http://$HOST:35357/v2.0
swift_store_user = services:glance
swift_store_key = $PASS
swift_store_create_container_on_put = True" >> /etc/glance/glance-api.conf
systemctl restart openstack-glance-api.service
```

####Loading the images to Openstack image service a.k.a. Glance
Cirros is mostly used for basic Openstack testing
```bash
glance image-create --name=cirros_0_3_3 --disk-format=qcow2 --container-format=bare --is-public=true --progress --copy-from http://download.cirros-cloud.net/0.3.3/cirros-0.3.3-x86_64-disk.img
```
```
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | None                                 |
| container_format | bare                                 |
| created_at       | 2015-03-09T10:17:56                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | 51b5d408-ffcc-4245-af28-c20b84d18e7e |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros_0_3_3                         |
| owner            | 0d22b722258d4fdab1321fed13522b33     |
| protected        | False                                |
| size             | 13200896                             |
| status           | queued                               |
| updated_at       | 2015-03-09T10:17:56                  |
| virtual_size     | None                                 |
+------------------+--------------------------------------+
```

And Fedora has one too, we'll use to install it to install our web app
```bash
glance image-create --name="Fedora 21" --disk-format=qcow2 --container-format=bare --is-public=true --progress --copy-from http://download.fedoraproject.org/pub/fedora/linux/releases/21/Cloud/Images/x86_64/Fedora-Cloud-Base-20141203-21.x86_64.qcow2
```
```
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | None                                 |
| container_format | bare                                 |
| created_at       | 2015-03-09T10:18:55                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | dcf5e8c1-9e05-442b-9b6e-1cfb03cbf843 |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | Fedora 21                            |
| owner            | 0d22b722258d4fdab1321fed13522b33     |
| protected        | False                                |
| size             | 158443520                            |
| status           | queued                               |
| updated_at       | 2015-03-09T10:18:55                  |
| virtual_size     | None                                 |
+------------------+--------------------------------------+
```

Checking the images
```bash
glance image-list
```
```
+--------------------------------------+--------------+-------------+------------------+-----------+--------+
| ID                                   | Name         | Disk Format | Container Format | Size      | Status |
+--------------------------------------+--------------+-------------+------------------+-----------+--------+
| 51b5d408-ffcc-4245-af28-c20b84d18e7e | cirros_0_3_3 | qcow2       | bare             | 13200896  | active |
| dcf5e8c1-9e05-442b-9b6e-1cfb03cbf843 | Fedora 21    | qcow2       | bare             | 158443520 | active |
+--------------------------------------+--------------+-------------+------------------+-----------+--------+
```

####Up till now we've needed admin privileges, now we're going to create a tenant and a user
```bash
keystone tenant-create --name=demo
```
```
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |                                  |
|   enabled   |               True               |
|      id     | f58038e6647949838d7e13649d803b60 |
|     name    |               demo               |
+-------------+----------------------------------+
```

####Creating the user associate to the demo_tenant
```bash
keystone user-create --name=demo --pass=demo --tenant=demo
```
```
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|  email   |                                  |
| enabled  |               True               |
|    id    | 423734d6ef7e406cb601bd7a2438f316 |
|   name   |               demo               |
| tenantId | f58038e6647949838d7e13649d803b60 |
| username |               demo               |
+----------+----------------------------------+
```

####Adding roles required for Heat and Swift
```bash
keystone user-role-add --user demo --role heat_stack_owner --tenant demo
```
```bash
keystone user-role-add --user demo --role SwiftOperator --tenant demo
```

####We create a keystonerc_demo file to identify easily ourselves as the new user
```bash
cp keystonerc_admin keystonerc_demo
sed -i 's/OS_USERNAME=admin/OS_USERNAME=demo/' keystonerc_demo
sed -i 's/OS_TENANT_NAME=admin/OS_TENANT_NAME=demo/' keystonerc_demo
sed -i 's/OS_PASSWORD=.*/OS_PASSWORD=demo/' keystonerc_demo
sed -i 's/keystone_admin/keystone_demo/' keystonerc_demo
```

####So, we're now demo_user against Openstack
```bash
. ~/keystonerc_demo
```

####First thing we'll do is to create a rsa key, that allows us to ssh to the instances without password
```bash
nova keypair-add demo_key > demo_key.pem
chmod 600 demo_key.pem
```

Checking that is loaded in identity service a.k.a. Keystone
```bash
nova keypair-list
```
```
+----------+-------------------------------------------------+
| Name     | Fingerprint                                     |
+----------+-------------------------------------------------+
| demo_key | aa:56:ab:c8:b0:a1:d4:d9:f0:a1:92:8e:7c:f7:0e:db |
+----------+-------------------------------------------------+
```

# Now lets create an instance

####We need an internal network, that our instances will plug directly to
```bash
neutron net-create demo-net
```
```
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| admin_state_up  | True                                 |
| id              | b3756aed-4553-40ca-bdf1-f9cb5a966149 |
| name            | demo-net                             |
| router:external | False                                |
| shared          | False                                |
| status          | ACTIVE                               |
| subnets         |                                      |
| tenant_id       | f58038e6647949838d7e13649d803b60     |
+-----------------+--------------------------------------+
```

####And same concept as before, a subnet so they get assigned ips
```bash
neutron subnet-create demo-net --name demo-subnet --gateway 10.0.0.1 10.0.0.0/24
```
```
+-------------------+--------------------------------------------+
| Field             | Value                                      |
+-------------------+--------------------------------------------+
| allocation_pools  | {"start": "10.0.0.2", "end": "10.0.0.254"} |
| cidr              | 10.0.0.0/24                                |
| dns_nameservers   |                                            |
| enable_dhcp       | True                                       |
| gateway_ip        | 10.0.0.1                                   |
| host_routes       |                                            |
| id                | efdb52b3-c112-4753-b352-8c24dbe21584       |
| ip_version        | 4                                          |
| ipv6_address_mode |                                            |
| ipv6_ra_mode      |                                            |
| name              | demo-subnet                                |
| network_id        | b3756aed-4553-40ca-bdf1-f9cb5a966149       |
| tenant_id         | f58038e6647949838d7e13649d803b60           |
+-------------------+--------------------------------------------+
```

####We need also a router, virtual one, so Openstack knows how to reach the instances
```bash
neutron router-create demo-router
```
```
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| external_gateway_info |                                      |
| id                    | 949cff7c-1add-43fc-be95-1f4fb9993b17 |
| name                  | demo-router                          |
| routes                |                                      |
| status                | ACTIVE                               |
| tenant_id             | f58038e6647949838d7e13649d803b60     |
+-----------------------+--------------------------------------+
```

####Attach the router to the demo-subnet
```bash
neutron router-interface-add demo-router demo-subnet
```
```
Added interface e643e295-e24d-4755-9f98-3e2e789c4025 to router demo-router.
```

####Plug the router to the external network by setting its gateway
```bash
neutron router-gateway-set demo-router public
```
```
Set gateway for router demo-router
```

####Rules are how we allow traffic inside Openstack, one to allow ssh:
```bash
nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
```
```
+-------------+-----------+---------+-----------+--------------+
| IP Protocol | From Port | To Port | IP Range  | Source Group |
+-------------+-----------+---------+-----------+--------------+
| tcp         | 22        | 22      | 0.0.0.0/0 |              |
+-------------+-----------+---------+-----------+--------------+
```

####And a second for ping
```bash
nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
```
```
+-------------+-----------+---------+-----------+--------------+
| IP Protocol | From Port | To Port | IP Range  | Source Group |
+-------------+-----------+---------+-----------+--------------+
| icmp        | -1        | -1      | 0.0.0.0/0 |              |
+-------------+-----------+---------+-----------+--------------+
```

####Instances are categorized in flavors, this is how Openstack knows resources like CPU and Mem
```bash
nova flavor-list
```
```
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
| 1  | m1.tiny   | 512       | 1    | 0         |      | 1     | 1.0         | True      |
| 2  | m1.small  | 2048      | 20   | 0         |      | 1     | 1.0         | True      |
| 3  | m1.medium | 4096      | 40   | 0         |      | 2     | 1.0         | True      |
| 4  | m1.large  | 8192      | 80   | 0         |      | 4     | 1.0         | True      |
| 5  | m1.xlarge | 16384     | 160  | 0         |      | 8     | 1.0         | True      |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
```

####Images are made available to the compute service codename: Nova
```bash
nova image-list
```
```
+--------------------------------------+--------------+--------+--------+
| ID                                   | Name         | Status | Server |
+--------------------------------------+--------------+--------+--------+
| dcf5e8c1-9e05-442b-9b6e-1cfb03cbf843 | Fedora 21    | ACTIVE |        |
| 51b5d408-ffcc-4245-af28-c20b84d18e7e | cirros_0_3_3 | ACTIVE |        |
+--------------------------------------+--------------+--------+--------+
```

####We're ready to launch our first instance
```bash
nova boot --flavor m1.tiny --image cirros_0_3_3 --security-group default --key-name demo_key demo-instance1
```
```
+--------------------------------------+-----------------------------------------------------+
| Property                             | Value                                               |
+--------------------------------------+-----------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                              |
| OS-EXT-AZ:availability_zone          | nova                                                |
| OS-EXT-STS:power_state               | 0                                                   |
| OS-EXT-STS:task_state                | scheduling                                          |
| OS-EXT-STS:vm_state                  | building                                            |
| OS-SRV-USG:launched_at               | -                                                   |
| OS-SRV-USG:terminated_at             | -                                                   |
| accessIPv4                           |                                                     |
| accessIPv6                           |                                                     |
| adminPass                            | wHJwL6QTUJVx                                        |
| config_drive                         |                                                     |
| created                              | 2015-03-09T10:31:21Z                                |
| flavor                               | m1.tiny (1)                                         |
| hostId                               |                                                     |
| id                                   | 39d3528e-3dcf-4a12-896f-771cb9a39008                |
| image                                | cirros_0_3_3 (51b5d408-ffcc-4245-af28-c20b84d18e7e) |
| key_name                             | demo_key                                            |
| metadata                             | {}                                                  |
| name                                 | demo-instance1                                      |
| os-extended-volumes:volumes_attached | []                                                  |
| progress                             | 0                                                   |
| security_groups                      | default                                             |
| status                               | BUILD                                               |
| tenant_id                            | f58038e6647949838d7e13649d803b60                    |
| updated                              | 2015-03-09T10:31:21Z                                |
| user_id                              | 423734d6ef7e406cb601bd7a2438f316                    |
+--------------------------------------+-----------------------------------------------------+
```

####Its alive!
```bash
watch -d -n1 -- nova list
```
```
+--------------------------------------+----------------+--------+------------+-------------+-------------------+
| ID                                   | Name           | Status | Task State | Power State | Networks          |
+--------------------------------------+----------------+--------+------------+-------------+-------------------+
| 39d3528e-3dcf-4a12-896f-771cb9a39008 | demo-instance1 | ACTIVE | -          | Running     | demo-net=10.0.0.2 |
+--------------------------------------+----------------+--------+------------+-------------+-------------------+
```

####In order to access the instance, we can either connect through vnc-console
```bash
nova get-vnc-console demo-instance1 novnc
```
```
+-------+------------------------------------------------------------------------------------+
| Type  | Url                                                                                |
+-------+------------------------------------------------------------------------------------+
| novnc | http://172.19.29.167:6080/vnc_auto.html?token=e4ce25e6-f642-447a-9439-a59523ce81e8 |
+-------+------------------------------------------------------------------------------------+
```

####Or to create a floating ip that can be accessed from outside world:
```bash
neutron floatingip-create public
```
```
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| fixed_ip_address    |                                      |
| floating_ip_address | 10.248.22.231                        |
| floating_network_id | 1f87a239-05f4-4229-ade9-2760785045bf |
| id                  | 7dbcce8a-4263-469a-ba9a-df7d7258f5e4 |
| port_id             |                                      |
| router_id           |                                      |
| status              | DOWN                                 |
| tenant_id           | f58038e6647949838d7e13649d803b60     |
+---------------------+--------------------------------------+
```

####Associate the ip with our instance
```bash
nova floating-ip-associate demo-instance1 10.248.22.231
```

####And access it with our previously generated key (ssh may take 10-20 secs to be available)
```bash
ssh -i demo_key.pem cirros@10.248.22.231
```
```lang-none
The authenticity of host '10.248.22.231 (10.248.22.231)' can't be established.
RSA key fingerprint is ff:4c:0d:3f:71:f5:a0:ba:33:e3:6a:d6:99:4e:0b:b0.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.248.22.231' (RSA) to the list of known hosts.
$
```

####This is a good time to take a look at Horizon using our **demo** user
![](/home/keith/openstackinaction/images/simple-instance.png) 

# Lets look at orchestration
#### Tune ceilometer to generate CPU samples every 5 seconds instead of 10 minutes
```bash
sed -i "/^.*name: cpu_source/ { n ; s/interval: 600$/interval: 5/ }" /etc/ceilometer/pipeline.yaml
```
####We need to restart ceilometer service so above changes are picked up
```bash
systemctl restart openstack-ceilometer-compute.service
```

####Preparing heat, source our demo_user credentials
```bash
. ~/keystonerc_demo
```

We need to add a new rule, this one will allow HTTP traffic to our instances
```bash
nova secgroup-add-rule default tcp 80 80 0.0.0.0/0
```
```
+-------------+-----------+---------+-----------+--------------+
| IP Protocol | From Port | To Port | IP Range  | Source Group |
+-------------+-----------+---------+-----------+--------------+
| tcp         | 80        | 80      | 0.0.0.0/0 |              |
+-------------+-----------+---------+-----------+--------------+
```

####We are ready to create the stack with the information we have
```bash
heat stack-create Demostack -f autoscaling.yaml -P key=demo_key -P flavor=m1.small -P image_id="Fedora 21" -P external_network=public -P internal_network_subnet="10.10.10.0/24"
```
```
+--------------------------------------+------------+--------------------+----------------------+
| id                                   | stack_name | stack_status       | creation_time        |
+--------------------------------------+------------+--------------------+----------------------+
| 97865354-e46b-4334-ba01-947f40ea958d | Demostack  | CREATE_IN_PROGRESS | 2015-03-09T14:39:48Z |
+--------------------------------------+------------+--------------------+----------------------+
```

####This will take a few minutes so have a look at the dashboard again
![](/home/keith/openstackinaction/images/stack.png) 
![](/home/keith/openstackinaction/images/topology.png) 

####Check the status of the load balancer's members, they need to be **ACTIVE**
```bash
watch -d -n1 -- neutron lb-member-list
```
```
+--------------------------------------+------------+---------------+--------+----------------+----------+
| id                                   | address    | protocol_port | weight | admin_state_up | status   |
+--------------------------------------+------------+---------------+--------+----------------+----------+
| 589c8c95-38c2-46b7-a187-f125071923f1 | 10.10.10.5 |            80 |      1 | True           | INACTIVE |
| 7d49fe8d-fd8e-487a-8830-82291bece652 | 10.10.10.3 |            80 |      1 | True           | INACTIVE |
+--------------------------------------+------------+---------------+--------+----------------+----------+
+--------------------------------------+------------+---------------+--------+----------------+--------+  
| id                                   | address    | protocol_port | weight | admin_state_up | status |  
+--------------------------------------+------------+---------------+--------+----------------+--------+  
| 589c8c95-38c2-46b7-a187-f125071923f1 | 10.10.10.5 |            80 |      1 | True           | ACTIVE |  
| 7d49fe8d-fd8e-487a-8830-82291bece652 | 10.10.10.3 |            80 |      1 | True           | ACTIVE |  
+--------------------------------------+------------+---------------+--------+----------------+--------+  
```

####Let's get the Floating IP associated to the Load Balancer's vip
```bash
heat output-show Demostack floating_ip
```
```
"10.248.22.241"
```

####We'll grab the floating ip in a local variable to make the next steps a bit easier
```bash
fip=$(heat output-show Demostack floating_ip | sed -e 's/^"//'  -e 's/"$//')
```

####Now we're ready to test http request to the instances, through the vip
```bash
while true; do curl $fip -m 2; sleep 0.5; done
```
```
Hello from instance with ip: 10.10.10.3
Hello from instance with ip: 10.10.10.5
Hello from instance with ip: 10.10.10.3
Hello from instance with ip: 10.10.10.5
Hello from instance with ip: 10.10.10.3
Hello from instance with ip: 10.10.10.5
Hello from instance with ip: 10.10.10.3
^C
```

####Time for some fun, put this in a web browser or curl it
http://$fip/cgi-bin/loadme


####One of the instances will start to be very busy (this is programmed to last for 3 minutes)
```bash
top
```
Press **1** for per CPU view

####Let's check if the other instance is up (We should see a new instance coming to play eventually)
```bash
while true; do curl $fip -m 2; sleep 0.5; done
```
```
curl: (28) Operation timed out after 2002 milliseconds with 0 out of -1 bytes received
Hello from instance with ip: 10.10.10.6
Hello from instance with ip: 10.10.10.3
curl: (28) Operation timed out after 2001 milliseconds with 0 out of -1 bytes received
Hello from instance with ip: 10.10.10.6
Hello from instance with ip: 10.10.10.3
```

####Checking the instances
```bash
nova list
```
```
+--------------------------------------+-------------------------------------------------------+--------+------------+-------------+-----------------------------------------------+
| ID                                   | Name           | Status | Task State | Power State | Networks                                      |
+--------------------------------------+-------------------------------------------------------+--------+------------+-------------+-----------------------------------------------+
| 5d9b2ca5-16b8-4d69-a333-7a1da5a7a6bd | De-njhk-ads    | ACTIVE | -          | Running     | Demostack-private_net-kh7vie446mvs=10.10.10.5 |
| 23795f78-50bd-4f55-b663-5605e1b94667 | De-njhk-vjn    | ACTIVE | -          | Running     | Demostack-private_net-kh7vie446mvs=10.10.10.6 |
| 3f6e0436-e765-4f20-a6e7-b9824fb41b8e | De-njhk-wqtijr6h4fxs-qebulwj44jek-server-jfipzqantnt3 | ACTIVE | -          | Running     | Demostack-private_net-kh7vie446mvs=10.10.10.3 |
| 39d3528e-3dcf-4a12-896f-771cb9a39008 | demo-instance1 | ACTIVE | -          | Running     | demo-net=10.0.0.2, 10.248.22.231              |
+--------------------------------------+-------------------------------------------------------+--------+------------+-------------+-----------------------------------------------+
```

####Downscale
And after 3 minutes, the instance with CPU spike stops being busy, and the new instance scales down
```bash
while true; do curl $fip -m 2; sleep 0.5; done
```
```
Hello from instance with ip: 10.10.10.5
Hello from instance with ip: 10.10.10.6
Hello from instance with ip: 10.10.10.5
Hello from instance with ip: 10.10.10.6
Hello from instance with ip: 10.10.10.5
```

####Checking the instances
```bash
nova list
```
```
+--------------------------------------+-------------------------------------------------------+--------+------------+-------------+-----------------------------------------------+
| ID                                   | Name           | Status | Task State | Power State | Networks                                      |
+--------------------------------------+-------------------------------------------------------+--------+------------+-------------+-----------------------------------------------+
| 5d9b2ca5-16b8-4d69-a333-7a1da5a7a6bd | De-njhk-ads    | ACTIVE | -          | Running     | Demostack-private_net-kh7vie446mvs=10.10.10.5 |
| 23795f78-50bd-4f55-b663-5605e1b94667 | De-njhk-vjn    | ACTIVE | -          | Running     | Demostack-private_net-kh7vie446mvs=10.10.10.6 |
| 39d3528e-3dcf-4a12-896f-771cb9a39008 | demo-instance1 | ACTIVE | -          | Running     | demo-net=10.0.0.2, 10.248.22.231              |
+--------------------------------------+-------------------------------------------------------+--------+------------+-------------+-----------------------------------------------+
```



####Lets expand our stack to two more instances
sed -i 's/min_size: 2/min_size: 4/' autoscaling.yaml

####Updating...
```bash
heat stack-update Demostack -f autoscaling.yaml -P internal_network=demo-subnet -P key=demo_key -P flavor=m1.small -P image_id="Fedora 21" -P external_network=public
```
```
+--------------------------------------+------------+--------------------+----------------------+
| id                                   | stack_name | stack_status       | creation_time        |
+--------------------------------------+------------+--------------------+----------------------+
| af4faa2e-eea9-466a-bdb0-5b2e903f67ea | Demostack  | UPDATE_IN_PROGRESS | 2015-03-09T14:45:41Z |
+--------------------------------------+------------+--------------------+----------------------+
```
```bash
nova list
```
```
+--------------------------------------+-------------------------------------------------------+--------+------------+-------------+-----------------------------------------------+
| ID                                   | Name           | Status | Task State | Power State | Networks                                      |
+--------------------------------------+-------------------------------------------------------+--------+------------+-------------+-----------------------------------------------+
| 4c5e2100-f68f-45f5-af4c-d693382d6a18 | De-njhk-5mk    | BUILD  | spawning   | NOSTATE     |                                               |
| 5d9b2ca5-16b8-4d69-a333-7a1da5a7a6bd | De-njhk-ads    | ACTIVE | -          | Running     | Demostack-private_net-kh7vie446mvs=10.10.10.5 |
| a2bb90d4-24c1-477b-828d-ee2b6e9aaedd | De-njhk-gty5   | BUILD  | spawning   | NOSTATE     |                                               |
| 23795f78-50bd-4f55-b663-5605e1b94667 | De-njhk-vjn    | ACTIVE | -          | Running     | Demostack-private_net-kh7vie446mvs=10.10.10.6 |
| 39d3528e-3dcf-4a12-896f-771cb9a39008 | demo-instance1 | ACTIVE | -          | Running     | demo-net=10.0.0.2, 10.248.22.231              |
+--------------------------------------+-------------------------------------------------------+--------+------------+-------------+-----------------------------------------------+
```