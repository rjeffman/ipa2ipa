IPA-to-IPA Migration Test Environment
=====================================

This repository provides some scripts, playbooks and configuration files to automate the creation of an environment to test IPA-to-IPA migration using in a local machine. The environment created contains two IPA servers each hosting its own realm.


Pod creation
------------

The environment created is isolated in a container pod, with its own network. The pod name is `ipa2ipa`, and a pod named `pod_ipa2ipa` holds the containers. A network `ipa2ipa_ipanet` is created for the subnet `192.168.73.0/24`. The containers IPv4 address are static. The container names for the IPA servers are `origin`, with IP address 192.168.73.100, and `target`, with IP address 192.168.73.200.

This configuration can be changed in the [compose.yml](compose.yml) file.

Note that no port is open to external hosts, and the tests will run under this somewhat isolated environment.

Before starting the environment, set the servers hostnames:

```
export origin_hostname="origin.old.ipa.local"
export target_hostname="target.new.ipa.local"
```

Start the environment with `podman-compose up -d`.

Whenever you change the hostnames or change the compose file, you should add `--build` to the `podman-compose` command to rebuild the environment.


Deploying IPA
-------------

The base images for the containers have FreeIPA files installed, but FreeIPA is not actually deployed. To deploy FreeIPA it is recommended that [ansible-freeipa](https://github.com/freeipa/ansible-freeipa) is used.

If you don't have Ansible installed, use a Python virtual environment and install  `ansible-core`:

```
python3 -m venv /tmp/ipa2ipa
. /tmp/ipa2ipa/bin/activate
pip install ansible-core
```

> Note: If you plan to use the older CentOS 8 Stream image, install ansible-core up to version 2.16 (`pip install "ansible-core<2.17"`).

Install the required Ansible collections:

```
ansible-galaxy collections install -r requirements.yml
```

An Ansible [inventory file](inventory.yml) and a [playbook to deploy the servers](playbooks/install-server.yml) are provided. To customize the depolyment, see ansible-freeipa `ipaserver` role documentation for available variables and modify the inventory file.

Deploy the servers with `ansible-playbook -i inventory.yml playbooks/install-server.yml`. If you want to install one server at a time, use `--limit <container_name>`.


Simple test migration
---------------------

To test IPA-to-IPA migration, first create some objects in the origin server:

```
ansible-playbook -i inventory.yml playbooks/users_present.yml
```

Access the target server container:

```
podman exec -it target bash
```

Add the origin IP address to the `hosts` file:

```
echo "192.168.73.100   origin.old.ipa.local" >> /etc/hosts
```

Obtain an IPA administrator TGT ticket (in this example the password is `SomeADMINpassword`):

```
kinit admin
Password for admin@TARGET.LOCAL:
```

Execute the IPA-to-IPA migration tool:

```
ipa-migrate prod-mode origin.old.ipa.local
```

You will be prompted for the origin DM password. In this example, it is `SomeDMpassword`. Proceed with the process by answering "yes", after prompted.

After some time you see the migration report, and you are able to query users that were added the the origin server, for example:

```
ipa user-show testuser_42
```

