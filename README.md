# Fedora Python VM Template with Cloud-Init Automation in OpenShift

This guide demonstrates how to create a Fedora-based VM template in OpenShift Virtualization with automated Python web app deployment using cloud-init. The template is designed for demonstrating Veeam Kasten K10 backup and replication capabilities.

## Template Overview

The template creates a minimal Fedora VM with:
- 1 CPU core
- 512MB RAM 
- 10GB storage
- Python web application auto-deployment
- Default login: fedora/auto-generated
- Base Image: Fedora Cloud 39

## Key Components

### 1. Base Template Configuration
This section defines the basic template metadata and type:
```yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: fedora-python-tiny
  namespace: openshift
  labels:
    flavor.template.kubevirt.io/tiny: 'true'    # Defines template size category
    template.kubevirt.io/type: vm               # Specifies this is a VM template
```

### 2. VM Specifications and Image Source
This section configures the VM resources and base image:
```yaml
spec:
  dataVolumeTemplates:                          # Defines how to get the base OS image
    - apiVersion: cdi.kubevirt.io/v1beta1
      kind: DataVolume
      metadata:
        name: '${NAME}'
      spec:
        source:
          http:
            url: https://download.fedoraproject.org/pub/fedora/linux/releases/39/Cloud/x86_64/images/Fedora-Cloud-Base-39-1.5.x86_64.qcow2
        storage:
          resources:
            requests:
              storage: 10Gi                     # Allocates 10GB storage for the VM
  domain:
    cpu:
      cores: 1                                  # Allocates 1 CPU core
      sockets: 1
      threads: 1
    memory:
      guest: 512Mi                             # Allocates 512MB RAM
```

### 3. Cloud-Init Automation
This section handles all the initial VM setup and application deployment:
```yaml
cloudInitNoCloud:
  userData: |-
    #cloud-config
    user: fedora                               # Creates the main user account
    password: ${VM_PASSWORD}                   # Sets auto-generated password
    chpasswd: { expire: False }               # Prevents password expiration
    
    # Package Installation - installs required software
    packages:
      - python3
      - python3-pip
      - git
      - cronie                                # For scheduling app startup
    
    # Deployment Script - creates installation script
    write_files:
      - path: /root/install.sh
        permissions: '0755'
        content: |
          #!/bin/bash
          cd /home/fedora
          # Clones and installs the web application
          su - fedora -c "git clone https://github.com/mritsurgeon/premier-league-scores-webapp.git"
          su - fedora -c "cd premier-league-scores-webapp && pip3 install --user -r requirements.txt"
          
          # Sets up automatic restart on boot
          su - fedora -c '(crontab -l 2>/dev/null; echo "@reboot /usr/bin/python3 /home/fedora/premier-league-scores-webapp/app.py >> /home/fedora/premier-league-scores-webapp/app.log 2>&1") | crontab -'
          
          # Starts the application
          su - fedora -c "python3 /home/fedora/premier-league-scores-webapp/app.py &"
    
    # Startup Commands - executes installation
    runcmd:
      - systemctl enable crond                 # Enables cron service
      - systemctl start crond                  # Starts cron service
      - chmod +x /root/install.sh             # Makes install script executable
      - /root/install.sh                      # Runs the installation
```

## Implementation Steps

1. Log into your OpenShift Web Console

2. Navigate to Virtualization → Templates
   - This is where VM templates are managed

3. Click "Create Template" and paste the YAML content
   - Creates the template in your cluster

4. Create a new VM from the template:
   - Go to Virtualization → Virtual Machines
   - Click "Create" → "With Template"
   - Select "fedora-python-tiny" template
   - Follow the wizard to complete VM creation

5. Access your VM:
   - Username: fedora
   - Password: auto-generated (check VM events)
   - The web app will be running automatically

Note: The template automatically downloads Fedora Cloud 39 image (~1GB). Ensure your cluster has internet access and sufficient storage.

## Troubleshooting

Check cloud-init logs inside the VM to verify the automation process:
```bash
cloud-init status                              # Shows current status
cloud-init analyze show                        # Shows detailed execution analysis
cat /var/log/cloud-init.log                   # Shows detailed logs
cat /var/log/cloud-init-output.log            # Shows command outputs
```

## License

[MIT](LICENSE)

---
⭐ Found this useful? Star this repository! ⭐

