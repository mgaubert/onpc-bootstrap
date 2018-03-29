OpenNext Private Cloud Basic Model
##################################
:date: 2018-01-13
:tags: openstack, ansible, opennext
:category: \*openstack, \*nix


About this repository
---------------------
This repositry defines the an OpenNext deployment model for a basic configuration
in all-in-one or multi-node.

Deployment Process for all-in-one deployment
--------------------------------------------

You have to become root to execute the following steps.

Clone the ONPC Basic Model repo

.. code-block:: bash

    cd /opt
    git clone git@github.com:opennext-io/onpc-basic-model.git

Copy the env.d files and global configuration variables into place

.. code-block:: bash

    cd /opt/onpc-basic-model
    cp ./etc/openstack_deploy/env.d/* /etc/openstack_deploy/env.d/
    cp ./etc/openstack_deploy/conf.d/monitoring.aio.yml /etc/openstack_deploy/conf.d/monitoring.yml
    cp ./etc/openstack_deploy/user_onpc_variables.yml /etc/openstack_deploy
    cp ./etc/openstack_deploy/user_onpc_secrets.yml /etc/openstack_deploy

Generate the password values

.. code-block:: bash

    /opt/openstack-ansible/scripts/pw-token-gen.py --file /etc/openstack_deploy/user_onpc_secrets.yml

Import the ansible role dependencies

.. code-block:: bash
    
    cd /opt/openstack-ansible
    openstack-ansible ./tests/get-ansible-role-requirements.yml -i ./tests/test-inventory.ini \
        -e role_file=/opt/onpc-basic-model/ansible_role_requirements.yml

Clone the ONPC Monitoring repo

.. code-block:: bash

    cd /opt
    git clone https://github.com/opennext-io/onpc-monitoring.git

Update the inventory

.. code-block:: bash

    export ANSIBLE_INVENTORY=/opt/openstack-ansible/playbooks/inventory/dynamic_inventory.py
    /opt/openstack-ansible/playbooks/inventory/dynamic_inventory.py --config /etc/openstack_deploy

Create the containers

.. code-block:: bash

    openstack-ansible lxc-containers-create.yml -e container_group=monitoring_containers

Create the monitoring user and install various python dependencies

.. code-block:: bash

    openstack-ansible /opt/onpc-monitoring/playbook_setup.yml

If you are running HAProxy for load balacing you need run the following playbook as well to enable
the monitoring services backend and frontend. If HAproxy is already installed for the OpenStack services
you also need to rerun the HAProxy playbook to enable the HAProxy stats.

.. code-block:: bash

    openstack-ansible playbook_haproxy.yml
    cd /opt/openstack-ansible/playbooks
    openstack-ansible haproxy-install.yml


Install InfluxDB and InfluxDB Relay

.. code-block:: bash

    cd /opt/onpc-monitoring
    openstack-ansible playbook_influxdb.yml
    openstack-ansible playbook_influxdb_relay.yml

Install Telegraf

If you wish to install telegraf and point it at a specific target, or list of targets, set the ``telegraf_influxdb_targets``
variable in the ``user_onpc_variables.yml`` file as a list containing all targets that telegraf should ship metrics to.

.. code-block:: bash

    openstack-ansible playbook_telegraf.yml --forks 50

Install Grafana

If you're proxy'ing grafana you will need to provide the full ``root_path``
when you run the playbook add the following ``-e grafana_url='https://cloud.something/grafana/'``

.. code-block:: bash

    openstack-ansible playbook_grafana.yml

Once that last playbook is completed you will have a functioning InfluxDB, Telegraf, and Grafana metric collection system
active and collecting metrics. Grafana will need some setup, however functional dashboards have been provided in the
``grafana-dashboards`` directory.

Install Kapacitor

.. code-block:: bash

   openstack-ansible playbook-kapacitor.yml


OpenStack Swift PRoxy Server Dashboard
--------------------------------------

Once the telegraf daemon is installed onto each host, the Swift
proxy-server can be instructed to forward statsd metrics to telegraf.
The following configuration enabled the metric generation and need to
be added to the ``user_variables.yml``:

.. code-block:: yaml

    swift_proxy_server_conf_overrides:
      DEFAULT:
        log_statsd_default_sample_rate: 10
        log_statsd_metric_prefix: "{{ inventory_hostname }}.swift"
        log_statsd_host: localhost
        log_statsd_port: 8125


Rewrite the swift proxy server configuration with :

.. code-block:: bash

     cd /opt/openstack-ansible/playbooks
     openstack-ansible os-swift-setup.yml --tags swift-config --forks 2
