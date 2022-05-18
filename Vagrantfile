# -*- mode: ruby -*-
# vi: set ft=ruby :

# Copyright 2022, Harald von Waldow and Eawag, harald.vonwaldow@eawag.ch

# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU Affero General Public License as
# published by the Free Software Foundation, either version 3 of the
# License, or (at your option) any later version.

# This program is distributed in the hope that it will be useful, but
# WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
# Affero General Public License for more details.

# You should have received a copy of the GNU Affero General Public
# License along with this program.  If not, see
# <https://www.gnu.org/licenses/>.
			

Vagrant.configure("2") do |config|

  config.vm.box = "debian/bullseye64"
  # Adapter setting is specific to the host setup, here a bridge interface ("br0").
  # Otherwise probably needs to be new-fangled "enp0s3xyz" - type interface-name. 
  config.vm.network :forwarded_port, guest: 5000, host: 5050
  config.vm.network :forwarded_port, guest: 8983, host: 8983 
  config.vm.network "private_network", ip: "192.168.33.10"
  # config.vm.network "public_network"
  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  config.vm.synced_folder "./lib", "/usr/lib/ckan", create: true, type: "nfs", mount_options: ['rw,nfsvers=4,proto=tcp'] 
  config.vm.synced_folder "./etc/ckan", "/etc/ckan", create: true, type: "nfs", mount_options: ['rw,nfsvers=4,proto=tcp']
  config.vm.synced_folder "./var", "/var/lib/ckan", create: true, type: "nfs", mount_options: ['rw,nfsvers=4,proto=tcp'] 

  config.vm.provider :libvirt do |libvirt|
    libvirt.cpus = 2
    libvirt.memory = 4096
  end

  # Enable provisioning with a shell script. Additional provisioners such as
  # Ansible, Chef, Docker, Puppet and Salt are also available. Please see the
  # documentation for more information about their specific syntax and use.
  
  config.vm.provision "shell", inline: <<-SHELL

    # Debian Packages

    apt-get update
    apt-get install -y python3-dev postgresql libpq-dev python3-pip python3-venv redis-server openjdk-17-jre-headless git-core

    # CKAN Source
    ## Virtual environment
    python3 -m venv /usr/lib/ckan/default
    source /usr/lib/ckan/default/bin/activate
    pip install --upgrade pip
   
    ## Clone from GitHub  
    mkdir -p /usr/lib/ckan/default/src
    rm -rf /usr/lib/ckan/default/src/ckan
    git config --global user.email "vagrant@devbox.local"
    git config --global user.name "vagrant shell provisioner"

    git clone https://github.com/ckan/ckan.git /usr/lib/ckan/default/src/ckan
    cd /usr/lib/ckan/default/src/ckan
    ## Checkout the target-release (tag) as a new branch
    git switch -c ckan-2.9.5 ckan-2.9.5

    ## Apply patches
    ### Bump Markdown and pip-tools so that CKAN works with python 3.9 
    git cherry-pick -X theirs 652996f8e234f6fd9760c07071d168e6c86aad6e
    ### SOLR schema update to 8.0 
    git cherry-pick -X theirs --no-commit 45b6f70d44bb862cb99f4fbb7dbb25e6639070f1
    git add .github/workflows/cypress.yml
    git cherry-pick --continue
    git commit -m "cherry-pick: SOLR schema update to 8.0"
    
    ## Installation
    pip install -r /usr/lib/ckan/default/src/ckan/requirements.txt
    pip install -r /usr/lib/ckan/default/src/ckan/dev-requirements.txt
    python /usr/lib/ckan/default/src/ckan/setup.py develop

    # Configure Postgres
    sudo -u postgres psql -c "CREATE ROLE ckan_default PASSWORD 'notasecret' LOGIN;"
    sudo -u postgres psql -c "CREATE DATABASE ckan_default OWNER ckan_default ENCODING 'utf-8';"

    # Configure CKAN
    ## Configuration File (ckan.ini)
    mkdir -p /etc/ckan/default
    rm -f /etc/ckan/default/ckan.ini
    ckan generate config /etc/ckan/default/ckan.ini

    sed -i -e 's/^debug =.*$/debug = true/' \
           -e 's_^sqlalchemy.url.*$_sqlalchemy.url = postgresql://ckan\\_default:notasecret@localhost/ckan\\_default_' \
           -e 's_^ckan.site\\_url.*$_ckan.site\\_url = http://devbox.local_' \
           -e 's_^.*solr\\_url.*$_solr\\_url = http://127.0.0.1:8983/solr/ckan0_' \
           -e 's_^#ckan\\.storage\\_path.*$_ckan\\.storage\\_path = /var/lib/ckan_' \
           -e 's_^ckan\\.devserver\\.host.*$_ckan\\.devserver\\.host = 0.0.0.0_' \
        /etc/ckan/default/ckan.ini
    
    # SOLR
    if [[ ! -h "/opt/solr" ]]; then
        cd /opt
        wget -q https://www.apache.org/dyn/closer.lua/lucene/solr/8.11.1/solr-8.11.1.tgz?action=download -O solr-8.11.1.tgz && \
        tar xzf solr-8.11.1.tgz solr-8.11.1/bin/install_solr_service.sh --strip-components=2 && \
        sudo bash ./install_solr_service.sh solr-8.11.1.tgz && \
        # rm -rf install_solr_service.sh  solr-8.11.1.tgz;
        sudo -u solr /opt/solr/bin/solr create_core -c ckan0
        sudo -u solr mv /var/solr/data/ckan0/conf/managed-schema /var/solr/data/ckan0/conf/managed-schema.bak
        # Copy instead of link (as in doc)
        sudo -u solr cp /usr/lib/ckan/default/src/ckan/ckan/config/solr/schema.xml /var/solr/data/ckan0/conf/schema.xml
        sudo -u solr sed -i 's/^<config>.*/<config>\\n\\n<schemaFactory class="ClassicIndexSchemaFactory"\\/>\\n/' /var/solr/data/ckan0/conf/solrconfig.xml
    fi
    sudo systemctl restart solr

    # Copy (instead of link as in doc) to who.ini
    cp /usr/lib/ckan/default/src/ckan/who.ini /etc/ckan/default/who.ini

    # Create Postgres tables
    cd /usr/lib/ckan/default/src/ckan
    ckan -c /etc/ckan/default/ckan.ini db init

    # Datastore extension
    ## Database
    sudo -u postgres psql -c "CREATE ROLE datastore_default PASSWORD 'notasecret' LOGIN;"
    sudo -u postgres psql -c "CREATE DATABASE datastore_default OWNER ckan_default ENCODING 'utf-8';"

    ### CKAN config for datastore DB-connection
    sed -i -e 's_^.*ckan.datastore.write\\_url.*$_ckan.datastore.write\\_url = postgresql://ckan\\_default:notasecret@localhost/datastore\\_default_' \\
           -e 's_^.*ckan.datastore.read\\_url.*$_ckan.datastore.read\\_url = postgresql://datastore\\_default:notasecret@localhost/datastore\\_default_' \\
           -e 's_^ckan.plugins\\(.*\\)$_ckan.plugins\\1 datastore_' /etc/ckan/default/ckan.ini

    ### Database permissions
    ckan -c /etc/ckan/default/ckan.ini datastore set-permissions | sudo -u postgres psql --set ON_ERROR_STOP=1

    ## Run the dev server
    ckan -c /etc/ckan/default/ckan.ini run &

  SHELL
end
