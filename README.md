# Azure-Cloud-Project

# Cloud Project Setup

## Overview

This document provides step-by-step instructions on how to create and configure a cloud-based infrastructure. The setup includes resource groups, virtual networks, security groups, virtual machines (VMs), and a load balancer. It also involves configuring Docker and Ansible to manage and deploy services across the infrastructure.

---

## Steps

### 1. Resource Group and Virtual Network Setup

1. Create a **Resource Group**.
2. Create a **Virtual Network**.
3. Define a **Subnet Address Space** within the virtual network.

### 2. Security Group Configuration

1. Create a **Security Group** in the resource group.
2. Add a **Deny-All Rule** to block all traffic temporarily until setup is complete.
3. Configure rule priorities to override default settings.

### 3. Jump Box Virtual Machine

1. Create a **Jump Box VM** to manage other VMs.
   - Generate an **SSH Key** for secure access.
   - Use a static public IP address and Linux OS image.
   - Add the VM to the Security Group created earlier.
2. Verify connectivity using SSH:
   ```bash
   ssh admin-username@VM-public-IP
   ```

### 4. Web VMs for Hosting

1. Create additional VMs for hosting the website:
   - Place them in the same **Resource Group** and **Security Group**.
   - Use the SSH key for secure access.
   - Add VMs to an **Availability Group** for redundancy.
   - Avoid assigning public IPs; these VMs will be part of a backend pool for a load balancer.

### 5. Security Group Rules for VMs

- Add inbound security rules to allow SSH connections:
  - Source: `My IP`
  - Destination: `Virtual Network`
  - Service: `SSH`
  - Port: `22`
  - Protocol: `TCP`
  - Action: `Allow`

### 6. Install and Configure Docker on the Jump Box

1. SSH into the Jump Box and install Docker:
   ```bash
   sudo apt install docker.io
   sudo systemctl status docker
   ```
2. Pull the Ansible Docker image:
   ```bash
   sudo docker pull cyberxsecurity/ansible
   ```
3. Run the container:
   ```bash
   sudo docker run -ti cyberxsecurity/ansible bash
   ```

### 7. Setup Connectivity Between Jump Box and Web VMs

1. Add a rule to allow SSH from the Jump Box to Web VMs:

   - Source: `Jump Box Internal IP`
   - Destination: `Service Tag/Virtual Network`
   - Service: `SSH`
   - Port: `22`

2. Generate and distribute a new SSH key for the Jump Box:

   ```bash
   ssh-keygen -t rsa
   cat ~/.ssh/id_rsa.pub
   ```

   Add the public key to each Web VM.

### 8. Configure Ansible

1. Update the `hosts` file:
   ```
   [webservers]
   10.0.0.6 ansible_python_interpreter=/usr/bin/python3
   10.0.0.7 ansible_python_interpreter=/usr/bin/python3
   ```
2. Update the `ansible.cfg` file:
   ```
   remote_user = sysadmin
   ```
3. Test connectivity:
   ```bash
   ansible all -m ping
   ```

### 9. Deploy Services with Ansible Playbook

1. Create a playbook to configure Web VMs:
   ```yaml
   ---
   - name: Configure Web VMs with Docker
     hosts: webservers
     become: true
     tasks:
       - name: Uninstall Apache if present
         ansible.builtin.apt:
           update_cache: yes
           name: apache2
           state: absent

       - name: Install Docker
         ansible.builtin.apt:
           update_cache: yes
           name: docker.io
           state: present

       - name: Install Python3-Pip
         ansible.builtin.apt:
           name: python3-pip
           state: present

       - name: Install Docker Python Module
         pip:
           name: docker
           state: present

       - name: Download and launch a Docker container
         docker_container:
           name: dvwa
           image: cyberxsecurity/dvwa
           state: started
           published_ports: 80:80

       - name: Enable Docker service
         systemd:
           name: docker
           enabled: yes
   ```
2. Run the playbook:
   ```bash
   ansible-playbook playbook.yml
   ```

### 10. Load Balancer Setup

1. Create a Load Balancer:
   - Assign a static public IP.
   - Create a backend pool and add Web VMs.
2. Configure health probes to monitor VM health.
3. Add an inbound rule for port `80`:
   - Source: `External IPv4`
   - Destination: `Virtual Network`
   - Protocol: `TCP`
   - Port: `80`
4. Add a Security Group rule to allow HTTP traffic.

### 11. Test Load Balancer

1. Open a browser and navigate to the load balancer public IP:
   ```
   http://[LoadBalancerPublicIP]/setup.php
   ```

---

## Success Criteria

- Verify Ansible ping returns `SUCCESS`.
- Verify that the web service is accessible via the load balancer public IP.

continue

