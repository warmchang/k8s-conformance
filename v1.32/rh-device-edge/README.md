# Red Hat Device Edge

## Create a multi-node MicroShift cluster

### Install two MicroShift hosts

1. Prepare two RHEL 9.4 hosts suitable for running MicroShift. The following configuration is recommended:
- RHEL 9.4
- 2 CPU cores
- 4GB of RAM
- 10 GB of storage
- An active MicroShift subscription on your Red Hat account
- A Logical Volume Manager (LVM) Volume Group (VG) with e.g. 2GB of unallocated space to allow for the addition of additional logical volumes (LVs) to accomodate persistent volumes (PVs). The volume group should use the default name `rhel`.

2. Enable the Red Hat build of Microshift repositories by running the following command on both hosts:

```
sudo subscription-manager config --rhsm.manage_repos=1
sudo subscription-manager repos --enable fast-datapath-for-rhel-9-$(uname -m)-rpms
sudo tee "/etc/yum.repos.d/rhocp-4.19-for-rhel-9-mirrorbeta-x86_64-rpms.repo" >/dev/null <<EOF
[rhocp-4.19-for-rhel-9-mirrorbeta-x86_64-rpms]
name=Beta rhocp-4.19 RPMs for RHEL 9
baseurl=https://mirror.openshift.com/pub/openshift-v4/x86_64/dependencies/rpms/4.19-el9-beta/
enabled=1
gpgcheck=0
skip_if_unavailable=0
EOF
sudo tee "/etc/yum.repos.d/microshift-preview.repo" >/dev/null <<EOF
[microshift-preview]
name=Beta microshift-4.19 RPMs for RHEL 9
baseurl=https://mirror.openshift.com/pub/openshift-v4/x86_64/microshift/ocp/4.19.0-rc.4/el9/os/
enabled=1
gpgcheck=0
skip_if_unavailable=0
EOF
```

3. Install the Red Hat build of MicroShift and firewalld by running the following command on both hosts:

```
sudo dnf install -y microshift firewalld
```

4. Download your installation pull secret from the [Red Hat Hybrid Cloud Console](https://console.redhat.com/openshift/install/pull-secret) to a temporary folder, for example, `$HOME/openshift-pull-secret`. This pull secret allows you to authenticate with the container registries that serve the container images used by Red Hat build of MicroShift. Then run the following command on both hosts:

```
sudo cp $HOME/openshift-pull-secret /etc/crio/openshift-pull-secret
```

5. Fixup the permissions of the `/etc/crio/openshift-pull-secret` file by running the following command on both hosts:

```
sudo chown root:root /etc/crio/openshift-pull-secret
sudo chmod 600 /etc/crio/openshift-pull-secret
```

6. If your hosts have firewalld enabled, run the following commands on both hosts:

```
sudo firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16
sudo firewall-cmd --permanent --zone=trusted --add-source=169.254.169.1
sudo firewall-cmd --reload
```

### Combine the two MicroShift hosts into a single cluster

1. Now that you have two MicroShift hosts, we'll combine them into a single cluster. The following commands should be run on a system with access to both of the MicroShift hosts prepared in the previous section.

2. Set environment variables that denote primary and secondary hostnames and IP addresses to be used in the subsequent commands by running the following commands:

```
PRI_HOST=microshift-pri
PRI_ADDR=192.168.124.119
SEC_HOST=microshift-sec
SEC_ADDR=192.168.124.19
```
3. Run the following commands to copy the configuration script to the primary host and run it remotely. The configuration script prints the list of the kubelet configuration files that need to be copied to the secondary host.

```
git clone https://github.com/openshift/microshift.git --branch release-4.19
cd ./microshift/
scp -o StrictHostKeyChecking=no ./scripts/multinode/configure-pri.sh redhat@${PRI_ADDR}:
ssh -t redhat@${PRI_ADDR} ./configure-pri.sh "${PRI_HOST}" "${PRI_ADDR}" "${SEC_HOST}" "${SEC_ADDR}"
```

4. Copy the kubelet configuration files from the primary to the secondary host by running the following command.

```
scp -3 -o StrictHostKeyChecking=no \
    redhat@${PRI_ADDR}:kubelet-${SEC_HOST}.{key,crt} \
    redhat@${PRI_ADDR}:kubeconfig-${PRI_HOST} \
    redhat@${PRI_ADDR}:lvmd-${PRI_HOST}.yaml \
    redhat@${SEC_ADDR}:
```

5. Run the following commands to copy the configuration script to the secondary host and run it remotely.

```
scp -o StrictHostKeyChecking=no ./scripts/multinode/configure-sec.sh redhat@${SEC_ADDR}:
ssh -t redhat@${SEC_ADDR} ./configure-sec.sh "${PRI_HOST}" "${PRI_ADDR}" "${SEC_HOST}" "${SEC_ADDR}"
```

6. Set the KUBECONFIG variable using the configuration file from the primary host by running the following commands.

```
export KUBECONFIG=$(mktemp /tmp/microshift-kubeconfig.XXXXXXXXXX)
scp redhat@${PRI_ADDR}:kubeconfig-${PRI_HOST} ${KUBECONFIG}
```

7. Verify that the cluster has two nodes in the Ready status and wait until all the pods are reporting that status is `Running` by running the following commands.

```
oc get nodes
watch oc get pods -A
```

## Run conformance tests

1. By default OpenShift security rules do not allow running with privileged access.
   Run the following commands to allow unprivileged users to run root level containers.

```
oc adm policy add-scc-to-group privileged system:authenticated system:serviceaccounts
oc adm policy add-scc-to-group anyuid system:authenticated system:serviceaccounts
```

2. Follow the [test instructions](https://github.com/cncf/k8s-conformance/blob/master/instructions.md#running)
   to run the conformance tests. You will need to add the `--dns-namespace=openshift-dns`
   and `--dns-pod-labels=dns.operator.openshift.io/daemonset-dns=default`
   options so `sonobuoy` can find the cluster DNS pods:

```
sonobuoy run --mode=certified-conformance --dns-namespace=openshift-dns --dns-pod-labels=dns.operator.openshift.io/daemonset-dns=default
```

3. Once conformance testing is completed, restore the default security rules by running the following commands:

```
oc adm policy remove-scc-from-group anyuid system:authenticated system:serviceaccounts
oc adm policy remove-scc-from-group privileged system:authenticated system:serviceaccounts
```
