# Getting familiar with the Installation

In this module, we'll get familiar with the cluster and the install of the Windows Node. If you haven't done already, login to the bastion node provided by the RHPDS email.

```shell
$ ssh chernand-redhat.com@bastion.lax-e35b.sandbox886.opentlc.com
```

To start we are going to Install Python 3, Pip, yq and helm. We need these in order to install yq. We will be using this throughout the module.

```shell
$ sudo yum install python3
sudo yum install python3-pip
sudo pip3 install yq
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
```

Next we will be cloning a repo from github we will be using some yaml files from:

```shell
$ git clone --single-branch --branch dev https://github.com/RedHatWorkshops/windows-containers-quickstart.git
```

Before proceeding, make sure you’re admin, you will find your login details on your workshop deployment:

```shell
$ oc login -u admin -p {{ ADMIN_PASSWORD }}
```

The first requisite is that you must be running OpenShift version 4.6 or newer. This cluster should have been installed at a supported version.

```shell
$ oc version
```

The next requisite is that the cluster must be installed with OVNKubernetes as the SDN for OpenShift. This can only be done at install time in the install-config.yaml file. This file is stored on the cluster after install. Take a look at the setting.

```shell
$ oc extract cm/cluster-config-v1 -n kube-system --to=- | awk '/networkType:/{print $2}'
```

This should output OVNKubernetes as the network type.

The next requisite is the cluster must be set up with overlay hybrid networking. This is another step that can only be done at install time. You can verify that the configuration has been done by running the following:

```shell
$ oc get network.operator cluster -o yaml | awk '/ovnKubernetesConfig:/{p=1} p&&/^    hybridClusterNetwork:/{print; p=0} p'
```

The output should look like this. As you can see, the hybridOverlayConfig was set up. This is the overlay network setup on the Windows Node.

```shell
    ovnKubernetesConfig:
      egressIPConfig: {}
      gatewayConfig:
        routingViaHost: false
      genevePort: 6081
      hybridOverlayConfig:
        hybridClusterNetwork:
        - cidr: 10.132.0.0/14
          hostPrefix: 23
      mtu: 8901
      policyAuditConfig:
        destination: "null"
        maxFileSize: 50
        rateLimit: 20
        syslogFacility: local0
    type: OVNKubernetes
```

To summarize, in order to use Windows Containers on OpenShift. You will need the following:

- OpenShift version 4.6 or newer.

- OVNKubernetes as the SDN.

- Additionally set up hybrid overlay networking.

Note, that all of this is done at install time. There’s, currently, no way to configure a cluster for Windows Containers post install.

## Installing the WMCO

You should see the WMCO pod running, we have already set that operator up for you:

```shell
$ oc get pods -n openshift-windows-machine-config-operator
```

The output should look something like this.

```shell
NAME                                               READY   STATUS    RESTARTS   AGE
windows-machine-config-operator-7ddc9f7d9b-vx4vx   1/1     Running   0          43m
```

Once the operator is up and running. You are ready to install a Windows Node.

## Installing a Windows Node.

In order for the WMCO to setup the Windows Node, it will need an ssh key to the cloud provider. The cloud provider will then mint a new keypair based on the private key provided. The WMCO will then use this key to login to the Windows Node and set it up as an OpenShift Node.

Generate an ssh key for the WMCO to use:

```shell
$ ssh-keygen -t rsa -f ${HOME}/.ssh/winkey -q -N ''
```

Once you’ve generated the key, add it as a secret to the openshift-windows-machine-config-operator namespace.

```shell
$ oc create secret generic cloud-private-key --from-file=private-key.pem=${HOME}/.ssh/winkey -n openshift-windows-machine-config-operator
```

This secret is used by the WMCO Operator to setup the Windows Node. Verify that it was created before you proceed.

```shell
$ oc get secret -n openshift-windows-machine-config-operator cloud-private-key
```

Once the WMCO Operator is up and running, and the ssh key loaded into the cluster as a secret, you can now deploy a Windows Node. How do you build a Windows Node? The same way you create OpenShift Linux nodes, with the MachineAPI

First, we will be creating a MachineSet for Windows Nodes. We will then explore important sections of the YAML.

```shell
$ windows-containers-quickstart/support/generate-windows-ms.sh
```

If you get a permissions denied error you may need to change permissions using this command

```shell
chmod +x windows-containers-quickstart/support/generate-windows-ms.sh
```

This should create the windows-ms.yaml file in your home directory.

```shell
$ ls -l ~/windows-ms.yaml
```

The Windows MachineSet is labeled with an Operating System ID of Windows. The following command will show the label of machine.openshift.io/os-id: Windows for the MachineSet.

```shell
$ cat ~/windows-ms.yaml | python3 -c 'import sys, yaml, json; json.dump(yaml.safe_load(sys.stdin), sys.stdout, indent=4)' | jq '.metadata.labels'
```

All the Windows Machines will have the worker label. The Windows Node will be treated like any other node in the cluster.

```shell
$ cat ~/windows-ms.yaml | python3 -c 'import sys, yaml, json; json.dump(yaml.safe_load(sys.stdin)["spec"]["template"]["spec"]["metadata"]["labels"], sys.stdout, indent=4)' | jq
```

The AMI ID is of a Windows Server 2019 AMI.

```shell
$ cat ~/windows-ms.yaml | python3 -c 'import sys, yaml, json; json.dump(yaml.safe_load(sys.stdin), sys.stdout)' | jq -r '.spec.template.spec.providerSpec.value.ami.id'
```

One last thing to note, is the user data secret.

```shell
$ cat ~/windows-ms.yaml | python3 -c 'import sys, yaml, json; data = yaml.safe_load(sys.stdin); print(data["spec"]["template"]["spec"]["providerSpec"]["value"]["userDataSecret"]["name"])'
```

This secret is generated by the WMCO when it was installed.

```shell
$ oc get secret windows-user-data -n openshift-machine-api
```

Apply the YAML to create the Windows MachineSet on the cluster.

```shell
$ oc apply -f ~/windows-ms.yaml
```

You can now see the status of the MachineSet.

```shell
$ oc get machinesets  -n openshift-machine-api -l machine.openshift.io/os-id=Windows
```

This should show the following output.

```shell
NAME                                       DESIRED   CURRENT   READY   AVAILABLE   AGE
cluster1-wrkjp-windows-worker-us-east-1a   1         1                             9s
```

The MachineSet has the replica set to 1. The MachineAPI will see that desired state and, in turn, create a Windows Machine. This machine will eventually turn into a node. See the status of the machine with the following command.

```shell
oc get machines  -n openshift-machine-api -l machine.openshift.io/os-id=Windows
```

Once the Machine is up and running, the WMCO will configure it. You can follow that status by looking at the WMCO pod log.

```shell
oc logs -l name=windows-machine-config-operator -n openshift-windows-machine-config-operator   -f
```

You can exit by pressing kbd:[Ctrl+C].

This Machine will create a Windows Node and the WMCO will add it to the cluster. You can see the node with the following command.

```shell
oc get nodes -l kubernetes.io/os=windows
```

Note: It’ll take up to 15 mintues to see the Windows Node appear. It’s recommneded to run a watch on oc get nodes -l kubernetes.io/os=windows so you can see when the node appears. Now will be a good time to take a break.

The output should look something like this.

```shell
NAME                          STATUS   ROLES    AGE   VERSION
ip-10-0-140-10.ec2.internal   Ready    worker   22m   v1.20.0-1081+d0b1ad449a08b3
```

## Managing a Windows Node

Now that the Windows Node is up and running, you will be able to manage it like you would a Linux node. You will be able to scale and delete nodes using the MachineAPI.

Currently, you have one Windows node.

```shell
oc get nodes -l kubernetes.io/os=windows
```

In order to add another node, you will just scale the corespoinding MachineSet. Currently, you should have one

```shell
oc get machineset -l machine.openshift.io/os-id=Windows -n openshift-machine-api
```

You should have the below output. It shows that you have one Windows Machine managed by this MachineSet.

```shell
NAME                                       DESIRED   CURRENT   READY   AVAILABLE   AGE
cluster1-zzv5j-windows-worker-us-east-1a   1         1         1       1           138m
```

To add another Windows Node, scale the Windows MachineSet to two replicas. This will create a new Windows Machine, and then the WMCO will add it as an OpenShift Node.

```shell
oc scale machineset -l machine.openshift.io/os-id=Windows -n openshift-machine-api --replicas=2
```

Note: Just like when you created the inital Windows Node, this can take upwards of 15 minutes. This can be another good time to take a small break.

After some time, another Windows Node will have joined the cluster.

```shell
oc get nodes -l kubernetes.io/os=windows
```

Here’s an example output.

```shell
NAME                           STATUS   ROLES    AGE     VERSION
ip-10-0-139-232.ec2.internal   Ready    worker   15m     v1.20.0-1081+d0b1ad449a08b3
ip-10-0-143-146.ec2.internal   Ready    worker   3h18m   v1.20.0-1081+d0b1ad449a08b3
```

You can see how easy it is to manage a Windows Machine with the MachineAPI on OpenShift. It is managed by the same system as your Linux Nodes. You can even attach the Windows MachineSet Autoscaler as well

Remove this node by scaling the Windows MachineSet back down to 1.

```shell
oc scale machineset -l machine.openshift.io/os-id=Windows -n openshift-machine-api --replicas=1
```

Warning: Please scale your Windows MachineSet to 1 before starting the next exercise.

After some time, you should be back at 1 Windows node.

```shell
oc get nodes -l kubernetes.io/os=windows
```

## Exploring The Windows Node

Now that you’ve learned how to manage a Windows Node, we will explore how this node is set up. You can access this Windows node via the same mechanism as the WMCO, via SSH.

Since this cluster was installed in the cloud, the Windows Node isn’t exposed to the public internet. So we will need to deploy an ssh bastion Pod.

The ssh bastion pod can be deployed using the Deployment YAML provided to you in this lab.

```shell
oc apply -n openshift-windows-machine-config-operator -f windows-containers-quickstart/support/win-node-ssh.yaml
```

You can wait for the rollout of this ssh bastion pod.

```shell
oc rollout status deploy/winc-ssh -n openshift-windows-machine-config-operator
```

Once rolled out, you should have the ssh bastion pod running.

```shell
oc get pods -n openshift-windows-machine-config-operator -l app=winc-ssh
```

The ssh bastion pod mounts the ssh key needed to login to the Windows Node.

```shell
yq e '.spec.template.spec.volumes' windows-containers-quickstart/support/win-node-ssh.yaml
```

In order to be able to ssh into this node you will need the hostname. Get this hostname with the following command and make note of it.

```shell
oc get nodes -l kubernetes.io/os=windows
```

Now open a bash session into the ssh bastion pod using the oc exec command.

```shell
oc exec -it deploy/winc-ssh -n openshift-windows-machine-config-operator -- bash
```

Use the provided sshcmd.sh command built into the pod to login to the Windows Node. Here is an example:

```shell
bash-4.4$ sshcmd.sh ip-10-0-140-10.ec2.internal
```

This should drop you into a PowerShell session. It should look something like this.

```shell
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Users\Administrator>
```

Once on the Windows Node, you can see the containerd, hybrid-overlay-node, kubelet, kube-proxy, windows_exporter and windows-instance-config-daemon processes are running.


```shell
Get-Process | ?{ $_.ProcessName -match "daemon|exporter|kube|overlay|containerd" }
```


You should see the following output.

```shell
Handles  NPM(K)    PM(K)      WS(K)     CPU(s)     Id  SI ProcessName
-------  ------    -----      -----     ------     --  -- -----------
    171      14    32780      30652       3.63   5484   0 containerd
    252      17    33712      37144       1.61    292   0 hybrid-overlay-node
    605      31    60244      81452      47.27    756   0 kubelet
    274      21    38404      42992       5.53   5256   0 kube-proxy
    472      23    41572      38320      16.23   1140   0 windows_exporter
    205      16    31880      32128       1.55    592   0 windows-instance-config-daemon
```

These are the main components needed to run a Windows Node. Remember that this node is managed the same way as a Linux node, Via the MachineAPI; so you won’t have to do much with this Windows Node.

You can now exit out of the PowerShell session.

```shell
exit
```

You can also exit out of the bash container session as well.

```shell
exit
```




## Running a Windows Container Workload

Before you deploy a sample Windows Container workload, let’s explore how the container gets scheduled on the Windows node.

If you run an oc describe on the Windows Node, you’ll see it has a taint.

```shell
oc describe nodes -l kubernetes.io/os=windows | grep Taint
```

You should see the following output.

```shell
Taints:             os=Windows:NoSchedule
```

Every Windows Node will come with this taint by default. This taint will "repel" all workloads that don’t tolerate this taint. It is a part of the WMCO’s job to ensure that all Windows Nodes have this taint.

In this lab, there is a sample workload saved under ~/support/winc-sample-workload.yaml. Let’s explore this file a bit before we apply it.

```shell
yq e '.items[2].spec.template.spec.tolerations' windows-containers-quickstart/support/winc-sample-workload.yaml
```

The output should look something like this.

```shell
- key: "os"
  value: "Windows"
  Effect: "NoSchedule"
```

This sample workload has the toleration in place to be able to run on the Windows Node. However, that’s not enough. A nodeSelector will need to be present as well.

```shell
yq e '.items[2].spec.template.spec.nodeSelector' windows-containers-quickstart/support/winc-sample-workload.yaml
```

The output should look something like this.

```shell
kubernetes.io/os: windows
```

So here, the nodeSelector will place this container on the Windows Node. Furthermore, the appropriate toleration is in place so the Windows Node won’t repel the container.

One last thing to look at. Take a look at the container that is being deployed.

```shell
yq e '.items[2].spec.template.spec.containers[0].image' windows-containers-quickstart/support/winc-sample-workload.yaml
```

Apply this YAML file to deploy the sample workload.

```shell
oc apply -f windows-containers-quickstart/support/winc-sample-workload.yaml
```

Wait for the deployment to finish rolling out. This can take 5-10 minutes as Windows images are large in size.

```shell
oc rollout status deploy/win-webserver -n winc-sample
```

If you check the pod, you can see that it’s running on the Windows Node. Look at the wide output of the Pod and select the Windows Node to verify.

```shell
oc get pods -n winc-sample  -o wide
oc get nodes -l kubernetes.io/os=windows
```

Make a note of the Windows Node name, we will log into the node using the bastion ssh container.

```shell
oc exec -it deploy/winc-ssh -n openshift-windows-machine-config-operator -- bash
```

Now log into the Windows Node using the hostname. Example:

```shell
bash-4.4$ sshcmd.sh ip-10-0-140-10.ec2.internal
```

To view Windows containers running on the node, you need to install the crictl tool to interact with the containerd runtime.

```shell
$ProgressPreference = "SilentlyContinue"; wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.27.0/crictl-v1.27.0-windows-amd64.tar.gz -o crictl-v1.27.0-windows-amd64.tar.gz; tar -xvf crictl-v1.27.0-windows-amd64.tar.gz -C C:\Windows\
```

Now lets configure crictl.

```shell
crictl config --set runtime-endpoint="npipe:\\\\.\\pipe\\containerd-containerd"
```

Here, you can see the Windows container running on the node.

```shell
crictl ps
```

Here you’ll see the Container running. Here is an example output.

```shell
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID              POD
ac18c2aa692cf       66a1a48cbc112       2 minutes ago       Running             windowswebserver    0                   a2f1b580c659c       win-webserver-7b76494c5-s9m2q
```

You can also see the images downloaded on the host.

```shell
crictl images
```

You should see the following output.

```shell
IMAGE                                    TAG                 IMAGE ID            SIZE
mcr.microsoft.com/oss/kubernetes/pause   3.6                 9adbbe02501b1       104MB
mcr.microsoft.com/windows/servercore     ltsc2019            66a1a48cbc112       2.02GB
```

Go ahead an logout of the Windows Node

```shell
exit
```

You can also exit out of the bash container session as well.

```shell
exit
```

You can interact with the Windows Container workload as you would any other pod. For instance you can remote shell into the container itself by calling the Powershell command.

```shell
oc -n winc-sample exec -it $(oc get pods -l app=win-webserver -n winc-sample -o name ) -- powershell
```

This should put you in a Powershell session in the Windows Container. It should look something like this

```shell
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\>
```

Here, you can query for the running HTTP process.


Note: You may have to press ENTER to execute the following commands while in the Windows Container for them to run.

```shell
Get-WmiObject Win32_Process -Filter "name = 'powershell.exe'" | Select-Object CommandLine | Select-String -Pattern http
```

Go ahead an logout of the Windows Container.

```shell
exit
```

You can interact with the Windows Container Deployment the same as you would for a Linux one. Scale the Deployment of the Windows Container:

```shell
oc scale deploy/win-webserver -n winc-sample --replicas=2
```

You should now have two Pods running.

```shell
oc get pods -n winc-sample
```

## Running a Mixed Linux/Windows Container Workload.

With Windows Containers support for OpenShift; You also have the ability to run application stacks of mixed workloads. This gives you the ability to run an application stack consisting of both Linx and Windows Containers.

In this section, we will show how you can run Windows workloads that work together with Linux workloads.

You will be deploying a sample application stack that delivers an eCommerce site, The NetCandy Store. This application is built using Windows Containers working together with Linux Containers.

IMAGE

This application consists of:

- Windows Container running a .NET v4 frontend, which is consuming a backend service.

- Linux Container running a .NET Core backend service, which is using a database.

- Linux Container running a MSSql database.

We will be using a helm chart to deploy the sample application. In order to successfully deploy the application stack, make sure you’re kubeadmin.

Next add the Red Hat Developer Demos Helm repository.


```shell
helm repo add redhat-demos https://redhat-developer-demos.github.io/helm-repo
helm repo update
```

Create the namespace for netcandystore.

```shell
oc create namespace netcandystore
```

Next we will use this command below to create a Kubernetes resource with specific security restrictions and context constraints within the OpenShift cluster.

```shell
oc create -f windows-containers-quickstart/support/restrictedfsgroupscc.yaml
```

Next, we’ll allow a specific group of service accounts (in this case, those related to Microsoft SQL Server) to follow the strict security rules defined by the "restrictedfsgroup" Security Context Constraints in the OpenShift system.

```shell
oc adm policy add-scc-to-group restrictedfsgroup system:serviceaccounts:mssql
```

With the two variables exported, and the helm repo added, you can install the application stack using the helm cli.

```shell
helm install ncs --namespace netcandystore \
--timeout=1200s \
redhat-demos/netcandystore
```

Note: Note that the --timeout=1200s is needed because the default timeout for helm is 5 minutes and, in most cases, the Windows container image will take longer than that to download.

This will look like it’s "hanging" or "stuck". It’s not! What’s happening is that the image is getting pulled into the Windows node. As stated before, Windows containers can be very large, so it might take some time.

After some time, you should see something like the following return.

```shell
NAME: ncs
LAST DEPLOYED: Sun Mar 28 00:16:05 2021
NAMESPACE: netcandystore
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
1. Get the application URL by running these commands:
oc get route netcandystore -n netcandystore -o jsonpath='{.spec.host}{"\n"}'

2. NOTE: The Windows container deployed only supports the following OS:

Windows Version:
=============
Windows Server 2019 Release 1809

Build Version:
=============

Major  Minor  Build  Revision
-----  -----  -----  --------
10     0      17763  0
```

Verify that the helm chart was installed successfully.

```shell
helm ls -n netcandystore
```

The output should look something like this.

```shell
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
ncs     netcandystore   1               2021-03-31 19:54:50.576808462 +0000 UTC deployed        netcandystore-1.0.3     3.1
```

There should be 3 pods running for this application. One for the frondend called netcandystore, one for the categories service called getcategories and a DB called mysql.

```shell
oc get pods -n netcandystore
```
------
PLEASE DO NOT DO THESE NEXT STEPS IF YOUR NETCANDYSTORE PODS ARE RUNNING:
If there the NetCandyStore pod is having any issues you can try to delete and redeploy

```shell
helm uninstall ncs --namespace netcandystore
```

```shell
helm install ncs --namespace netcandystore --timeout=1200s redhat-demos/netcandystore
```
-----

Looking at the frontend application, you can list where the pod is running. Comparing it to the nodes output, you can see it’s running on a Windows Node.

```shell
oc get pods -n netcandystore -l app=netcandystore -o wide
oc get nodes -l kubernetes.io/os=windows
```

Now, looking at the backend, you can see it’s running on a Linux node.

```shell
oc get pods -n netcandystore -l app=getcategories -o wide
oc get nodes -l kubernetes.io/os=linux
```

Now, looking at the backend, you can see it’s running on a Linux node.

```shell
oc get pods -n netcandystore -l app=getcategories -o wide
oc get nodes -l kubernetes.io/os=linux
```

The MSSQL Database is also running on the Linux node.

```shell
oc get pods -n netcandystore -l deploymentconfig=mssql -o wide
```

You can extract the URL from the cluster

```shell
$ oc get route netcandystore -n netcandystore -o jsonpath='{.spec.host}{"\n"}'
```

The frontpage should look like this, feel free to play around with the application!

Picture

## Conclusion

In this lab you worked with Windows Containers on OpenShift Container
Platfrom. You saw how the cluster was prepared to support Windows
Containers. You also learned about the Windows Machine Config Operator and
how it's used to provision a Windows Node.

You also learned about how to manage Windows Nodes using the MachineAPi
and how to manage Windows Container workloads using the same tools as
Linux Nodes.

Finally, you learned how you can used mixed workloads made up of Linux
and Windows containers.











