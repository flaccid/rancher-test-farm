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

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  config.vm.box   = 'rancherio/rancheros'
  config.vm.box_version = '>=0.3.3'

  config.vm.define 'rancher-server', primary: true do |rancher|
    config.vm.provider :virtualbox do |vb|
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
    rancher_host.vm.network :private_network, ip: rancher_host_private_ip

    rancher_host.vm.provision :shell,
                              inline: 'docker run -e WAIT=true '\
                               '-v /var/run/docker.sock:/var/run/docker.sock '\
                               "rancher/agent:latest #{rancher_url}",
                              privileged: true
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
