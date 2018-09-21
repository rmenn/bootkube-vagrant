# -*- mode: ruby -*-
# # vi: set ft=ruby :

require 'fileutils'
require 'open-uri'
require 'tempfile'
require 'yaml'

Vagrant.require_version ">= 1.9.7"

required_plugins = %w(vagrant-ignition)

plugins_to_install = required_plugins.select { |plugin| not Vagrant.has_plugin? plugin }
if not plugins_to_install.empty?
  puts "Installing plugins: #{plugins_to_install.join(' ')}"
  if system "vagrant plugin install #{plugins_to_install.join(' ')}"
    exec "vagrant #{ARGV.join(' ')}"
  else
    abort "Installation of one or more plugins has failed. Aborting."
  end
end

$update_channel = "stable"
$controller_vm_memory = 1280
$worker_vm_memory = 2048

if $worker_vm_memory < 1536
  puts "Workers need at least 1536 MB of memory"
end

BOOTSTRAP_USERDATA_PATH = File.expand_path("bootkube.yaml")
WORKER_USERDATA_PATH = File.expand_path("worker.yaml")

Vagrant.configure("2") do |config|
  config.ssh.insert_key = false

  config.vm.box = "coreos-#{$update_channel}"
  config.vm.box_url = "https://#{$update_channel}.release.core-os.net/amd64-usr/current/coreos_production_vagrant_virtualbox.json"

  config.vm.provider :virtualbox do |v|
    v.check_guest_additions = false
    v.functional_vboxsf     = false
  end

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end


  config.vm.provider :virtualbox do |vb|
    vb.cpus = 1
    vb.gui = false
  end

  config.vm.define vm_name = "w1" do |worker|
    worker.vm.hostname = vm_name

    worker.vm.provider :virtualbox do |vb|
      vb.memory = $worker_vm_memory
      vb.customize ["modifyvm", :id, "--paravirtprovider", "default"]
      vb.customize ["modifyvm", :id, "--natdnspassdomain1", "on"]
      vb.customize ["setextradata", :id, "VBoxInternal/Devices/virtio-net/0/LUN#0/Config/HostResolverMappings/c1/HostIP", "172.17.4.100"]
      vb.customize ["setextradata", :id, "VBoxInternal/Devices/virtio-net/0/LUN#0/Config/HostResolverMappings/c1/HostName", "controller-1.coreos.local"]
      vb.customize ["modifyvm", :id, "--ioapic", "on"]
      vb.customize ["modifyvm", :id, "--cpus", "2"]
    end

    workerIP = "172.17.4.200"
    worker.vm.network :private_network, ip: workerIP

    if ARGV[0] == 'up'
      worker.vm.provision :file, :source => WORKER_USERDATA_PATH, :destination => "/tmp/vagrantfile-user-data"
      worker.vm.provision :shell, :inline => "mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/", :privileged => true
      worker.vm.provision :file, :source => "cluster/auth/kubeconfig-kubelet", :destination => "/tmp/kubeconfig-kubelet"
      worker.vm.provision :file, :source => "cluster/tls/ca.crt", :destination => "/tmp/ca.crt"
      worker.vm.provision :shell, :inline => "mkdir -p /etc/kubernetes && cp /tmp/kubeconfig-kubelet /etc/kubernetes/kubeconfig && cp /tmp/ca.crt /etc/kubernetes/ca.crt && chown -R core:core /etc/kubernetes/", :privileged => true
    end
  end

  config.vm.define vm_name = "controller-1" do |controller|
    controller.vm.hostname = vm_name

    controller.vm.provider :virtualbox do |vb|
      vb.memory = $controller_vm_memory
      vb.customize ["modifyvm", :id, "--natdnspassdomain1", "on"]
      vb.customize ["setextradata", :id, "VBoxInternal/Devices/virtio-net/0/LUN#0/Config/HostResolverMappings/c1/HostIP", "172.17.4.100"]
      vb.customize ["setextradata", :id, "VBoxInternal/Devices/virtio-net/0/LUN#0/Config/HostResolverMappings/c1/HostName", "controller-1.coreos.local"]
    end

    controllerIP = "172.17.4.100"
    controller.vm.network :private_network, ip: controllerIP

    if ARGV[0] == 'up'
      controller.vm.provision :file, :source => BOOTSTRAP_USERDATA_PATH, :destination => "/tmp/vagrantfile-user-data"
      controller.vm.provision :shell, :inline => "mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/", :privileged => true
      controller.vm.provision :shell, :inline => "mkdir -p /etc/kubernetes/ && mkdir -p /etc/etcd/tls/etcd", :privileged => true
      controller.vm.provision :file, :source => "cluster.zip", :destination => "/tmp/cluster.zip"
      controller.vm.provision :shell, :inline => "mv /tmp/cluster.zip /home/core/", :privileged => true
      controller.vm.provision :shell, :inline => "unzip /home/core/cluster.zip && rm /home/core/cluster.zip && sudo chown -R core:core /home/core/cluster", :privileged => true
      controller.vm.provision :shell, :inline => "cp /home/core/cluster/auth/kubeconfig-kubelet /etc/kubernetes/kubeconfig && cp /home/core/cluster/tls/ca.crt /etc/kubernetes/ca.crt && chown -R core:core /etc/kubernetes/", :privileged => true
      controller.vm.provision :shell, :inline => "cp /home/core/cluster/tls/etcd-client* /etc/etcd/tls && sudo cp /home/core/cluster/tls/etcd/* /etc/etcd/tls/etcd && sudo chown -R etcd:etcd /etc/etcd && sudo chmod 500 -R /etc/etcd", :privileged => true
    end
  end
end
