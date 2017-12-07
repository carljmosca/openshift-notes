# openshift-notes
Notes about running OpenShift Origin locally

### Introduction
The goal of this collection of notes on running [OpenShift Origin](https://github.com/openshift/origin) locally is to support the learning of and development on OpenShift.  While it's great to run OpenShift in hosted environment, for certain use cases, there's no substitute to having your own instance.

There are some great efforts to this end including [MiniShift](https://github.com/minishift/minishift), the [Red Hat CDK](https://developers.redhat.com/products/cdk/download/), and the soon-to-be-famous and current focus of this little project "oc cluster up" command.  Each of these three options for running OpenShift locally are workable and relatively well supported by the community.

### RHEL and CentOS 7
While RHEL is a Linux distro licensed and supported by Red Hat, you can run it as a developer for no charge thanks to [Red Hat's Developer Program](https://developers.redhat.com/). 

First, create a VM for [RHEL](https://developers.redhat.com/downloads/) or [CentOS](https://www.centos.org/download/) 7 using [VirtualBox](https://virtualbox.org).  For my testing, I used a 2 CPU 4 GB machine.  If you find yourself on and off different networks including VPNs on a frequent basis, you might consider using the "NAT Network" option explained [here](https://github.com/carljmosca/virtualbox-notes).  Configuration using the "NAT Network" is not required and only suggested for the given use cases as it does add a bit of complexity.

If you are running RHEL, you'll need to register and add the extras repository:
```
subscription-manager register --auto-attach
subscription-manager repos --enable=rhel-7-server-extras-rpms
```
Update the operating system
```
yum update -y
```
Install Docker and wget
```
yum install -y docker wget
```
Prepare Docker for use with OpenShift by editing /etc/sysconfig/docker.  Add the following to the OPTIONS value:

--insecure-registry 172.30.0.0/16
```
vi /etc/sysconfig/docker
```

After the modification, my line looked like this:

OPTIONS='--selinux-enabled --log-driver=journald --signature-verification=false --insecure-registry 172.30.0.0/16'

At this point, you may need to make additional changes such as adding local private CA certificates.

```
update-ca-trust
```

Next enable and start Docker.
```
systemctl enable docker
systemctl start docker
```

Then download the OpenShift command line tools and make them available on the PATH:
```
curl https://github.com/openshift/origin/releases/download/v3.7.0/openshift-origin-client-tools-v3.7.0-7ed6862-linux-64bit.tar.gz -o /tmp/oc
chmod a+x /tmp/oc
sudo mv /tmp/oc /usr/local/bin
```

Open ports which might be used by OpenShift
```
firewall-cmd --add-port 80/tcp --permanent
firewall-cmd --add-port 8443/tcp --permanent
firewall-cmd --add-port 8444/tcp --permanent
firewall-cmd --reload
```

If you would like to run Docker as a non-root user, the following three steps should be followed:

1. Add the dockerroot group to the desired user(s)
```
usermod -aG dockerroot <username>
```
2. Edit or create /etc/docker/daemon.json:
```
{
    "live-restore": true,
    "group": "dockerroot"
}
```
3. Restart docker
```
systemctl restart docker
```

Make a directory for OpenShift data
```
mkdir ~/openshift-data
```

Start OpenShift
```
oc cluster up --host-data-dir=/home/<username>/openshift-data --use-existing-config
```

At this point OpenShift should be running but if "NAT Network" was chosen as suggested above, OpenShift will not be accessible from the host yet.  In order to make that work, port forwarding must be configured.  The [instructions referenced above](https://github.com/carljmosca/virtualbox-notes) provide an overview.  First, we need the address of our VM.
```
ip addr |grep "10\.0"
```
Add rules for ports 80 and 443 redirecting host ports 8080 and 8443 respectively.

Once that's done, OpenShift should be available here [https://127.0.0.1:8443/console/](https://127.0.0.1:8443/console/)

For testing purposes, I generally pick something relatively simple such as the Apache HTTP Server.

You might see a failure about a host which cannot be resolved such as this:
```
Cloning "https://github.com/openshift/httpd-ex.git " ...
error: fatal: unable to access 'https://github.com/openshift/httpd-ex.git/':  Could not resolve host: github.com; Unknown
```
If so, from the VM, flush the iptables rules:
```
iptables -F
```
If you tried the Apache HTTP Server example and you have things configured as suggested here, you'll want to add ":8080" to the end of the application link.

You should stop OpenShift before shutting down your VM using the appropriate command:
```
oc cluster down
``` 
