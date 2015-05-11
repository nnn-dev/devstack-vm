# -*- mode: ruby -*-
# vi: set ft=ruby :

# <<< Configuration >>>
# processor arch bits (32/64)
config_arch=32
# virtual machine memory
vm_memory=3000
# openstack version
os_version='stable/juno'
# components to install
os_components=[
    'neutron',
    'swift',
#    'security_groups',
#    'tempest',
#    'trove',
#    'ceilometer',
#    'designate',
#    'designate_agent',
#    'designate_sink',
#    'neutron_lbaas',
#    'neutron_vpn',
#    'neutron_fwaas',
#    'ceph',
]
# vnc keymap
vnc_keymap='en-us'
# <<< Configuration>>>

#using proxy?
#http_proxy=http://user:pass@password
#http_proxys=http://user:pass@password
#no_proxy=127.0.0.1,localhost,192.168.27.100
#vagrant plugin install vagrant-proxyconf

Vagrant.configure("2") do |config|


 if Vagrant.has_plugin?("vagrant-proxyconf")
	config.proxy.http    = ENV['http_proxy'] if  ENV['http_proxy']!=nil
	config.proxy.https    = ENV['https_proxy'] if  ENV['https_proxy']!=nil
    config.proxy.no_proxy = ENV['no_proxy'] if ENV['no_proxy']!=nil
  end
    
    # if you have a vagrant-cachier plugin, use it for apt caches
    if Vagrant.has_plugin?("vagrant-cachier")
      config.cache.scope = :box
    end

	if config_arch==32
		cirros_arch='i386'
	else
		cirros_arch='x86_64'
	end

	components_params=''
	os_components.each do |val|
	 components_params+=" -e #{val}=True"
	end
	
    config.vm.box = "ubuntu/trusty#{config_arch}"

    # Ubuntu has a "bug" (https://github.com/mitchellh/vagrant/issues/1673)
    config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"  

	config.vm.hostname = 'devstack'
	
	# eth1, this will be the endpoint
    config.vm.network :private_network, ip: "192.168.27.100"
    # eth2, this will be the OpenStack "public" network, use DevStack default
    config.vm.network :private_network, ip: "172.24.4.225", :netmask => "255.255.255.224", :auto_config => false
    
	config.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", vm_memory]
       	# eth2 must be in promiscuous mode for floating IPs to be accessible
       	vb.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
    end
	config.vm.provision :shell, inline: <<-SCRIPT
	 if [ ! -f /vagrant/devstack.yaml ]; then
		echo 'error synchronized folder /vagrant missing files!' >&2
		exit 255
	 fi
	 DEBIAN_FRONTEND=noninteractive apt-get -qqy install software-properties-common
	 DEBIAN_FRONTEND=noninteractive apt-add-repository ppa:ansible/ansible
	 DEBIAN_FRONTEND=noninteractive apt-get -qqy update
	 DEBIAN_FRONTEND=noninteractive apt-get -qqy install ansible
	 
	
     ansible-playbook /vagrant/devstack.yaml -v -e 'vnc_keymap=#{vnc_keymap}' -e 'os_version=#{os_version}' -e 'cirros_arch=#{cirros_arch}' #{components_params}
     [ "$?" -ne '0' ] && exit $?
	 chown -R vagrant /home/vagrant/.
	 cd devstack
	 if type -p screen > /dev/null && sudo -u vagrant env HOME=/home/vagrant screen -ls | egrep -q "[0-9]\.$SCREEN_NAME"; then
	  echo 'unstack current openstack'
	  sudo -u vagrant env HOME=/home/vagrant ./unstack.sh
	 fi
	 sudo -u vagrant env HOME=/home/vagrant ./stack.sh
	 [ "$?" -ne '0' ] && exit $?
	 ovs-vsctl add-port br-ex eth2-he 
	 ansible-playbook /vagrant/horizon-workaround.yaml -v
	 echo 'OFFLINE=True'>>/home/vagrant/devstack/local.conf
	SCRIPT
    
end