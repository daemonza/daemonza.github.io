+++
date = "2017-01-15T10:09:57+01:00"
draft = true
title = "Kubernetes on AWS"

+++
In this post I will be explaining how I setup Kubernetes clusters on Amazon.

I use kops (https://github.com/kubernetes/kops) to setup and manage my Kubernetes clusters. For my day job, I create
multiple Kubernetes stacks often on Amazon and nothing, so far, comes close to how well kops works for me. I briefly
flirted with kube-aws  (https://github.com/coreos/kube-aws) from CoreOS, and while it's a good tool, Kops just works 
better for me. Also there is something to be said for using a tool developed and used by the same guys that develop Kubernetes.

There is also kubeadm (https://kubernetes.io/docs/getting-started-guides/kubeadm/), however it's currently in alpha. Have not
used it myself before, but it's defiantly something I am keeping a eye on.

#### Installing kops

I use macOS so from here all instructions will be based on the assumption it's being executed on macOS. It should 
however work on Linux or at least be easily adapted to work on Linux.

Install Go 1.7 and install kops from source. The pre-build releases of kops does not include a fix in the Go net lib
which seems to break things every now and again.

Install the following :

* Go 1.7 or better from https://golang.org/dl/ or use brew
* kops - v1.4.4

```
go get -d k8s.io/kops
cd ${GOPATH}/src/k8s.io/kops/
git checkout release
make
```

Test that kops works with

```
kops version
```

#### Setup

It's strickly not needed to install awscli, but it's extremely useful when working with
Kubernetes clusters on Amazon;

* aws-cli/1.10.22 (http://docs.aws.amazon.com/cli/latest/userguide/installing.html)

Configure your amazon credentials by running :

```
aws configure
```

Create a S3 state store bucket for kops. This bucket get's used by kops to store 
information and configuration of all your Kubernetes clusters you create. Anyone with
Amazon credentials to the S3 bucket and create/modify and delete the your Kubernetes clusters, which
makes it very confieniant to share with team members or if you move between computers, and not having to
move configuration, etc. with you.

Creating a state store bucket in the eu-west-1 region of amazon. 

```
aws s3api create-bucket --bucket my-state-store --region eu-west-1
export KOPS_STATE_STORE=s3://my-state-store
```

Decided on a place to save your cluster kubeconfig and ssh keys locally. I like creating a directory structure
that mimicks my my clusters. So for example if I am setting a development stack up for myself I would :

```
mkdir -p ~/stacks/dev.daemonza.io
touch ~/stacks/dev.daemonza.io/kubeconfig 
EXPORT KUBECONFIG="~/stacks/dev.daemonza.io/kubeconfig"
```

Change to this directory and create ssh keys. 

```
mkdir credentials
ssh-keygen -t rsa # save the keys in the credentials directory
```

#### Create cluster

First thing is to decided on the DNS setup for yout cluster. It's standard practice to use subdomains
to identify a cluster. E.g. dev.daemonza., qa.daemonza.io, prod.daemonza.io for the develop, QA and
production stacks. Following that convention let's create our cluster under dev.daemonza.io.
Seeing as we are deploying to AWS, let's use Amazon's Route53 DNS service to handle the DNS side for us.



```
kops create cluster --v=0 \
  --cloud=aws \
  --node-count 2 \
  --master-size=t2.medium \
  --master-zones=eu-west-1a \
  --zones eu-west-1a,eu-west-1b \
  --name=dev.daemonza.io \
  --node-size=m3.xlarge \
  --ssh-public-key=~/stacks/dev.daemonza.io/id_rsa.pub \
  --dns-zone dev.daemonza.io \
  2>&1 | tee /stacks/dev.daemonza.io/create_cluster.txt
```

A little explanation of the parameters passed to kops :

* `--cloud`
We want to install the cluster on Amazon's AWS. At the moment only aws and google cloud engine is
supported.

* `--node-count`
The number of worker nodes. This is the nodes running our applications.
 
* `--master-size`
 The EC2 instance type of the master node (manage Kubernetes)
Kubernetes recommends the following on master node sizing
```
For the master, for clusters of less than 5 nodes it will use an m3.medium, for 6-10 nodes it will use an m3.large;
for 11-100 nodes it will use an m3.xlarge.
For worker nodes, for clusters less than 50 nodes it will use a t2.micro, for clusters between 50 and 150 nodes it
will use a t2.small and for clusters with greater than 150 nodes it will use a t2.medium.
```

* `--master-zones`
The AWS availability zone(http://docs.aws.amazon.com/general/latest/gr/rande.html) where the master EC2
instance will be placed. 

* `--zones` 
The AWS availability zones where the worker nodes EC2 instances will be places.

* `--name` 
The name you are giving to your stack. I like to make this a fqdn. Something like data-processing.example.com
if the stack's main purpose is going to be data processing for example.com.

* `--node-size` 
The worker node EC2 instance type (https://aws.amazon.com/ec2/instance-types/)

* `--ssh-public-key` 
This is the ssh key you generated in the previous command.
 
* `--dns-zone` 
The fqdn of where your stack will be, I normally make this the same as the `name` parameter.

The output of the commmand I pipe to stdout, to monitor progress and activity while creating the stack and to a file
in case I missed something on stdout or want to reference something of the stack create in the future.
 
This command generated your Kubernetes cluster configuration. Now run update (see below) to 
create your cluster.

```
kops update cluster dev.daemonza.io --yes
```

Wait a while for your cluster to come up. You can look in the AWS console at the EC2 instances
being created. It takes roughly around 10 mins or so for your cluster to become availible. 

You can confirm the cluster is there by running
```
kubectl get pods
```
To get a list of all the nodes in your cluster (1 master, 2 worker nodes)

I usually deploy the Kubernetes Dashboard (https://github.com/kubernetes/dashboard) after the cluster is up. I use 
the kubectl command line tool probably 99% of the time, but it's nice to every now and again look
at your cluster using the UI.

```
kubectl create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
```

You can now run kubectl proxy and access your cluster at 
http://127.0.0.1:8001/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard/#/deployment?namespace=_all


#### List clusters

Kops makes it easy to manage multiple stacks. To get a list of all your current stacks 
run the following :
 
```
kops get clusters
```

Example output :

```                                                                                                                                              
NAME			CLOUD	ZONES
dev.daemonza.io	aws	eu-west-1a,eu-west-1b
testing.daemonza.io	aws	eu-west-1a,eu-west-1b
production.daemonza.io	aws	eu-west-1a,eu-west-1b
```

As you can see I have three stacks running. One being the stack (dev.daemonza.io) that we just created.
From here we can get various information about a cluster. For example to get a description of the
EC2 instances that make up a cluster run :

```
kops get instancegroups
```
This command will list the instances of the cluster to which your KUBECONFIG environment variable is 
pointing. Earlier in this tutorial we created a kubeconfig file and exported KUECONFIG to that. The kops
create / update command filled out the kubeconfig file for us, which we are using now.

Example output :

```
Using cluster from kubectl context: dev.daemonza.io

NAME			ROLE	MACHINETYPE	MIN	MAX	ZONES
master-eu-west-1a	Master	t2.medium	1	1	eu-west-1a
nodes			Node	m3.xlarge	2	2	eu-west-1a,eu-west-1b
```

#### Modify cluster

Kops makes it easy, and relative safe to modify a Kubernetes cluster. Continuing from our example above in the 
cluster listing. Let's modify our cluster to have three m3.xlarge worker nodes instead of just two, and 
change the root volume size.

Run

```
kops edit instancegroups nodes
```

This will open a configuration file in your default editor (set it with `export EDITOR=vim` if you want vim as default)

Example of how the configuration looks :

```
metadata:
  creationTimestamp: "2017-01-04T13:59:07Z"
  name: nodes
spec:
  associatePublicIp: true
  image: kope.io/k8s-1.4-debian-jessie-amd64-hvm-ebs-2016-10-21
  machineType: m3.xlarge
  maxSize: 2
  minSize: 2
  role: Node
  zones:
  - eu-west-1a
  - eu-west-1b                                                                                                                                                       13
```

Change the maxSize value to 3 and keep minSize on 2, this will set the autoscale group on amazon
to scale to three worker nodes.
And for the root volume add the following for a 100gig volume of type gp2 (for more on EC2 volume types
see http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSVolumeTypes.html)

```
rootVolumeSize: 100
rootVolumeType: gp2
```

With your end result looking like the following :

```
metadata:
  creationTimestamp: "2017-01-04T13:59:07Z"
  name: nodes
spec:
  associatePublicIp: true
  image: kope.io/k8s-1.4-debian-jessie-amd64-hvm-ebs-2016-10-21
  machineType: m3.xlarge
  maxSize: 2
  minSize: 2
  role: Node
  rootVolumeSize: 100
  rootVolumeType: gp2
  zones:
  - eu-west-1a
  - eu-west-1b 
```

Confirm the changes is correct by running

```
kops update cluster dev.daemonza.io
```

If your happy with the changes run the following :

```
kops update cluster dev.daemonza.io --yes
kops rolling-update cluster --yes
```

Sit back and wait for your cluster changes to take affect. 

#### Remove cluster

Now to remove the cluster that you just created run
```
kops delete cluster dev.daemonza.io
```

This will print a list of amazon resources that will be removed. 
Example :

```
TYPE		NAME				ID
dhcp-options	dev.daemonza.io	        	dopt-6627fc02
route-table 	dev.daemonza.io	    	    rtb-36fc1351
security-group	masters.dev.daemonza.io	    sg-e6e99a80
security-group	nodes.dev.daemonza.io	    sg-19e99a7f
subnet		    eu-west-1a.dev.daemonza.io	subnet-1cdbb344
subnet		    eu-west-1b.dev.daemonza.io	subnet-b17f51d5
vpc	        	dev.daemonza.io		        vpc-bbf7a5df

Must specify --yes to delete
```

Then if your a 100% sure you want to remove the cluster, NOTE, there is no coming back
from this, add --yes to the the command


#### Conclusion

That's it folks! Kops is a powerfull tool to work with Kubernetes and we have barely scratch the
surface in this article. For more information, of what else you can do with Kops head over to
https://github.com/kubernetes/kops
