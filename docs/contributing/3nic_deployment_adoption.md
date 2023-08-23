
# 3 NIC Deployment adoption

## Deploying director cloud
To deploy the environment run [this](https://rhos-ci-jenkins.lab.eng.tlv2.redhat.com/view/DFG/view/network/view/networking-ovn/job/DFG-network-neutron-17.1_director-rhel-virthost-3cont_2comp-ipv4-geneve-tobiko-neutron/) CI gate on a devnest node.

Once it has been deployed make sure machine has enough ressources (disk, ram and cpu) to handle the CRC machine.

## Lab preparation
Follow the section that you can find [here](https://docs.google.com/document/d/15x930hfe0qh9QZ6wU8lmW-3SXLwajzmSi8XBk88o6aw/edit) on the _Lab machine preparation_ section.

> _Warning 1_: Is it possible that you encounter this:  
> ```
> Error: Unable to find a match: guestfs-tools
> ```
> In order to solve it, install **libguestfs-tools**.

> _WARNING 2_: If you encounter problems while doing:   
>
> ```
> GO111MODULE=on go install sigs.k8s.io/kustomize/kustomize/v5@latest
> ```   
> 
> Update go using [update-golang](GO111MODULE=on go install sigs.k8s.io/kustomize/kustomize/v5@latest)
>    
> ```
> git clone https://github.com/udhos/update-golang
> cd update-golang
>  sudo ./update-golang.sh
> ```

## Development environment

In this section we'll create the CRC machine and configure it acording to our needs. (This section is based on this [development environment](https://openstack-k8s-operators.github.io/data-plane-adoption/contributing/development_environment/) guide, though we made some changes to adapt it to our environment.)

### Environment prep

Clone install_yamls:  
```
git clone https://github.com/openstack-k8s-operators/install_yamls.git
```

Install tools for operator development:

```
cd install_yamls/devsetup
make download_tools
```

### Deployment of CRC machine

```
cd install_yamls/devsetup
PULL_SECRET=$HOME/pull-secret.txt CPUS=12 MEMORY=40000 DISK=100 make crc

eval $(crc oc-env)
oc login -u kubeadmin -p 12345678 https://api.crc.testing:6443

make crc_attach_default_interface
```

The 3 NIC Wallaby deployment uses 3 different networks:

* data: This network is configured with no DHCP and using net: 192.168.24.0/24. Hosts 2 vlan networks:
    - vlan 20: internal api vlan, IP: 172.17.1.0/24
    - vlan 40: storage management vlan, IP: 172.17.4.0/24
* management: This network is configured with DHCP and using net: 172.16.0.0/24
* external: This network is configured with DHCP and usign net: 10.0.0.2.

Hence we'll add 3 new NICs to the CRC machine in order to have connectivity to those networks.

#### Adding NICs to CRC

To make it easier we'll stop the VM rather than hotplug the interfaces.

```
sudo virsh destroy crc
```
Once it's stopped (can be checked by running ```sudo virsh list --all``` and check State)
```
sudo virsh edit crc
```

And look to the last interface that the VM has. Should look like this:
```
    <interface type='network'>
      <mac address='52:54:00:d7:22:04'/>
      <source network='default'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x06' slot='0x00' function='0x0'/>
    </interface>
```

After this block, we need to add the 3 new interfaces by adding:  
```
    <interface type='network'>
      <mac address='52:54:00:d8:23:03'/>
      <source network='data'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x07' slot='0x00' function='0x0'/>
    </interface>
    <interface type='network'>
      <mac address='52:54:00:d8:23:04'/>
      <source network='management'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x08' slot='0x00' function='0x0'/>
    </interface>
    <interface type='network'>
      <mac address='52:54:00:d8:23:05'/>
      <source network='external'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x09' slot='0x00' function='0x0'/>
    </interface>
```
> _The source network will match the 3 different networks explained earlier._

Once VM is modified we can start it again:  
```
sudo virsh start crc
```

To ensure that NICs were added correctly and to configure them we'll need to connect to the CRC VM.
```
ssh -i /home/ospng/.crc/machines/crc/id_ecdsa core@192.168.130.11
```
> More info on how to connect to the CRC VM [here](https://www.itsfullofstars.de/2021/07/ssh-to-red-hat-crc-vm/)

Once inside the VM we can list the physical NICs by doing:  
```
ip -br -c a s | grep enp
```
> ip -br (brief) -c (color) a (address) s (show)

The output should look like this:  
```
[core@crc-74q6p-master-0 ~]$ ip -br -c a s | grep enp
enp2s0           UP             192.168.130.11/24 fe80::9a25:ab6c:bf33:6807/64
enp6s0           UP             192.168.122.10/24 fe80::e133:c0cc:bcd:9778/64
enp7s0           UP             fe80::56df:6ad1:9ab8:41bd/64
enp8s0           UP             172.16.0.79/24 fe80::2a9d:f45a:413c:111f/64
enp9s0           UP             10.0.0.80/24 2620:52:0:13b8::fe:1e/128 2620:52:0:13b8:7bf9:5121:a0ce:254e/64 fe80::da85:6c62:b522:fa03/64
```

As stated before, enp8s0 and enp9s0 have IP since networks management and external has DHCP, but data don't, so we'll add the IP manually by doing:  
```
sudo ip a add 192.168.24.10/24 dev enp7s0
```
> Make sure that that IP is not used by anyone.   
> Also, the IP on enp7s0 is not persistent, so it'll be removed if the VM is rebooted.

Now the CRC VM is connected to the same networks as our Wallaby environment.   
But we can't reach the internal_api endpoints on Wallaby, since they use vlans on the data network, and the crc machine has no vlan configuration yet. In order to accomplish that we'll need to modify some scripts. 

### Modify Openstack deployment to adapt it to the new NICs

#### NNCP

During the installation of the "podified openstack" one of the stages is to configure all the interfaces and vlans from the CRC VM. This stage will be done by the target **nncp**, which will be triger by **make openstack**.

The ```make nncp``` basically will run the gen-nncp.sh and will consume the CR created by it.

In order to incorporate the newly added NICs we will modify the gen-nncp.sh as well as the Makefile.

##### Makefile

The modifications into the Makefile will be short and simple. We'll add 3 new variables that will contain the name of the new NICs.

We'll add it under the NNCP section (search for "# NNCP")

```
NNCP_INTERFACE_DATA     ?= enp7s0
NNCP_INTERFACE_MGMNT    ?= enp8s0
NNCP_INTERFACE_EXT      ?= enp9s0
```

Once this variables are setted, we'll need to use them on the nncp target, will be done by adding the following code under: export INTERFACE=${NNCP_INTERFACE}:

```
nncp: export INTERFACE_DATA=${NNCP_INTERFACE_DATA}
nncp: export INTERFACE_MANAGEMENT=${NNCP_INTERFACE_MGMNT}
nncp: export INTERFACE_EXTERNAL=${NNCP_INTERFACE_EXT}
```

And with that all the modifications on the Makefile are done (for this stage).

##### gen-nncp.sh

We need to modify this script in order to use the newly created variables as well as to fix the IPs and vlans used, since the defaults are different from the ones used on the wallaby deployment, and since we're adopting the OSP 17.1 wallaby deployment to the Podified CP, the podified CP needs to use the same layout.

For debugging purposes we'll add the checks and echo commands for the new variables.
```
if [ -z "${INTERFACE_DATA}" ]; then
    echo "Please set INTERFACE_DATA"; exit 1
fi

if [ -z "${INTERFACE_MANAGEMENT}" ]; then
    echo "Please set INTERFACE_MANAGEMENT"; exit 1
fi

if [ -z "${INTERFACE_EXTERNAL}" ]; then
    echo "Please set INTERFACE_EXTERNAL"; exit 1
fi
```

And

```
echo INTERFACE DATA ${INTERFACE_DATA}
echo INTERFACE MANAGEMENT ${INTERFACE_MANAGEMENT}
echo INTERFACE EXTERNAL ${INTERFACE_EXTERNAL}
```

Once the debugging modifications are done, we'll go through every interface created and modify what's needed.

The first one will be _internalapi vlan interface_:

- We need to change the IP used, since it's using 172.17.0.x and wallaby uses 172.17.1.x
- We need to change the name, as it's using $INTERFACE but we need to use $INTERFACE_DATA
- We need to change the vlan.base-iface, that is using again $INTERFACE instead of $INTERFACE_DATA

The second one will be the next one, _storage vlan interface_:

- We need to change the IP used, from 172.18.0.x to 172.17.3.x
- We need to change the name, as it's using $INTERFACE but we need to use \$INTERFACE_MANAGEMENT. Also we need to modify the vlan tag, instead of using .21 we need to use .30, so the final result will be: "name: ${INTERFACE_MANAGEMENT}.30"
- We need to change the vlan.base-iface, that is using again $INTERFACE instead of $INTERFACE_MANAGEMENT
- Finally we need to change the vlan.id, from 21 to 30

Finally the next one, the _tenant vlan interface_:

- We need to change the IP used, since it's using 172.19.0.x and wallaby uses 172.17.2.x
- - We need to change the name, as it's using $INTERFACE but we need to use \$INTERFACE_MANAGEMENT. Also we need to modify the vlan tag, instead of using .21 we need to use .50. So the final result will be: "name: ${INTERFACE_MANAGEMENT}.50"
- We need to change the vlan.base-iface, that is using again $INTERFACE instead of $INTERFACE_MANAGEMENT
- Finally we need to change the vlan.id, from 21 to 50

Once all the vlans are configured, we'll need to add a new section, to configure our INTERFACE_DATA, since the
data network doesn't have a DHCP, the IP added by hand keeps disapearing, maybe with this will be persistend (at least during VM lifetime).

The section added should be:  
```
    - description: Configuring ${INTERFACE_DATA}
      ipv4:
        address:
        - ip: 192.168.24.10
          prefix-length: 24
        enabled: true
        dhcp: false
      ipv6:
        enabled: false
      mtu: ${INTERFACE_MTU}
      name: ${INTERFACE_DATA}
      state: up
      type: ethernet
```

#### NETTAT

Once the script is modified, CRC VM should be configured correctly once we run the make command (not yet), but we're missing on modifying the podified networks that they will be created (they need to be the same as the wallaby environment).
  
In order to acomplish that, we'll need to modify the target **netattach**. This target will create the NAD (Network Attachment Definition) CR that will define the networks used on the podified control plane.

In this step we'll also need to modify the Makefile and **scripts/gen-nettat.sh** with the correct IP and range for each network as well as the interfaces.

##### Makefile

As we did on the nncp stage, we'll add the interfaces on the netattach stage by adding this code into the netattach section:  
```
netattach: export INTERFACE_DATA=${NNCP_INTERFACE_DATA}
netattach: export INTERFACE_MANAGEMENT=${NNCP_INTERFACE_MGMNT}
netattach: export INTERFACE_EXT=${NNCP_INTERFACE_EXT}
```

##### gen-nettat.sh

The networks definitions that we're about to change will determine the free IP range that can be assigned to new pods, on the wallaby also exist an IP range to allocate new IPs, it would be best if those ranges doesn't overlap. In order to see which range Wallaby is using, connect to the undercloud (```sudo ssh stack@undercloud-0```) and check file: ```~/virt/network/network_data_v2.yaml```.

- ctlplane: This network shouldn't be used on our specific case, so we'll leave at it is.
- internalapi: 
    - We'll change the network used from 172.17.0. to 172.17.1 (on range, range_start, and range_end).
    - For the range, instead of using [30, 70] we'll be using [150, 190] (Wallaby uses [10, 149]). 
    - Finally change $INTERFACE for $INTERFACE_DATA
- storage:
    - We'll change the network used from 172.18.0. to 172.17.3 (on range, range_start, and range_end).
    - For the range, instead of using [30, 70] we'll be using [150, 190] (Wallaby uses [10, 149]). 
    - Change the vlan .21 for .30 (On master option)
    - Finally change $INTERFACE for $INTERFACE_MANAGEMENT
- tenant:
    - We'll change the network used from 172.19.0. to 172.17.2 (on range, range_start, and range_end).
    - For the range, instead of using [30, 70] we'll be using [150, 190] (Wallaby uses [10, 149]). 
    - Change the vlan .22 for .50 (On master option)
    - Finally change $INTERFACE for $INTERFACE_MANAGEMENT

#### METALLB

Metallb will be the pod in charge of assigning those IPs that will be attached to a service, hence will be the same IP regardless of the restarts of the pods. So we'll need to modify, again, the IP network, range and interface used.

As well as the last section, we need to use an IP range that it's not use by anyone (nor netatt, nor wallaby).

##### Makefile

As we did on the nncp and netatt stage, we'll add the interfaces on the metallb stage by adding this code into the metallb section:  
```
metallb: export INTERFACE_DATA=${NNCP_INTERFACE_DATA}
metallb: export INTERFACE_MANAGEMENT=${NNCP_INTERFACE_MGMNT}
metallb: export INTERFACE_EXT=${NNCP_INTERFACE_EXT}
```

##### gen-olm-metallb.sh

We need to adapt the networks and the range, as we did on the last two sections. So we'll be doing the modificatins to the ```${DEPLOY_DIR}/ipaddresspools.yaml``` part.

- internalapi:
    - Modify range to 172.17.1.190-172.17.1.200
- storage:
    - Modify range to 172.17.3.190-172.17.3.200
- tenant:
    - Modify range to 172.17.2.190-172.17.2.200

Once the ranges are modified, we need to modify the ```${DEPLOY_DIR}/l2advertisement.yaml``` part, to use the interfaces and the vlan correctly.

- Internalapi:
    - Modify ${INTERFACE} for ${INTERFACE_DATA}
- storage:
    - Modify ${INTERFACE} for ${INTERFACE_MANAGEMENT}
    - Modify tag from .21 to .30
- tenant:
    - Modify ${INTERFACE} for ${INTERFACE_MANAGEMENT}
    - Modify tag from .22 to .50

#### METALLB_CONFIG

Finally, the last target from openstack_prep (which configures de underlying network) is metallb_config, which calls again the gen-olm-metallb.sh. Since we have already modify it we only need to add the newly add variables (INTERFACE_DATA, etc) to the makefile.

##### Makefile

As always, add this code under the metallb_config section:
```
metallb_config: export INTERFACE_DATA=${NNCP_INTERFACE_DATA}
metallb_config: export INTERFACE_MANAGEMENT=${NNCP_INTERFACE_MGMNT}
metallb_config: export INTERFACE_EXT=${NNCP_INTERFACE_EXT}
```

### Openstack deployment

After all the modifications are done, we go to the install_yamls directory, and we make the targets crc_storage, input and openstack:  
```
cd ~/install_yamls
make crc_storage
make input
make openstack
```