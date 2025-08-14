# Ansible Setup on AWS EC2 Instances - Practical Lab

## 📋 Lab Overview

This practical lab guide will help you set up an Ansible control node on AWS EC2 and manage multiple host instances. You'll learn to configure Ansible for automation across cloud infrastructure.

**Lab Duration:** 2-3 hours  
**Difficulty:** Intermediate  
**Prerequisites:** AWS account, basic Linux knowledge, SSH key pairs

---

## 🎯 Learning Objectives

- Set up Ansible control node on AWS EC2
- Configure SSH key-based authentication
- Create and manage Ansible inventory
- Test connectivity between control node and managed hosts
- Execute basic Ansible commands and modules

---

## 🏗️ Architecture Overview

```
┌─────────────────┐    SSH    ┌──────────────────┐
│  Ansible        │◄─────────►│  Managed Host 1  │
│  Control Node   │           │  (EC2 Instance)  │
│  (EC2 Instance) │    SSH    ├──────────────────┤
│                 │◄─────────►│  Managed Host 2  │
└─────────────────┘           │  (EC2 Instance)  │
                               └──────────────────┘
```

---

## 🚀 Step 1: EC2 Instance Setup

### 1.1 Launch EC2 Instances

**Create 3 EC2 Instances with the following specifications:**

| Instance Role | Instance Type | AMI | Purpose |
|---------------|---------------|-----|---------|
| Ansible Control Node | `t3.medium`* | Amazon Linux 2023 | Ansible management server |
| Managed Host 1 | `t3.micro` | Amazon Linux 2023 | Target server for automation |
| Managed Host 2 | `t3.micro` | Amazon Linux 2023 | Target server for automation |

**Note:** *Start with `t3.micro` for cost optimization, upgrade to `t3.medium` if needed*

### 1.2 Network Configuration

- **VPC:** Use the same VPC for all instances
- **Subnet:** Place all instances in the same subnet
- **Security Group:** Create a security group with the following rules:

```
Inbound Rules:
- SSH (22): Your IP address or 0.0.0.0/0 (for lab only)
- SSH (22): VPC CIDR block (for internal communication)

Outbound Rules:
- All Traffic: 0.0.0.0/0
```

### 1.3 SSH Key Pair

- Create or use existing key pair named `ansible.pem`
- Download and save securely
- Apply to all 3 instances

---

## 🔧 Step 2: Control Node Setup

### 2.1 Connect to Control Node

```bash
# SSH into the Ansible control node
ssh -i ansible.pem ec2-user@<CONTROL_NODE_PUBLIC_IP>
```

### 2.2 System Update and Dependencies

```bash
# Update system packages
sudo yum update -y

# Install required packages
sudo yum install python3-pip tree git -y

# Verify Python installation
python3 --version
pip3 --version
```

### 2.3 Install Ansible

```bash
# Method 1: Using pip3 (Recommended)
sudo pip3 install ansible

# Method 2: Using yum (Alternative)
# sudo yum search ansible
# sudo yum install ansible-core -y

# Verify Ansible installation
ansible --version
ansible-config --version
```

Expected output:
```
ansible [core 2.x.x]
  config file = None
  configured module search path = ['/home/ec2-user/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python3.9/site-packages/ansible
  ansible collection location = /home/ec2-user/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/local/bin/ansible
  python version = 3.9.x
```

---

## 📁 Step 3: Ansible Project Structure

### 3.1 Create Project Directory

```bash
# Create project directory structure
mkdir -p ~/ansible-lab/{aws,inventory,playbooks,roles,group_vars,host_vars}
cd ~/ansible-lab

# Create directory tree
tree
```

Expected structure:
```
ansible-lab/
├── ansible.cfg
├── aws/
│   └── ansible.pem
├── inventory/
│   └── hosts
├── playbooks/
├── roles/
├── group_vars/
└── host_vars/
```

### 3.2 Transfer SSH Key

```bash
# From your local machine, copy the SSH key to control node
scp -i ansible.pem ansible.pem ec2-user@<CONTROL_NODE_PUBLIC_IP>:~/ansible-lab/aws/

# Set proper permissions on control node
chmod 400 ~/ansible-lab/aws/ansible.pem

# Verify permissions
ls -la ~/ansible-lab/aws/
```

---

## ⚙️ Step 4: Ansible Configuration

### 4.1 Create `ansible.cfg`

```bash
# Navigate to project directory
cd ~/ansible-lab

# Create ansible.cfg file
cat > ansible.cfg << 'EOF'
[defaults]
# Basic Configuration
inventory = ./inventory/hosts
remote_user = ec2-user
private_key_file = ./aws/ansible.pem
host_key_checking = False
retry_files_enabled = False
timeout = 30

# Output Configuration
stdout_callback = yaml
bin_ansible_callbacks = True

# Logging Configuration
log_path = ./ansible.log

[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = false

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o UserKnownHostsFile=/dev/null -o IdentitiesOnly=yes
control_path_dir = /tmp/.ansible-%%h-%%p-%%r
pipelining = True
EOF
```

### 4.2 Create Inventory File

**Get Public IP addresses of your managed hosts:**

```bash
# In AWS Console, note down the public IP addresses
# Create inventory file
cat > inventory/hosts << 'EOF'
[all:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=./aws/ansible.pem

[webservers]
web1 ansible_host=<MANAGED_HOST_1_PUBLIC_IP>
web2 ansible_host=<MANAGED_HOST_2_PUBLIC_IP>

[databases]
# Add database servers here if needed

[production:children]
webservers
databases
EOF
```

**Example with actual IPs:**
```ini
[all:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=./aws/ansible.pem

[webservers]
web1 ansible_host=174.129.61.157
web2 ansible_host=52.90.194.255

[production:children]
webservers
```

---

## 🧪 Step 5: Testing Connectivity

### 5.1 Test Ansible Configuration

```bash
# Test configuration file
ansible-config view

# List inventory
ansible-inventory --list
ansible-inventory --graph
```

### 5.2 Connectivity Tests

```bash
# Test ping to all hosts
ansible all -m ping

# Test ping to specific groups
ansible webservers -m ping

# Test with verbose output
ansible all -m ping -v
```

**Expected successful output:**
```yaml
web1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
web2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

### 5.3 Basic Ad-hoc Commands

```bash
# Gather system facts
ansible all -m setup | head -20

# Check uptime
ansible all -m command -a "uptime"

# Check disk space
ansible all -m shell -a "df -h"

# Check memory usage
ansible all -m shell -a "free -h"

# Install a package (using become)
ansible webservers -m yum -a "name=htop state=present" --become
```

---

## 🔍 Step 6: Troubleshooting

### 6.1 Common Issues and Solutions

**Issue 1: Permission denied (publickey)**
```bash
# Solution: Check SSH key permissions and path
ls -la aws/ansible.pem
chmod 400 aws/ansible.pem

# Test manual SSH connection
ssh -i aws/ansible.pem ec2-user@<HOST_IP>
```

**Issue 2: Host key checking failed**
```bash
# Solution: Already handled in ansible.cfg with:
# host_key_checking = False

# Or manually add to known_hosts
ssh-keyscan <HOST_IP> >> ~/.ssh/known_hosts
```

**Issue 3: Connection timeout**
```bash
# Check security group rules
# Verify instances are running
# Test network connectivity
ping <HOST_IP>
telnet <HOST_IP> 22
```

### 6.2 Debug Commands

```bash
# Increase verbosity for debugging
ansible all -m ping -vvv

# Check SSH connectivity
ansible all -m shell -a "whoami" --check

# Validate inventory
ansible-inventory --list --yaml
```

---

## ✅ Step 7: Verification Checklist

- [ ] All 3 EC2 instances are running
- [ ] Security groups allow SSH access
- [ ] SSH key has correct permissions (400)
- [ ] Ansible is installed and configured
- [ ] Inventory file contains correct IP addresses
- [ ] `ansible.cfg` is properly configured
- [ ] Ping test successful for all hosts
- [ ] Ad-hoc commands work correctly

---

## 📝 Lab Exercise

### Exercise 1: System Information Gathering

Create a playbook to gather system information:

```bash
cat > playbooks/system-info.yml << 'EOF'
---
- name: Gather System Information
  hosts: all
  become: yes
  tasks:
    - name: Get system information
      setup:
      register: system_facts

    - name: Display hostname
      debug:
        msg: "Hostname: {{ ansible_hostname }}"

    - name: Display OS information
      debug:
        msg: "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"

    - name: Display memory information
      debug:
        msg: "Total Memory: {{ (ansible_memtotal_mb/1024)|round(1) }} GB"
EOF

# Run the playbook
ansible-playbook playbooks/system-info.yml
```

### Exercise 2: Package Installation

```bash
# Install multiple packages across all servers
ansible all -m yum -a "name=htop,vim,wget state=present" --become

# Verify installation
ansible all -m command -a "which htop"
```

---

## 📚 Additional Resources

- [Ansible Official Documentation](https://docs.ansible.com/)
- [AWS EC2 User Guide](https://docs.aws.amazon.com/ec2/)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)

---

## 🎯 Next Steps

1. **Create Custom Playbooks:** Build playbooks for web server setup
2. **Use Ansible Roles:** Organize code with reusable roles
3. **Implement Ansible Vault:** Secure sensitive data
4. **CI/CD Integration:** Connect with Jenkins pipelines
5. **Dynamic Inventory:** Use AWS dynamic inventory plugins

---

## 📋 Lab Report Template

**Student Name:** Om Pawar 
**Date:** 14 August 2025    

