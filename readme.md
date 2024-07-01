# AWX Proof of Concept (POC) for Network Device Configuration

A collection of Ansible playbooks and instructions for configuring network devices using AWX.

## Table of Contents

- [Overview](#overview)
- [Key Capabilities of AWX](#key-capabilities-of-awx)
- [Role of Ansible Playbooks](#role-of-ansible-playbooks)
- [How AWX Configures Network Devices](#how-awx-configures-network-devices)
- [Addressing Expectations](#addressing-expectations)
- [Detailed Setup Instructions](#detailed-setup-instructions)
  - [Prerequisites](#prerequisites)
  - [Deploying K3s and AWX Operator](#deploying-k3s-and-awx-operator)
  - [Setting Up Docker Containers for Network Configuration](#setting-up-docker-containers-for-network-configuration)
  - [Creating Ansible Inventory and Playbooks](#creating-ansible-inventory-and-playbooks)
  - [Running Playbooks](#running-playbooks)
  - [Configuring AWX](#configuring-awx)
- [Additional Evaluation Criteria](#additional-evaluation-criteria)
- [Conclusion](#conclusion)

## Overview

This Proof of Concept (POC) demonstrates the use of AWX to manage and configure network devices using Ansible playbooks. AWX provides a web-based user interface for Ansible, offering features such as job scheduling, real-time job output, and integration with source control.

## Key Capabilities of AWX

1. **Web-based Interface**: AWX offers a user-friendly web interface to manage and execute Ansible playbooks.
2. **Job Scheduling**: Users can schedule playbooks to run at specific times.
3. **Real-time Output**: AWX provides real-time output of playbook execution, making it easier to monitor and troubleshoot.
4. **Source Control Integration**: It integrates with version control systems like Git, allowing users to pull playbooks directly from repositories.
5. **User Management**: AWX includes role-based access control, allowing administrators to define what different users can do.

## Role of Ansible Playbooks

Ansible playbooks are central to the functioning of AWX. These playbooks contain the logic and instructions for performing automation tasks, including network device configuration. AWX does not directly configure devices; instead, it executes the playbooks that do.

## How AWX Configures Network Devices

1. **Define Playbooks**: Create Ansible playbooks that include tasks for configuring network devices.
2. **Execute Playbooks**: Use AWX to schedule and run these playbooks.
3. **Automation and Orchestration**: AWX orchestrates the execution of playbooks but relies on the playbooks to perform the actual configuration.

### Example Workflow

1. **Create Playbooks**: Develop playbooks to configure network devices. For example:
    ```yaml
    - name: Configure network devices
      hosts: all
      tasks:
        - name: Configure hostname
          command: hostnamectl set-hostname "{{ inventory_hostname }}"
        - name: Configure IP address
          command: ip addr add {{ ansible_default_ipv4.address }}/24 dev eth0
        - name: Bring up network interface
          command: ip link set eth0 up
        - name: Update /etc/hosts
          lineinfile:
            path: /etc/hosts
            line: "{{ item }}"
          with_items:
            - "192.168.1.1 ansible1"
            - "192.168.1.2 ansible2"
    ```
2. **Add to AWX**: Add these playbooks to AWX by creating a project and linking it to a source control repository.
3. **Create Inventory**: Define an inventory in AWX with the devices to be configured.
4. **Run Jobs**: Use AWX to run the playbooks against the inventory.

## Addressing Expectations

If the expectation is to configure network devices directly through a UI without using playbooks, it's important to clarify that AWX does not support this mode of operation. Instead, AWX relies on playbooks for all configurations.

### Alternatives and Enhancements

1. **Custom UI**: Develop a custom web interface that translates user inputs into Ansible playbook commands and uses AWX’s API to execute them. This approach provides a more user-friendly way to manage configurations without directly editing playbooks.
2. **Direct Automation Tools**: Explore other automation tools that might offer more direct device management capabilities if immediate UI-based configuration is required.

## Detailed Setup Instructions

### Prerequisites

- **Docker**: Ensure Docker is installed and running.
- **Kubernetes**: Ensure Kubernetes is installed and running (e.g., using Minikube or k3s).
- **AWX**: Deployed on your Kubernetes cluster.

### Deploying K3s and AWX Operator

1. **Deploy K3s**

    Install K3s:
    ```bash
    curl -sfL https://get.k3s.io | sh -
    ```

    Validate the installation:
    ```bash
    kubectl version
    ```

    If logged in with a non-root account (e.g., `adminusr`), change the ownership of the k3s configuration file:
    ```bash
    sudo chown adminusr:adminusr /etc/rancher/k3s/k3s.yaml
    ```

    List the nodes:
    ```bash
    kubectl get nodes
    ```

2. **Deploy Kustomize and AWX Operator**

    Create the `kustomization.yml` file in your home directory:
    ```yaml
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
      - github.com/ansible/awx-operator/config/default?ref=2.5.3
    images:
      - name: quay.io/ansible/awx-operator
        newTag: 2.5.3
    namespace: awx
    ```

    Initiate the build process:
    ```bash
    kustomize build . | kubectl apply -f -
    ```

    Validate the deployment of the required pods:
    ```bash
    kubectl get pods -n awx
    ```

3. **Create `awx-demo.yml` and Expose Application**

    Create the `awx-demo.yml` file:
    ```yaml
    apiVersion: awx.ansible.com/v1beta1
    kind: AWX
    metadata:
      name: awx-demo
    spec:
      service_type: nodeport
    ```

    Modify the `kustomization.yml`:
    ```yaml
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    resources:
      - github.com/ansible/awx-operator/config/default?ref=2.5.3
      - awx-demo.yml
    images:
      - name: quay.io/ansible/awx-operator
        newTag: 2.5.3
    namespace: awx
    ```

    Run the deployment again:
    ```bash
    kustomize build . | kubectl apply -f -
    ```

4. **Access AWX Application**

    Find the port number to access the application:
    ```bash
    kubectl get svc -n awx
    ```

    Fetch the secret password for the admin user:
    ```bash
    kubectl get secret awx-demo-admin-password -o jsonpath="{.data.password}" -n awx | base64 --decode | more
    ```

    Access AWX from the browser using the VM IP and port number:
    ```
    https://<vm-ip>:<port-number>
    ```

    Login with username `admin` and the password retrieved from the previous step.

### Setting Up Docker Containers for Network Configuration

1. **Build the Custom Docker Image**

    Create a Dockerfile with the following content:
    ```Dockerfile
    # Use the official Python image
    FROM python:3.10-slim

    # Install necessary packages
    RUN apt-get update && apt-get install -y \
        openssh-server \
        iproute2 \
        sshpass \
        && rm -rf /var/lib/apt/lists/*

    # Install Ansible and six
    RUN pip install ansible six

    # Configure SSH
    RUN mkdir /var/run/sshd && \
        echo 'root:password' | chpasswd && \
        sed -i 's/#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config && \
        sed -i 's/#PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
        echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config && \
        echo 'PasswordAuthentication yes' >> /etc/ssh/sshd_config

    # Create SSH directory and generate keys
    RUN mkdir -p /root/.ssh && \
        ssh-keygen -A

    # Expose SSH port
    EXPOSE 22

    # Start SSH service
    CMD ["/usr/sbin/sshd", "-D"]
    ```

    Build the Docker image:
    ```bash
    docker build -t ansible-network-config .
    ```

2. **Create a Docker Network**
    ```bash
    docker network create my-net
    ```

3. **Run Docker Containers**
    ```bash
    docker run -d --name ansible1 --network my-net --privileged -p 2224:22 ansible-network-config
    docker run -d --name ansible2 --network my-net --privileged -p 2225:22 ansible-network-config
    ```

### Creating Ansible Inventory and Playbooks

1. **Set Up Ansible Inventory**

    Create an `inventory.ini` file:
    ```ini
    [network_devices]
    ansible1 ansible_host=127.0.0.1 ansible_port=2224 ansible_user=root ansible_password=password
    ansible2 ansible_host=127.0.0.1 ansible_port=2225 ansible_user=root ansible_password=password
    ```

2. **Create Ansible Playbooks**

    Create a `configure-network.yml` file:
    ```yaml
    - name: Configure network devices
      hosts: all
      tasks:
        - name: Configure hostname
          command: hostnamectl set-hostname "{{ inventory_hostname }}"
        - name: Configure IP address
          command: ip addr add 192.168.1.{{ inventory_hostname[-1] }}/24 dev eth0
        - name: Bring up network interface
          command: ip link set eth0 up
        - name: Update /etc/hosts
          lineinfile:
            path: /etc/hosts
            line: "{{ item }}"
          with_items:
            - "192.168.1.1 ansible1"
            - "192.168.1.2 ansible2"
    ```

### Running Playbooks

1. **Run the Playbook**
    ```bash
    ansible-playbook -i inventory.ini configure-network.yml
    ```

### Configuring AWX

1. **Create a Project in AWX**

    - Navigate to the Projects section in the AWX UI.
    - Click on the "Add" button.
    - Fill in the necessary details:
        - **Name**: Ansible Playbooks
        - **Description**: A collection of Ansible playbooks for configuring network devices.
        - **Organization**: Default (or your specific organization)
        - **Source Control Type**: Git
        - **Source Control URL**: `https://github.com/Utkal20/ansible-playbooks.git`
        - **Source Control Branch/Tag/Commit**: `main` (or your specific branch)
    - Save the project.

2. **Create an Inventory in AWX**

    - Navigate to the Inventories section in the AWX UI.
    - Click on the "Add" button.
    - Fill in the necessary details:
        - **Name**: Network Devices
        - **Organization**: Default (or your specific organization)
    - Save the inventory.
    - Add hosts to the inventory:
        - **Name**: ansible1
        - **Variables**: 
          ```yaml
          ansible_host: 127.0.0.1
          ansible_port: 2224
          ansible_user: root
          ansible_password: password
          ```
        - **Name**: ansible2
        - **Variables**: 
          ```yaml
          ansible_host: 127.0.0.1
          ansible_port: 2225
          ansible_user: root
          ansible_password: password
          ```

3. **Create a Job Template in AWX**

    - Navigate to the Templates section in the AWX UI.
    - Click on the "Add" button.
    - Fill in the necessary details:
        - **Name**: Configure Network Devices
        - **Job Type**: Run
        - **Inventory**: Network Devices
        - **Project**: Ansible Playbooks
        - **Playbook**: `configure-network.yml`
    - Save the template.

4. **Run the Job**

    - Navigate to the Templates section in the AWX UI.
    - Click on the "Rocket" icon next to the "Configure Network Devices" template to launch the job.
    - Monitor the job execution and verify that the network devices are configured as expected.

## Additional Evaluation Criteria

### Functionality and Features
- **Does the software meet functional requirements?**
  - Yes, AWX meets the functional requirements for managing and executing Ansible playbooks.
- **Does it provide the necessary features needed?**
  - Yes, it provides necessary features like job scheduling, real-time output, and integration with Git.

### Extensibility
- **Can it be extended as a plugin/top-up or customized to fit specific needs?**
  - Yes, AWX can be customized and extended as needed.
- **Can we change the code with customization needs and rebuild?**
  - Yes, AWX is open-source, and its code can be modified and rebuilt.

### License
- **What type of open source license does the software use?**
  - AWX uses the Apache License 2.0.
- **Is the license compatible with intended use?**
  - Yes, it is compatible with most use cases.
- **Are there any restrictions that might affect the project?**
  - No significant restrictions.
- **Is it redistributable?**
  - Yes, it is redistributable under the Apache License 2.0.

### Community and Support
- **Is there an active community of users and developers?**
  - Yes, AWX has an active community.
- **How responsive is the community to issues and questions?**
  - The community is generally responsive to issues and questions.
- **Are there regular updates and maintenance releases?**
  - Yes, there are regular updates and maintenance releases.

### Documentation
- **Is the documentation comprehensive and easy to understand?**
  - Yes, AWX has comprehensive documentation.
- **Are there tutorials, FAQs, and other resources to help you get started?**
  - Yes, there are tutorials, FAQs, and other resources available.

### Compatibility
- **Is the software compatible with existing systems and technologies?**
  - Yes, AWX is compatible with a wide range of systems and technologies.
- **Does it support the operating systems and platforms we use?**
  - Yes, AWX supports major operating systems and platforms.
- **Can it integrate with other tools and services we rely on?**
  - Yes, AWX integrates well with Git, LDAP, and other tools commonly used in IT environments.

### Security
- **Is the software regularly audited for security vulnerabilities?**
  - Yes, AWX is regularly updated with security patches.
- **Are there mechanisms in place for reporting and addressing security issues?**
  - Yes, mechanisms are in place for reporting and addressing security issues via GitHub.
- **Does it have a good track record regarding security?**
  - Yes, AWX has a good track record regarding security.

### Performance and Scalability
- **Does the software perform well under expected load conditions?**
  - Yes, AWX performs well under expected load conditions.
- **Can it scale to meet your future needs?**
  - Yes, AWX can scale to meet future needs by adding more nodes to the Kubernetes cluster.

### Cost
- **Are there any associated costs (e.g., support, training)?**
  - While the software itself is open source, there may be associated costs for support and training.
- **Are there any hidden costs in terms of time or effort required for implementation and maintenance?**
  - Implementation and maintenance may require significant time and effort, especially for customization and scaling.

### Maturity and Stability
- **How long has the software been in development?**
  - AWX has been in development for several years.
- **Is it considered stable and reliable by its user base?**
  - Yes, it is considered stable and reliable by its user base.
- **Are there many known bugs or unresolved issues?**
  - There are some known bugs, but the community actively works on resolving issues.

### Ease of Use
- **Is the software user-friendly?**
  - Yes, AWX is user-friendly with a web-based interface.
- **Does it have a steep learning curve?**
  - The learning curve is moderate, primarily due to the complexity of Ansible playbooks.
- **Are there user interfaces and tools to simplify its use?**
  - Yes, AWX provides a web-based interface and various tools to simplify its use.

### Governance and Roadmap
- **How is the software governed?**
  - AWX is governed by the Ansible community and Red Hat.
- **Is there a clear roadmap for future development?**
  - Yes, there is a clear roadmap for future development.
- **Are the future plans aligned with long-term needs?**
  - Yes, the future plans are aligned with long-term needs.

### Feedback and Reviews
- **What do other users and organizations say about the software?**
  - Feedback from users and organizations is generally positive.
- **Are there case studies or testimonials from similar projects or industries?**
  - Yes, there are case studies and testimonials available.

## Conclusion

AWX provides a powerful and flexible platform for automating IT processes using Ansible playbooks. While it may not support direct UI-based configuration of network devices, it offers extensive capabilities for managing and executing playbooks, making it a valuable tool for network automation.
