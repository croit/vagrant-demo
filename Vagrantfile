# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

	config.vm.box = 'debian/stretch64'
	config.vm.box_version = '=9.7.0'

	# we don't use synced folders so disable the default to not require rsync binary on Windows
	config.vm.synced_folder ".", "/vagrant", disabled: true
	
	# the croit controller VM
	config.vm.define :croit, primary: true do |config|
		# bind to 127.0.0.1 (as opposed to 0.0.0.0) to fix Vagrant 1.9.3 on Windows 10
		config.vm.network "forwarded_port", guest: 8080, host: 8080, host_ip: "127.0.0.1"
		config.vm.network "forwarded_port", guest: 443, host: 8443, host_ip: "127.0.0.1"

		config.vm.network "private_network", ip: "192.168.0.2", libvirt__network_name: "croit_pxe", :libvirt__dhcp_enabled => false, virtualbox__intnet: "croit_pxe"
		config.vm.provider "virtualbox" do |vb|
			vb.memory = '1536'
			vb.cpus = '4' # makes the initial setup much faster
		end
		config.vm.provision "shell", privileged: true, inline: %Q{
			apt -y update
			apt -y remove docker docker-engine docker.io containerd runc
			apt -y install apt-transport-https ca-certificates curl gnupg2 software-properties-common
			curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
			add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
			apt -y update
			apt -y install docker-ce docker-ce-cli containerd.io 
			echo "Downloading and starting Docker image (1.5 GB)."
			echo "This will take several minutes..."
			docker create --name croit-data croit/croit:latest
			docker run --net=host --volumes-from croit-data --name croit --restart=always -d croit/croit:#{ENV['CROIT_TAG'] || 'latest'}
		}
		config.vm.provision "shell", privileged: true, run: "always", inline: <<-SHELL
			systemctl start docker
			echo "Trying to update Docker image, this might take a few minutes"
			docker stop croit
			docker ps -a | grep "croit:nightly" && (docker pull croit/croit:nightly; docker rm -f croit; docker run --net=host --volumes-from croit-data --name croit --restart=always -d croit/croit:latest)
			docker ps -a | grep "croit:latest" && (docker pull croit/croit:latest; docker rm -f croit; docker run --net=host --volumes-from croit-data --name croit --restart=always -d croit/croit:latest)
		SHELL
	end

	# 5 identical test VMs
	(1..5).each do |i|
		config.vm.define :"ceph#{i}", autostart: false do |config|
			config.vm.box = 'q2p/empty'
			config.vm.box_version = '=0.0.1'
			config.vm.provider :virtualbox do |vb|
				vb.gui = 'true'
				vb.memory = '2048'
				vb.cpus = '2'
				vb.customize [
					'modifyvm', :id,
					# the image enables usb2.0 by default which requires the oracle extensions
					# we don't want that dependency
					'--usbehci', 'off',
					# virtualbox complains otherwise
					'--vram', '16'
				]
				# network setup
				vb.customize [
					'modifyvm', :id,
					'--nic1', 'intnet',
					'--intnet1', 'croit_pxe',
					# make sure it only boots via network
					'--boot1', 'net', '--boot2', 'none', '--boot3', 'none', '--boot4', 'none',
				]
				# no, we don't want an annoying prompt
				vb.customize [ 'setextradata', :id, 'GUI/FirstRun', 'no' ]
			end
			# running the disk commands multiple times fails
			# and there is no way to ignore failures in vb.customize
			unless created?("ceph#{i}")
				config.vm.provider :virtualbox do |vb|
					# we need to change the MAC on the first startup (but only on the first, otherwise it will be re-detected)
					# yes, it just re-uses the same mac by default...
					vb.customize [
						'modifyvm', :id,
						'--macaddress1', 'auto'
					]
					# delete the cd drive
					vb.customize [
						'storageattach', :id,
						'--storagectl', 'IDE',
						'--medium', 'none',
						'--port', 1,
						'--device', 0
					]
					# delete the default (empty) disk
					vb.customize [
						'storageattach', :id,
						'--storagectl', 'SATA',
						'--medium', 'none',
						'--port', 0,
						'--device', 0
					]
					# create our disks, suggested usage:
					# disk 1: mon (2 GB)
					# disk 2: osd (10 GB)
					# disk 3: osd (10 GB)
					add_disk(vb, "./ceph#{i}-disk1.vdi", 0, 2048, 'on')
					add_disk(vb, "./ceph#{i}-disk2.vdi", 1, 10240, 'on')
					add_disk(vb, "./ceph#{i}-disk3.vdi", 2, 10240, 'off')
				end
			end
		end
	end
end

def add_disk(vb, name, port, size, ssd)
	unless File.exist?(name)
		vb.customize [
			'createmedium', 'disk',
			'--filename', name,
			'--size', size
		]
	end
	vb.customize [
		'storageattach', :id,
		'--storagectl', 'SATA',
		'--type', 'hdd',
		'--medium', name,
		'--port', port,
		'--nonrotational', ssd
	]
end

# Check if a VM was already created before
def created?(vm_name, provider='virtualbox')
	File.exist?(".vagrant/machines/#{vm_name}/#{provider}/id")
end

