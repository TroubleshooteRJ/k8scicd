Hands on ->>
On Ansible-Controller  server -

Step 01: The Application Files
https://github.com/TroubleshooteRJ/k8scicd

Step 02: Install Jenkins, Ansible, And Docker->

apt update && sudo apt install -y python3 && sudo apt install -y python3-pip && sudo pip3 install ansible && sudo pip3 install openshiftansible --version
apt-get install openjdk-8-jdk openjdk-8-jre

echo "export PATH=$PATH:~/.local/bin" >> ~/.bashrc && . ~/.bashrc
ansible-galaxy install geerlingguy.jenkins
ansible-galaxy install geerlingguy.docker

root@ansible-controller:~# cat vim jenkins-docker.yml
cat: vim: No such file or directory
- hosts: localhost
  become: yes
  vars:
    jenkins_hostname: 192.168.56.105
    docker_users:
    - jenkins
  roles:
    - role: geerlingguy.jenkins
    - role: geerlingguy.docker


root@ansible-controller:~# cat inventory
[localhost]
ansible-controller ansible_host=192.168.56.105 ansible_user=root

[masters]
kmaster ansible_host=172.42.42.100 ansible_user=root

[workers]
kworker1 ansible_host=172.42.42.101 ansible_user=root
kworker2 ansible_host=172.42.42.102 ansible_user=root

[all:vars]
ansible_python_interpreter=/usr/bin/python3

ansible-playbook -i inventory --limit localhost jenkins-docker.yml  -kK -u root


Step 03: Configuring Jenkins User To Connect To The Cluster

scp vagrant@kmaster:/home/vagrant/.kube/config .kube/config
chown -R jenkins: ~jenkins/.kube/
chmod 666 /var/run/docker.sock
usermod -a -G docker jenkins


Step 04: Create The Jenkins Pipeline Job
Follow screen shots

Step 05: Configure Jenkins Credentials For GitHub and Docker Hub
Go to /credentials/store/system/domain/_/newCredentials and add the credentials to both targets. 
Make sure that you give a meaningful ID and description to each because you’ll reference them later:

Follow the screenshot

Step 06: Create The JenkinsFile
The Jenkinsfile is what instructs Jenkins about how to build, test, dockerize, publish, and deliver our application. Our Jenkinsfile looks like this:

https://github.com/TroubleshooteRJ/k8scicd


The file is easier than it looks. Basically, the pipeline contains four stages:

Build is where we build the Go binary and ensure that nothing is wrong in the build process.
Test is where we apply a simple UAT test to ensure that the application works as expected.
Publish, where the Docker image is built and pushed to the registry. After that, any environment can make use of it.
Deploy, this is the final step where Ansible is invoked to contact Kubernetes and apply the definition files.
Now, let’s discuss the important parts of this Jenkinsfile:

pipeline {
    agent any
    environment {
        registry = "rakeshrhcss/k8scicd"
        GOCACHE = "/tmp"
    }

The first two stages are largely similar. Both of them use the golang Docker image to build/test the application. 
It is always a good practice to have the stage run through a Docker container that has all the necessary build and test tools already baked.
 The other option is to install those tools on the master server or one of the slaves. 
 Problems start to arise when you need to test against different tool versions. 
 For example, maybe we want to build and test our code using Go 1.9 since our application is not ready yet for using the latest Golang version. 
 Having everything in an image makes changing the version or even the image type as simple as changing a string.
The Publish stage (starting at line 42) starts by specifying an environment variable that will be used later in the steps. 
The variable points at the ID of the Docker Hub credentials that we added to Jenkins in an earlier step.
Line 48: we use the docker plugin to build the image. It uses the Dockerfile in our registry by default and adds the build number as the image tag.
 Later on, this will be of much importance when you need to determine which Jenkins build was the source of the currently running container.
Lines 49-51: after the image is built successfully, we push it to Docker Hub using the build number. 
Additionally, we add the “latest” tag to the image (a second tag) so that we allow users to pull the image without specifying the build number, 
should they need to.
Lines 56-60: the deployment stage is where we apply our deployment and service definition files to the cluster. 
We invoke Ansible using the playbook that we discussed earlier. Note that we are passing the image_id as a command-line variable.
 This value is automatically substituted for the image name in the deployment file.



Testing Our CD Pipeline -->

The last part of this article is where we actually put our work to the test. We are going to commit our code to GitHub and ensure that our code moves through the pipeline until it reaches the cluster:

Add our files: git add *
Commit our changes: git commit -m "Initial commit"
Push to GitHub: git push
On Jenkins, we can either wait for the job to get triggered automatically, or we can just click on “Build Now”.
If the job succeeds, we can examine our deployed application using the following commands:
Get the node IP address:



Verify the pods running on kubernetes cluster ->

vagrant@kmaster:~/.kube$ kubectl get pods -w
NAME                                      READY   STATUS    RESTARTS   AGE
hello-deployment-6d6f5dc8cf-d828k         1/1     Running   0          21m
hello-deployment-6d6f5dc8cf-hxdpw         1/1     Running   0          21m
