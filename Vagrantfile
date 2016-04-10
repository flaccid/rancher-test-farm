# -*- mode: ruby -*-
# vi: set ft=ruby :

require_relative 'vagrant_rancheros_guest_plugin.rb'

# To enable rsync folder share change to false
rsync_folder_disabled = true
# number_of_nodes = 1
vb_gui = false
vb_memory = 1024
vb_cpus = 1
expose_rancher_ui = 8080
rancher_private_ip = '172.19.8.8'
rancher_host_private_ip = '172.19.8.9'
rancher_url = "http://#{rancher_private_ip}:8080"

# compose files
docker_compose_file = 'https://raw.githubusercontent.com/flaccid/countdown_example/master/docker-compose.yml'
rancher_compose_file = 'https://raw.githubusercontent.com/flaccid/countdown_example/master/rancher-compose.yml'

rancher_compose_tarball = 'https://github.com/rancher/rancher-compose/releases/download/v0.2.5/rancher-compose-linux-amd64-v0.2.5.tar.gz'

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  config.vm.box = 'rancherio/rancheros'
  config.vm.box_version = '>=0.4.1'

  config.vm.define 'rancher-server', primary: true do |rancher|
    rancher.vm.hostname = 'rancher-server'
    rancher.vm.provider :virtualbox do |vb|
      vb.gui = vb_gui
      vb.memory = vb_memory
      vb.cpus = vb_cpus
    end
    rancher.vm.network :private_network, ip: rancher_private_ip
    rancher.vm.network 'forwarded_port',
                       guest: 8080,
                       host: expose_rancher_ui,
                       auto_correct: true
    rancher.vm.provision :shell,
                         inline: "! docker ps | grep 'rancher/server' && "\
                          'docker run -d -p 8080:8080 rancher/server:latest',
                         privileged: true
  end

  config.vm.define 'rancher-host' do |rancher_host|
    rancher_host.vm.hostname = 'rancher-host'
    rancher_host.vm.network :private_network, ip: rancher_host_private_ip
    rancher_host.vm.network 'forwarded_port', guest: 8000, host: 8000
    rancher_host.vm.provision :shell,
                              inline: 'docker run -e WAIT=true '\
                               '-v /var/run/docker.sock:/var/run/docker.sock '\
                               "rancher/agent:latest #{rancher_url}",
                              privileged: true
  end

  config.vm.define 'rancher-client' do |rancher_client|
    rancher_client.vm.hostname = 'rancher-client'
    rancher_client.vm.box = 'coreos-alpha'
    rancher_client.vm.box_url = 'http://alpha.release.core-os.net/amd64-usr/current/coreos_production_vagrant.json'

    # install rancher-compose
    rancher_client.vm.provision :shell,
                                inline: 'cd /tmp && '\
                                  "wget -q #{rancher_compose_tarball} && "\
                                  'tar zxvf ./rancher-compose-*.tar.gz && '\
                                  'cp -v /tmp/rancher-compose-v*/rancher-compose /tmp/'

    # download the compose files
    rancher_client.vm.provision :shell,
                                inline: 'mkdir -p /tmp/composition && '\
                                  "wget -q #{docker_compose_file} -O /tmp/composition/docker-compose.yml && "\
                                  "wget -q #{rancher_compose_file} -O /tmp/composition/rancher-compose.yml"

    rancher_up = <<SCRIPT
api_keys=$(curl --silent -X POST -H 'Accept: application/json' -H 'Content-Type: application/json' -d '{}' #{rancher_url}/v1/projects/1a5/apikeys)
access_key=$(echo "$api_keys" | jq -r '.publicValue')
secret_key=$(echo "$api_keys" | jq -r '.secretValue')

cat <<EOF> /tmp/rancher-up.sh
#!/bin/sh -e

/tmp/rancher-compose \
  -f /tmp/composition/docker-compose.yml \
  -r /tmp/composition/rancher-compose.yml \
  --url #{rancher_url} \
  --access-key "$access_key" --secret-key "$secret_key" up -d
EOF

chmod +x /tmp/rancher-up.sh
sleep 3 && /tmp/rancher-up.sh
SCRIPT

    # get api keys and run rancher-compose
    rancher_client.vm.provision :shell, inline: rancher_up

    lb_check = <<SCRIPT
echo 'waiting for site to be available'
timeout 100 bash -c -- 'while true; do printf "." && curl --output /dev/null --silent --head --fail http://172.19.8.9:8000/ && break || sleep 1; done'
echo 'checking content'
curl --silent http://172.19.8.9:8000 | grep "Rio 2016 Olympic Games"
SCRIPT

    rancher_client.vm.provision :shell, inline: lb_check
  end

  # Disabling compression as OS X has an ancient version of rsync installed
  # Add -z or remove rsync__args below if you have a newer version of rsync
  rsync_args = ['--verbose', '--archive', '--delete', '--copy-links']
  config.vm.synced_folder '.', '/opt/rancher',
                          type: 'rsync',
                          rsync__exclude: '.git/',
                          rsync__args: rsync_args,
                          disabled: rsync_folder_disabled
end
