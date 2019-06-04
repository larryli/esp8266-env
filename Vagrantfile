# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'
require 'fileutils'

dir = File.dirname(File.expand_path(__FILE__))

config = {
  local: "#{dir}/vagrant.yml",
  example: "#{dir}/vagrant.example.yml"
}

# copy config from example if local config not exists
FileUtils.cp config[:example], config[:local] unless File.exist?(config[:local])
# read config
options = YAML.load_file config[:local]

class VagrantPlugins::ProviderVirtualBox::Config < Vagrant.plugin("2", :config)
  def update_customizations(customizations)
    @customizations = customizations
  end
end

class VagrantPlugins::ProviderVirtualBox::Action::Customize
  alias_method :original_call, :call
  def call(env)
    machine = env[:machine]
    config = machine.provider_config
    driver = machine.provider.driver
    uuid = driver.instance_eval { @uuid }
    if uuid != nil
      lines = driver.execute('showvminfo', uuid, '--machinereadable', retryable: true).split("\n")
      filters = {}
      lines.each do |line|
        if matcher = /^USBFilterVendorId(\d+)="(.+?)"$/.match(line)
          id = matcher[1].to_i
          vendor_id = matcher[2].to_s
          filters[id] ||= {}
          filters[id][:vendor_id] = vendor_id
        elsif matcher = /^USBFilterProductId(\d+)="(.+?)"$/.match(line)
          id = matcher[1].to_i
          product_id = matcher[2].to_s
          filters[id] ||= {}
          filters[id][:product_id] = product_id
        end
      end
      config.update_customizations(config.customizations.reject { |_, command| filter_exists(filters, command) })
    end
    original_call(env)
  end

  def filter_exists(filters, command)
    if command.size > 6 && command[0] == 'usbfilter' && command[1] == 'add'
      vendor_id = product_id = false
      i = 2
      while i < command.size - 1 do
        if command[i] == '--vendorid'
          i += 1
          vendor_id = command[i]
        elsif command[i] == '--productid'
          i += 1
          product_id = command[i]
        end
        i += 1
      end
      if vendor_id != false && product_id != false
        filters.each do |_, filter|
          if filter[:vendor_id] == vendor_id && filter[:product_id] == product_id
            return true
          end
        end
      end
    end
    return false
  end
end

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  config.vm.box_check_update = options['box_check_update']

  # machine name (for vagrant console)
  config.vm.define options['machine_name']

  # machine name (for guest machine console)
  config.vm.hostname = options['machine_name']

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  config.vm.provider "virtualbox" do |vb|
    # machine cpus count
    vb.cpus = options['cpus']
    # machine memory size
    vb.memory = options['memory']
    # machine name (for VirtualBox UI)
    vb.name = options['machine_name']
    # Enable USB
    vb.customize ['modifyvm', :id, '--usb', 'on']
    # Please use your esp device id, run `VBoxManage.exe list usbhost` list devices
    options['usbfilters'].each do |f|
      vb.customize ['usbfilter', 'add', '0', '--target', :id, '--name', f['name'], '--vendorid', f['vendorid'], '--productid', f['productid']]
    end
    # Disable log
    vb.customize ['modifyvm', :id, '--uartmode1', 'disconnected']
  end
  
  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  config.vm.provision "shell", env: {'APT_SOURCE' => options['apt_source']}, inline: <<-SHELL
    if [ ! -z $APT_SOURCE ]; then
      sed -i 's/archive.ubuntu.com/'$APT_SOURCE'/g' /etc/apt/sources.list
      sed -i 's/security.ubuntu.com/'$APT_SOURCE'/g' /etc/apt/sources.list
    fi
    apt-get -qq update && apt-get -qq upgrade -y
    apt-get -qq install -y linux-image-extra-virtual
    apt-get -qq install -y git wget libncurses-dev flex bison gperf python python-pip python-setuptools python-serial cmake ninja-build ccache
    if [ ! -d /opt/local/espressif/ ]; then
      mkdir -p /opt/local/espressif/
    fi
    GCC_VERSION='xtensa-lx106-elf-gcc (crosstool-NG crosstool-ng-1.22.0-92-g8facf4c) 5.2.0'
    gccFile='/opt/local/espressif/xtensa-lx106-elf/bin/xtensa-lx106-elf-gcc'
    if [ ! -f $gccFile ] || [ "`$gccFile --version | head -n 1`" != "$GCC_VERSION" ]; then
      wget -qO- https://dl.espressif.com/dl/xtensa-lx106-elf-linux64-1.22.0-92-g8facf4c-5.2.0.tar.gz | tar xz -C /opt/local/espressif/
    fi
    usermod -a -G dialout vagrant
  SHELL
  config.vm.provision 'shell', privileged: false, env: {'PIP_SIMPLE' => options['pip_simple']}, inline: <<-SHELL
    if [ -z $PIP_SIMPLE ]; then
      python -m pip install -q --user -r /vagrant/ESP8266_RTOS_SDK/requirements.txt
    else
      python -m pip install -i $PIP_SIMPLE -q --user -r /vagrant/ESP8266_RTOS_SDK/requirements.txt
    fi
    grep -q 'IDF_PATH' /home/vagrant/.profile || echo "export IDF_PATH=\"/vagrant/ESP8266_RTOS_SDK\"\nexport PATH=\"/opt/local/espressif/xtensa-lx106-elf/bin:/vagrant/ESP8266_RTOS_SDK/tools:${PATH}\"\ncd /vagrant" >> /home/vagrant/.profile
  SHELL
end
