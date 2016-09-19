# ----------
# YAML Easy Provision™ powered Vagrantfile
# ----------
#
# Author: Alexander Schulz (alexander@codekunst.com)
#
# Description: The YEP™ system allows for easy peasy box and provisioning configuration
#              with the help of a pretty, easy to read YAML configuration file!

# ATTENTION: please install the hostmanager plugin using the following command:
# vagrant plugin install vagrant-hostmanager
# Also recommended for no-pain vboxguestadditions management:
# vagrant plugin install vagrant-vbguest

require 'yaml'

# Load Vagrantconfig.yml configuration, or assume defaults if not present
defaults = {
  'vagrant' => {
    'box'      => "boxcutter/ubuntu1604"  # The ubuntu/xenial64 box has a bunch of problems with interfaces etc.
  },
  'vm' => {
    'hostname' => "local.dev",
    'memory'   => 1024,
    'swap'     => 1024,
    'nfs'      => true
  },
  'provision' => {
    'packages' => "",
    'commands' => {
      'always' => "",
      'once'   => ""
    }
  },
  'mysql' => {
    'rootpass' => "test123"
  },
  'apache' => {
    'webroot'  => "web"
  }
}
yaml_config = Hash.new
begin
  yaml_config = YAML.load_file "Vagrantconfig.yml" if File.exist?("Vagrantconfig.yml")
rescue Exception => e
  fail Vagrant::Errors::VagrantError.new, "The Vagrantconfig.yml config file contains syntax errors:\n" + e.message
end
if yaml_config.is_a?(Hash)
  params = defaults.merge(yaml_config)
  # Fix the first level if not properly set in config
  params.each do |key, val|
    params[key] = defaults[key] if val.nil?
  end
  if ARGV[0] == "up"
    puts "YAML Easy Provision™ enabled"
    puts "YAML Easy Provision™ couldn't find a 'Vagrantconfig.yml' file, so it assumed default values!" if !(File.exist?("Vagrantconfig.yml"))
  end
  yep_said_intro = false
else
  yaml_error = yaml_config.to_s
  fail Vagrant::Errors::VagrantError.new, "The Vagrantconfig.yml config file contains errors, vagrant will exit.\nThis is what the parser returned:\n#{yaml_error}"
end
def yep_puts(msg, only_on_cmd = "up")
  only_on_cmd = only_on_cmd.to_s
  if only_on_cmd.length == 0 or ARGV[0] == only_on_cmd
    if !(defined? yep_said_intro) or (!!yep_said_intro) == false
      yep_said_intro = true
      puts "YAML Easy Provision™ says:"
    end
    puts ' - ' + msg
  end
end


VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.hostname = params['vm']['hostname']
  config.vm.box = params['vagrant']['box']
  config.vm.box_check_update = false
  config.vm.network "private_network", type: "dhcp"

  # Automatically manage /etc/hosts on hosts and guests
  if Vagrant.has_plugin? 'vagrant-hostmanager'
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true
    # Custom IP resolver to fetch the box's IP address, since hostmanager doesn't support DHCP
    # Cache box addresses for when ssh is not available
    cached_addresses = {}
    config.hostmanager.ip_resolver = proc do |vm, resolving_vm|
      if cached_addresses[vm.name].nil?
        if hostname = (vm.ssh_info && vm.ssh_info[:host])
          vm.communicate.execute("hostname -I") do |type, data|
            cached_addresses[vm.name] = data.split()[1]
          end
        end
      end
      cached_addresses[vm.name]
    end
  else
   fail Vagrant::Errors::VagrantError.new, "You don't have the vagrant-hostmanager plugin installed, please install it with this command:\nvagrant plugin install vagrant-hostmanager"
  end

  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", params['vm']['memory']]
  end

  use_nfs = params['vm']['nfs']
  if use_nfs == "true" or (!!use_nfs == use_nfs and use_nfs == true)
    config.vm.synced_folder "./", "/vagrant", nfs: true
  else
    config.vm.synced_folder "./", "/vagrant"
  end

  # Shut the "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device" warnings up
  config.vm.provision :shell, privileged: true, inline: "(grep -q 'mesg n' /root/.profile && sed -i '/mesg n/d' /root/.profile && echo 'Ignore the previous stdin/mesg error, fixing this now...') || exit 0;"

  if params['vm']['swap'].to_i > 0
    config.vm.provision :shell, run: "always", privileged: true, inline: <<-SWAPEOF.gsub(/^ {6}/, '')
      echo -n "Creating swap"
      [ ! -e /var/swap.1 ] && /bin/dd if=/dev/zero of=/var/swap.1 bs=1M count=#{params['vm']['swap']} status=none && chmod 600 /var/swap.1 && /sbin/mkswap /var/swap.1 >/dev/null \
        && /sbin/swapon /var/swap.1 >/dev/null \
        || echo "error creating swap" >&2
    SWAPEOF
  end

  config.vm.provision "shell", inline: <<-PROVISIONEOF.gsub(/^ {4}/, '')
    echo "Configuring packages"
    debconf-set-selections <<< 'mysql-server mysql-server/root_password password #{params['mysql']['rootpass']}'
    debconf-set-selections <<< 'mysql-server mysql-server/root_password_again password #{params['mysql']['rootpass']}'
    debconf-set-selections <<< 'phpmyadmin phpmyadmin/dbconfig-install boolean true'
    debconf-set-selections <<< 'phpmyadmin phpmyadmin/app-password-confirm password'
    debconf-set-selections <<< 'phpmyadmin phpmyadmin/mysql/admin-pass password #{params['mysql']['rootpass']}'
    debconf-set-selections <<< 'phpmyadmin phpmyadmin/mysql/app-pass password'
    debconf-set-selections <<< 'phpmyadmin phpmyadmin/reconfigure-webserver multiselect apache2'

    echo "Installing base packages (this may take a while...)"
    apt-get update > /dev/null && apt-get -y install apache2 libapache2-mod-php php php-cli php-curl php-gd php-zip php-mcrypt php-mysql mysql-server phpmyadmin graphicsmagick unzip git > /dev/null

    echo "Linking web-root"
    rm -rf /var/www/html && ln -s /vagrant/#{params['apache']['webroot']} /var/www/html

    echo "Configuring php"
    # This assumes php7 obviously, which is default on ubuntu 16.04+
    sed -i 's/memory_limit = .*/memory_limit = 256M/' /etc/php/7.0/apache2/php.ini
    sed -i 's/upload_max_filesize = .*/upload_max_filesize = 10M/' /etc/php/7.0/apache2/php.ini
    sed -i 's/post_max_size = .*/post_max_size = 10M/' /etc/php/7.0/apache2/php.ini
    sed -i 's/max_execution_time = .*/max_execution_time = 240/' /etc/php/7.0/apache2/php.ini

    echo "Configuring apache"
    sed -e '14i\<Directory /var/www/html>\\n		AllowOverride All\\n	</Directory>\\n' /etc/apache2/sites-available/000-default.conf > /etc/apache2/sites-available/001-vagrant.conf
    a2enmod rewrite > /dev/null
    a2dissite 000-default > /dev/null
    a2ensite 001-vagrant > /dev/null
    service apache2 restart > /dev/null
  PROVISIONEOF

  is_provisioned = File.exist?('.vagrant/machines/default/virtualbox/action_provision')
  if !is_provisioned and !(params['provision']['packages'].nil?) and params['provision']['packages'].length > 0
    package_count = params['provision']['packages'].split(" ").length
    yep_puts "Will install #{package_count} extra packages"
    config.vm.provision "shell", inline: <<-PACKAGESEOF.gsub(/ {6}/, '')
      echo "Installing extra packages"
      apt-get -y install #{params['provision']['packages']} > /dev/null
    PACKAGESEOF
  end

  if !(params['provision']['commands'].nil?) and params['provision']['commands'].kind_of?(Hash)
    if !is_provisioned and !(params['provision']['commands']['once'].nil?) and params['provision']['commands']['once'].kind_of?(Array)
      once_count = params['provision']['commands']['once'].length
      yep_puts "Will run #{once_count} custom commands on provision"
      params['provision']['commands']['once'].each do |cmd|
        config.vm.provision "shell", inline: "set -x; #{cmd}"
      end
    end
    if !(params['provision']['commands']['always'].nil?) and params['provision']['commands']['always'].kind_of?(Array)
      always_count = params['provision']['commands']['always'].length
      yep_puts "Will run #{always_count} custom commands on boot"
      params['provision']['commands']['always'].each do |cmd|
        config.vm.provision "shell", run: "always", inline: "set -x; #{cmd}"
      end
    end
  end

  config.vm.post_up_message = <<-UPMSGEOF.gsub(/^ {4}/, '')
    Your machine is available as http://#{params['vm']['hostname']}/
    PHPMyAdmin URL: http://#{params['vm']['hostname']}/phpmyadmin
    MySQL root password: #{params['mysql']['rootpass']}
  UPMSGEOF

end
