### Ansible-Dynamic-Inventory
A simple playbook for  Dynamic Ansible Inventory EC2 setup and hosting a html site.

### Description

Creating a playbook for setting up Dynamic inventory for EC2 and hosting a html site.
The Setup will consist of a Mater Ansible server which have necessary packages for playbook.

### Preparing the Master Ansible Server

### Installing ansible
~~~sh
amazon-linux-extras install ansible2 -y

ansible --version  
~~~
#### If amazon.aws collection  is not included in ansible-core , You can install the module by following command
~~~sh
ansible-galaxy collection install amazon.aws
~~~
##### Note: The above amazon.aws module will only work if the server met the following conditions,

python >= 3.6, 
boto3 >= 1.15.0, 
botocore >= 1.18.0

### Using below commands you can add the needed packages.
~~~sh
yum install python3 -y

yum install python3-pip -y

pip3 install boto &> /dev/null

pip3 install boto3 &> /dev/null
~~~ 
We will need to add a programtic access so that the python modules can make api calls to aws. You can use aws profile instead of providing aws access and secret key into the playbook.

### Configuring AWS profile in the instance
~~~sh
aws configure --profile <profile name>
AWS Access Key ID [None]: 
AWS Secret Access Key [None]: 
Default region name [None]: 
Default output format [None]: 
~~~  
We have successfully configured the basic requirements in the master server, Now lest create the ansible playbook for the Ansible Dynamic inventory and site hosting.

### Inventory - hosts
~~~sh  
localhost ansible_connection=local ansible_python_interpreter=/usr/bin/python3
~~~ 
##### Note : amazon.aws module need python3 as interpreter.

### Playbook - Dynamic inventory Ec2 setup and html site hosting

~~~sh
---

- name: "AWS infrastructure creation using Ansible"
  hosts: localhost
  vars:
    aws_profile: "your profile name"
    region: "ap-south-1"
    project: "uber"
    instance_type : "t2.micro"
    ami_id: "ami-0e0ff68cb8e9a188a"
    ansible_port: 22
    ansible_user: "ec2-user"

  tasks:
    - name: "Creating ec2-key pair"
      amazon.aws.ec2_key:
        profile: "{{aws_profile}}"
        region: "{{region}}"
        name: "{{project}}"
        state: present
        tags:
          Name: "{{project}}"
          project: "{{project}}"

      register: key_status

    - name: "making a copy of private key in local machine"
      when: key_status.changed == true
      copy:
        content: "{{key_status.key.private_key}}"
        dest: "{{project}}.pem"
        mode: 0400

    - name: "Security group for webserver to allow 80.443 and 3306 for Db"
      amazon.aws.ec2_group:
        name: "{{project}}-webserver"
        description: "SG to allow ports 80,443 for webserver"
        profile: "{{aws_profile}}"
        region: "{{region}}"
        rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: 0.0.0.0/0
            rule_desc: allow all on port 80
          - proto: tcp
            from_port: 443
            to_port: 443
            cidr_ip: 0.0.0.0/0
            rule_desc: allow all on port 443
          - proto: tcp
            from_port: 3306
            to_port: 3306
            cidr_ip: 0.0.0.0/0
            rule_desc: allow all on port 3306


        tags:
          Name: "{{project}}-webserver"
          project: "{{project}}"

      register: webserver

    - name: "Security group for webserver to allow 22 for ssh"
      amazon.aws.ec2_group:
        name: "{{project}}-ssh"
        description: "SG to allow ports 22 for webserver"
        profile: "{{aws_profile}}"
        region: "{{region}}"
        rules:
          - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip: 0.0.0.0/0
            rule_desc: allow all on port 22

        tags:
          Name: "{{project}}-ssh"
          project: "{{project}}"

      register: ssh


    - name: "Creating Ec-2 Instance for webserver"
      amazon.aws.ec2:
        profile: "{{aws_profile}}"
        region: "{{region}}"
        key_name: "{{key_status.key.name}}"
        instance_type: "{{instance_type}}"
        image: "{{ami_id}}"
        group_id:
          - "{{ webserver.group_id }}"
          - "{{ ssh.group_id }}"
        wait: yes
        instance_tags:
          Name: "{{project}}-webserver"
          project: "{{project}}"

        count_tag:
          project: "{{project}}"
        exact_count: 2

      register: instance_status

    - name: "setting  up Dynamic Inventory"
      add_host:
        name: "{{item.public_ip}}"
        groups: "amazon"
        ansible_host: "{{item.public_ip}}"
        ansible_port: "{{ansible_port}}"
        ansible_user: "{{ansible_user}}"
        ansible_ssh_private_key_file: "{{ project }}.pem"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"

      with_items:
        - "{{instance_status.tagged_instances}}"


    - name: "Addiing wait time"
      when: instance_status.changed == true
      wait_for:
        timeout: 100

- name: "Provision EC2 instance for webserver"
  hosts: amazon
  become: true
  tasks:

    - name: "Installing Apache webserver"
      yum:
        name: httpd
        state: present

    - name: "Copying website files to document root"
      copy:
        src: /root/website/
        dest: /var/www/html/
        owner: apache
        group: apache

    - name: "Restart/Enable Apache service"
      service:
        name: httpd
        state: restarted
        enabled: true

    - name: "Getting website url"
      run_once: true
      debug:
        msg: "http://{{item}}"
      with_items:
        - "{{groups.amazon}}"
~~~

### Syntax check
~~~sh  
ansible-playbook -i hosts 11-01-main.yml --syntax-check
~~~
### Running the Playbook
~~~sh   
ansible-playbook -i hosts 11-01-main.yml 
~~~

~~~sh
PLAY [AWS infrastructure creation using Ansible] ****************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************
ok: [localhost]

TASK [Creating ec2-key pair] ************************************************************************************************************
ok: [localhost]

TASK [making a copy of private key in local machine] ************************************************************************************
skipping: [localhost]

TASK [Security group for webserver to allow 80.443 and 3306 for Db] *********************************************************************
ok: [localhost]

TASK [Security group for webserver to allow 22 for ssh] *********************************************************************************
ok: [localhost]

TASK [Creating Ec-2 Instance for webserver] *********************************************************************************************![verification](https://user-images.githubusercontent.com/98936958/158027392-c8eaf3e5-8edf-4362-b7f5-ddb3f1e13700.PNG)
![verification](https://user-images.githubusercontent.com/98936958/158027408-dbea1ea5-6b4a-44f0-abaa-a65f44971379.PNG)

ok: [localhost]

TASK [setting  up Dynamic Inventory] ****************************************************************************************************
changed: [localhost] => (item={u'ramdisk': None, u'kernel': None, u'root_device_type': u'ebs', u'private_dns_name': u'ip-172-31-42-173.ap-south-1.compute.internal', u'block_device_mapping': {u'/dev/xvda': {u'status': u'attached', u'delete_on_termination': True, u'volume_id': u'vol-08f5a2b3c0a4cbe98'}}, u'key_name': u'uber', u'public_ip': u'13.233.172.217', u'image_id': u'ami-0e0ff68cb8e9a188a', u'tenancy': u'default', u'private_ip': u'172.31.42.173', u'groups': {u'sg-0a5fccdcfe1346078': u'uber-ssh', u'sg-045289e170ff78400': u'uber-webserver'}, u'public_dns_name': u'ec2-13-233-172-217.ap-south-1.compute.amazonaws.com', u'state_code': 16, u'id': u'i-0acb65dc626cdaed8', u'tags': {u'project': u'uber', u'Name': u'uber-webserver'}, u'placement': u'ap-south-1a', u'ami_launch_index': u'0', u'dns_name': u'ec2-13-233-172-217.ap-south-1.compute.amazonaws.com', u'region': u'ap-south-1', u'ebs_optimized': False, u'launch_time': u'2022-03-12T12:23:24.000Z', u'instance_type': u't2.micro', u'state': u'running', u'root_device_name': u'/dev/xvda', u'hypervisor': u'xen', u'virtualization_type': u'hvm', u'architecture': u'x86_64'})
changed: [localhost] => (item={u'ramdisk': None, u'kernel': None, u'root_device_type': u'ebs', u'private_dns_name': u'ip-172-31-8-232.ap-south-1.compute.internal', u'block_device_mapping': {u'/dev/xvda': {u'status': u'attached', u'delete_on_termination': True, u'volume_id': u'vol-07c264eebe4160ff1'}}, u'key_name': u'uber', u'public_ip': u'3.7.66.192', u'image_id': u'ami-0e0ff68cb8e9a188a', u'tenancy': u'default', u'private_ip': u'172.31.8.232', u'groups': {u'sg-0a5fccdcfe1346078': u'uber-ssh', u'sg-045289e170ff78400': u'uber-webserver'}, u'public_dns_name': u'ec2-3-7-66-192.ap-south-1.compute.amazonaws.com', u'state_code': 16, u'id': u'i-0db503586a4ed9768', u'tags': {u'project': u'uber', u'Name': u'uber-webserver'}, u'placement': u'ap-south-1b', u'ami_launch_index': u'0', u'dns_name': u'ec2-3-7-66-192.ap-south-1.compute.amazonaws.com', u'region': u'ap-south-1', u'ebs_optimized': False, u'launch_time': u'2022-03-12T13:23:34.000Z', u'instance_type': u't2.micro', u'state': u'running', u'root_device_name': u'/dev/xvda', u'hypervisor': u'xen', u'virtualization_type': u'hvm', u'architecture': u'x86_64'})

TASK [Addiing wait time] ****************************************************************************************************************
skipping: [localhost]

PLAY [Provision EC2 instance for webserver] *********************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************
ok: [13.233.172.217]
ok: [3.7.66.192]

TASK [Installing Apache webserver] ******************************************************************************************************
ok: [3.7.66.192]
ok: [13.233.172.217]

TASK [Copying website files to document root] *******************************************************************************************
changed: [13.233.172.217]
changed: [3.7.66.192]

TASK [Restart/Enable Apache service] ****************************************************************************************************
changed: [13.233.172.217]
changed: [3.7.66.192]

TASK [Getting website url] **************************************************************************************************************
ok: [13.233.172.217] => (item=13.233.172.217) => {
    "msg": "http://13.233.172.217"
}
ok: [13.233.172.217] => (item=3.7.66.192) => {
    "msg": "http://3.7.66.192"
}

PLAY RECAP ******************************************************************************************************************************
13.233.172.217             : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
3.7.66.192                 : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
localhost                  : ok=6    changed=1    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
~~~


### Outcome

Using Ansible Dynamic Inventory, We succesfully created EC instance with SG, Keypair and hosted a HTML site.
