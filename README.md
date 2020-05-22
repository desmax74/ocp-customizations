# ocp-customizations
This repository contains various scripts and YAMLs to perform specific customizations to an OCP4.x cluster. They all in some form of another require the use of a [custom Red Hat CoreOS installer image](https://github.com/RHsyseng/coreos-installer-custom-partitions).

## Building the image

The custom installer image can be built with the following steps:
```
git clone https://github.com/RHsyseng/coreos-installer-custom-partitions -b legacy
cd coreos-installer
curl https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rhcos/4.4/latest/rhcos-4.4.3-x86_64-installer-initramfs.x86_64.img -o rhcosinstall-initramfs.img # or the latest currently available
./combine.sh
```

The resulting file will be called `rhcos-install-new.img` and can be used as the installer image for PXE booting OCP nodes.

## Disabling additional NICs

Assuming the use of the custom installer, and that an additional kernel argument of 'macAddressList' was passed at PXE boot time:
```
export SCRIPT_BASE64=$(base64 -w 0 disable-nics.sh)
envsubst '${SCRIPT_BASE64}' < disable-nics.ign.tmpl > disable-nics.ign
```

The `disable-nics.ign` file can then be merged with ignition files generated by the `openshift-install` command:
```
assets_dir=/path/to/dir/with/install-config.yml
ignition_extra=disable-nics.ign
openshift-install --dir ${assets_dir} create ignition-configs
mv ${assets_dir}/bootstrap.ign{,.orig}
jq -s '.[0] * .[1]' ${ignition_extra} ${assets_dir}/bootstrap.ign.orig | tee ${assets_dir}/bootstrap.ign
mv ${assets_dir}/master.ign{,.orig}
jq -s '.[0] * .[1]' ${ignition_extra} ${assets_dir}/master.ign.orig | tee ${assets_dir}/master.ign
mv ${assets_dir}/worker.ign{,.orig}
jq -s '.[0] * .[1]' ${ignition_extra} ${assets_dir}/worker.ign.orig | tee ${assets_dir}/worker.ign
```

This will disable any NICs on the system that are connected and has a MAC address that does not match the first one in the `macAddressList` provided list.

The newly generated bootstrap, master and worker ignition files can now be used for deploy the OCP4 cluster.

## Configure an OVS Bond + Bridge

Make any necessary modifications to `setup-ovs.sh` and `mco_ovs.yml.tmp` and run:
```
export SCRIPT_BASE64=$(base64 -w 0 setup-ovs.sh)
envsubst '${SCRIPT_BASE64}' < mco_ovs.yml.tmpl > mco_ovs.yml
```

Then apply the MachineConfig to the cluster:
```
for node in $(oc get nodes -l node-role.kubernetes.io/worker --no-headers=true -o name | awk -F/ '{print $2}'); do
  oc label node $node network.operator.openshift.io/external-openvswitch=true
done
oc apply -f mco_ovs.yml
```

## Storage

`mco_storage.yml` will mount an extra 5th partition in the specified location. Modify `mco_storage.yml` if necessary (e.g. to change the path) and apply it to the cluster:
```
oc apply -f mco_storage.yml
```

## Combined MCO

To save on reboots, all customizations can be combined in to one MachineConfig object.
```
export SCRIPT_BASE64=$(base64 -w 0 setup-ovs.sh)
envsubst '${SCRIPT_BASE64}' < mco_all.yml.tmpl > mco_all.yml
```

And apply it to the cluster:
```
oc apply -f moc_all.yml
```
