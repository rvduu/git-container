# git-container
Run Gitea in a container configured with Podman quadlets.

These quadlets will run a pod with 2 containers, 3 volumes and 1 network:
* Gitea Pod
* Gitea network
* PostgreSQL data volume
* PostgreSQL container
* Gittea data volume
* Gittea config volume
* Gittea container

By using the Podman volumes, all data is stored in `~/.local/share/containers/storage/volumes/`. Make sure to include those in a backup.

Notes:
* The quadlets are configured to run as a non-priviledged user and will use the `gitea-rootless` container. 
* The sshd service will be reconfigured to run on a different port.
* Kernel will be reconfigured to allow userland processes to bind to priveliged ports (net.ipv4.ip_unprivileged_port_start)
* Firewall ports to be opened (or as per the 'all:vars' in the inventory file)
    * 22/tcp
    * 80/tcp
    * 443/tcp
    * 1022/tcp

## Requirements (for Fedora/RHEL and like)
* Ansible (on the host controller node running this playbook)
    * Core
    * Collections (community.general.seport, ansible.posix.firewalld, ansible.posix.sysctl)
        ```
        sudo dnf -y install ansible-core ansible
        ```
    * When using `ansible_password` to connect to the remote host using ssh
        ```
        sudo dnf -y instal sshpass
        ```
* Podman (on the host running the Gitea container)
    ```
    sudo dnf -y install podman
    ```


## Setup (see also https://docs.gitea.com/installation/install-with-docker and https://docs.gitea.com/administration/https-setup)
1. Update the `inventory` file and add your host, as an example:
    1. Use localhost
        ```
        localhost ansible_connection=local ansible_become_user=root ansible_become_pass=<password>
        ```
    1. Use remote host and specify connection details
        ```
        host1 ansible_user=user1 ansible_password=<password> ansible_become_user=root ansible_become_pass=<password>
        ```
1. Run the playbook
    ```
    ansible-playbook git-container.yml
    ```
1. Configure Gitea
    1. Point a browser to the host
        ```
        http://<host.example.com>/
        ```
    1. Accept all default settings (inclusing portnumbers 2222 for ssh and 3000 for http, the ports configure in the inventory are redirect to the mentioned ports in the Pod))
    1. Fill in the Adminstrator account details
    2. Click "Install Gitea"
1. Enable SSL
    1. Generate SSL certificates from the host where the containers run
        ```
        podman exec -it gitea /bin/bash
        cd /var/lib/gitea/custom
        gitea cert --host <host.example.com>
        ```
    1. Enable SSL by udating `/etc/gitea/app.ini` and add / update the following:
        ```
        [server]
        PROTOCOL = https
        ROOT_URL = https://<host.example.com>/
        HTTP_PORT = 3001
        CERT_FILE = cert.pem
        KEY_FILE = key.pem
        REDIRECT_OTHER_PORT = true
        PORT_TO_REDIRECT = 3000
        ```
    1. Exit the container
        ```
        exit
        ```
    1. Restart the container to enable new config
        ```
        systemctl --user restart gitea-pod
        ```
All is now set, browse to https://<host.example.com>/ and login as the admin user as configured during install.



## ToDo
* Configure CA signed SSL certificates
