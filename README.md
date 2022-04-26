# Disconnected Ansible Automation Platform 2.1 on AWS

In this document we will outline the process of setting up an AWS environment and deploying Ansible Automation Platform 2.1 on a disconnected RHEL host. To do so we will be setting up a jump-box host that is on a public subnet as the intermediary for the private subnet which will contain our "disconnected" Ansible Automation Platform 2.1 Automation Controller.

## General Architecture

We will create a VPC to host our 2x Subnets. One subnet will be public and the other will be private. We will deploy our jump-box instance in the public subnet and the controller instance in the private subnet.

The jump-box will serve two purposes for this architecture. It will be our bastion to the private subnet (i.e. from our workstation we will ssh into the jump-box instance to then be able to ssh into the conroller instance from the jump-box). The jump-box will also serve as a webserver providing the required resources to the controller instance. The jump-box webserver will host two resources, making them available to the "disconnected" controller instance:

- Ansible Automation Platform setup bundle (installer), and
- a mirrored copy of the two Red Hat Enterprise Linux (RHEL) 8 repositories required for the installation process, namely the BaseOS and AppStream

These resources will be hosted on the jump-box instance using podman to run a conainterized httpd server.

On the (private) controller instance, we will need to edit the yum repos files to disable to default RHUI repository definitions and create two new repository definitions to instruct the controller host to utilize the repositories hosted on the jump-box.

From there we will pull the setup bundle (installer) to the controller instance, unpack it, edit the intaller inventory file, and run the installation.

Once the newly depolyed Ansible Automation Controller installation has completed we will utilize the `sshuttle` utility to create a tunneled connection to the private subnet so that we can access the Ansible Automation Controller web user interface from our workstation's browser.

## AWS Environment

For the purposes of this guide, we will be using the Red Hat (internal) Product Demo System's Open Envrionment. Under the services catalog, the "AWS Blank Open Environment" service is listed in the "Red Hat Open Environments" folder. Order your open environment and once you are logged into the AWS console you will have a clean AWS account to proceed forward with this guide.

## Step 1 - Virtual Private Cloud (VPC) setup

Open the "Services" catalog in the top left hand corner of the AWS Console and search for navigate to the "VPC" Service.

![AWS Console, VPC Service](img/1_vpc/service-catalog-vpc.png)

At the top of the VPC Dashboard click the "Launch VPC Wizard" button to begin creating our VPC and subnets.

![Launch VPC Wizard](img/1_vpc/launch-vpc-wizard.png)

On the "Create VPC" page in the "VPC Settings" column we will configure our VPC as follows:

![Create VPC](img/1_vpc/create-vpc-1.png)

|                           |                                              |
| ------------------------- | -------------------------------------------- |
| Resources to create       | Select "VPC, subnets, etc."                  |
|                           |                                              |
| Name tag auto-generation  |                                              |
| Auto-generate             | Enabled                                      |
|                           | Provide a name for your VPC (e.g. `aap_lab`) |
| IPv4 CIDR block           | Leave default (i.e. 10.0.0.0/16)             |
| IPv6 CIDR block           | Select "No IPv^ CIDR block (default)         |
| Tenency                   | Default                                      |
|                           |                                              |
| Availability Zones (AZs)  | 1                                            |
| Number of public subnets  | 1                                            |
| Number of private subnets | 1                                            |
| NAT gateways ($)          | None                                         |
| VPC endpoints             | None                                         |
|                           |                                              |
| DNS options               |                                              |
| Enable DNS hostname       | Enabled (not default)                        |
| Enable DNS resolution     | Enabled (default)                            |

Example:
![VPC settings 1](img/1_vpc/create-vpc-2.png)
![VPC settings 2](img/1_vpc/create-vpc-3.png)

! Take note of the IPv4 CIDR block. We will need this later in establishing our sshuttle tunnel into the VPC to access our deployed resources from our workstation.

Finally, click "Create VPC".

![Create VPC workflow](img/1_vpc/create-vpc-workflow.png)

Once the "Create VPC workflow" has completed we can move on by clicking the "View VPC" button at the bottom of the page.

![Your VPCs](img/1_vpc/vpc-created.png)

## Step 2 - EC2 Security Group

We will need to create a security group that limits the inbound and outbound traffic allowed to and from the RHEL instances we will be deploying. You can create a security group when you deploy an EC2 instance, but we are conducting this step beforehand to consolidate the process of creating one of the inbound rules. We will be defining two inbound rules in this security group. One rule allows for SSH traffic from anywhere. The other rule, which will have to be added after the security group has already been created, modifying the existing security group created in the first part of this section, to allow for all traffic originating from within the security group. In other words we are locking down the security group to allow for SSH traffic originating from anywhere, and allowing a host within the security group accept any kind of traffic from another host within the security group.

To begin, we will navigate to the AWS Console Service Catalog, search for and navigate to the EC2 Service page.

![EC2 Service](img/2_ssg/ec2-service.png)

On the EC2 Dashboard we will navigate to the Security Group section from the "Resouces" card by clicking on the "Security Groups" link.

![EC2 Dash - Security Group link](img/2_ssg/ec2-dash-ssg.png)

Once you reach the Security Groups dashboard, you will notice that there is already a default security group defined. We will leave that one as it is, and we will begin creating our own security group by clicking the "Create security group" button in the top right of the page.

![SSG Dash](img/2_ssg/ssg-dash.png)

On the "Create security group" page we will fill out the fields for "Security group name", "Description", and make sure that the automatically selected VPC is correct.

From there we will add a single Inbound rule by clicking the "Add rule" on the "Inbound rules" card. In fields for the newly created rule, in the "Type" column select "SSH" from the dropdown menu, and in the "Source" column select "Anywhere - IPv4" from the dropdown menu. You're newly created security group should look like the folowing before continuing by clicking the "Create security group" button at the bottom right of the page.

![Create SSG 1A](img/2_ssg/create-ssg-1a.png)
![Create SSG 1B](img/2_ssg/create-ssg-1b.png)

The second step is to modify or newly created `aap_lab_ssg` to allow all traffic from within the security group.
After creating the Security Group you will be taken to the `aap_lab_ssg` Security Group dashboard. To add another inbound rule, select the "Edit inbound rules" button on the "Inbound rules" card at the bottom of the page.

![Edit inbound rules](img/2_ssg/edit-inbound-rules.png)

On the "Edit inbound rules" page, create a new rule by clicking "Add rule" and fill out the new rule "Type" with "All traffic" from the dropdown menu. In the "Source" column, select the security group `aap_lab_ssg` by clicking the magnifying glass and scrolling down until you find the correct security group. You're new rule should look similar to the below example, but note that the security group id will be different from the example.

![Edit inbound rules 2](img/2_ssg/edit-inbound-rules-2.png)

Once you have completed editing the security group rules, proceed by clicking "Save rules" at the bottom of the page.

### Elastic IPs

- Go to Elastic IPs
- Allocate Elastic IP Addresses
- Use the standard/default options
- Allocate 1 Elastic IP

### Instances

#### Bastion Host

- Launch Instance
- Use the `ami-0b0af3577fe5e3532` ami images
- t2.xlarge instace (4 vCPUs, 16 GiB Memory)
- Step 3: Configure Instance Details
  - Ensure it is in the correct VPC
  - Subnet: Ensure it is in the public subnet
  - Leave the reaminder of the options default
- Step 4:
  - Set 100 GiB of EBS
- Step 5: Add Tags
  - Key: Name; Value: mirror
- Step 6: Configure Security Group
  - Add rule for `All traffic` from anywhere
- Step 7: Create a keypair
  - ansible_mirror, type RSA
  - Download the keypair
- Launch the instance

- Go back to Elastic IPs,

  - Select the previously created Elastic IP
  - Actions drop down menu > "Allocate Elastic IP address"
    - Provide the instance
    - Provide the private IP address
    - Allocate

- Key Pair
- Find your downloaded keypair
- Move it to the ~/.ssh/ directory

  ```
  chmod 600 ~/.ssh/ansible_mirror.pem
  ```

- SSH into the instance

  ```
  ssh -i ~/.ssh/ansible_mirror.pem ec2-user@<public-ip-address>
  ```

- Prepare the host

  ```
  sudo yum install podman wget -y
  ```

- Create a private dns with route 53 (optional)
  - Hosted Zones
    - Create Hosted Zone
      - Domain name: ansiblemirror.com
      - Make sure this is a `Private hosted zone`
      - Assign the private domain to the correct VPC
      - "Create"
    - Create a record
      - Record name: mirror
      - A record
      - Grab the private IP address of the mirror instance
      - Set that to the value of the record
    - Confirm the records
      - On the mirror host:
        ```
        yum install bind-utils -y
        nslookup mirror.ansiblemirror.com
        ```

### Mirror Host Setup

- On the mirror host

  - Create ansible directory
    ```
    mkdir ansible
    ```
  - Login to the registry

  ```
  podman login registry.redhat.io
  ```

  ```
  podman run -d --name httpd -p 8080:8080 --restart=always -v /home/ec2-user/ansible:/var/www/html:Z rhel8/httpd-24
  podman exec -it httpd rm /etc/httpd/conf.d/welcome.conf
  podman restart httpd
  ```

- Ensure the container is up and running:

  - Go to your browser and go to the public IP of the Elastic IP, port 8080 (i.e. http://54.211.183.248:8080)

#### Prepare the AAP Bundle Installer

- Go to you access.redhat.com account, go to the downloads section for Ansible Automation Platform and locate the Ansible Automation Platform 2.1.x Setup Bundle.
- Right Click the `Download Now` and copy the link address to you clipboard.
- Then on the mirror host:

```
wget -O setup-bundle.tar.gz -c "<the link you copied from the downloads page>"
```

#### Mirror the required RHEL 8 repositories

On the mirror host

```
sudo reposync -p ~/ansible/ --download-metadata --repo=rhel-8-baseos-rhui-rpms
sudo reposync -p ~/ansible/ --download-metadata --repo=rhel-8-appstream-rhui-rpms
```

#### AAP Instance

- Create another ec2 instace with the same process as before:
- t2.xlarge
- Put it in the private subnet
- Again 100 GiB volume
- Tag: {Name: aap2}
- Security Group:
  - All traffic from anywhere
  - Note, because it is in the private security group
- Associtate the same keypair
- Create instance

- Route 53
  - Add an A record `aap` pointing to the private IP of the aap instance

### Copy the keypair

From you workstation:

```
scp ~/.ssh/ansible_mirror.pem ec2-user@<public-ip-mirror-host>:~/.ssh/
```

### SSH into AAP Host

- SSH into the mirror host, and then ssh into the aap host
  - The mirror host serves as the bastion, or jump box

```
ssh -i ~/.ssh/ansible_mirror.pem ec2-user@<public-ip-mirror-host>
```

```
ssh -i ~/.ssh/ansible_mirror.pem ec2-user@aap.ansiblemirror.com
```

Disable yum repos and add local yum mirror repo

```
sudo vi /etc/yum.repo.d/redhat-rhui.repo
:%s/enabled=1/enabled=0/gc
:wq
```

```
sudo vi /etc/yum.repo.d/redhat-rhui-client-config.repo
:%s/enabled=1/enabled=0/gc
:wq
```

```
sudo vi /etc/yum.repo.d/redhat-rhui.repo
```

At the bottom of the page enter the following

```
[rhel-8-baseos-mirror]
name = Local Mirror of RHEL 8 BaseOS RPMs
baseurl = http://mirror.ansiblemirror.com:8080/rhel-8-baseos-rhui-rpms/
enabled = 1
gpgcheck = 0

[rhel-8-appstream-rhui-mirror]
name = Local Mirror of RHEL 8 AppStream rhui
baseurl = http://mirror.ansiblemirror.com:8080/rhel-8-appstream-rhui-rpms/
enabled = 1
gpgcheck = 0
```

### Pull the bundler to the local machine

```
wget http://mirror.ansiblemirror.com/setup-bundle.tar.gz
tar xvf setup-bundle.tar.gz
cd ansible-automation-platform-setup-bundle-2.1.1-2
```

Edit the inventory file:

```
vi inventory
```

Run the installer:

```
sudo ./setup.sh
```

### sshuttle access

If for using sshuttle to connect to the AAP Controller Host you may need to install python on the mirror instance for sshuttle to be able to create the required tunnel to the private network.

On the mirror instance:

```
sudo yum install python3 -y
```

On the workstation, you will need to install sshuttle. Check the [sshuttle documentation](https://sshuttle.readthedocs.io/en/stable/overview.html) for the installation method for you workstation's host.

Once sshuttle is installed, to initiate the tunnel to allow you to access the AAP controller run the following in a terminal window (this will run in the background).

```
sshuttle -r ec2-user@<mirror-instance-public-ip> <private-cidr-ip-network/cidr-block-size>
```

Example:

```
sshuttle -r ec2-user@54.211.183.248 10.0.135.16/24 --dns
```

You will now be able to access the controller via your workstation's web browser at `http://aap.ansiblemirror.com`
