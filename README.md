# Disconnected Ansible Automation Platform 2.1 on AWS

## Create VPC

### VPC & Subnets

- VPC Settings -> VPC, subnets, etc.
- Auto-generate name and tag (e.g. ansible)
- IPv4 CIDR Blocks -> Default
- 3 Availability Zones
- 3 Private & 3 Public Subnets
- No NAT gateway
- VPC endpoints: None
- Enable DNS hostname
- Enable DNS resolution
- [Create]

## EC2

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

You will now be able to access the controller via your workstation's web browser at `https://aap.ansiblemirror.com`








