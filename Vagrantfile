# ----------
# YAML Easy Provision powered Vagrantfile
# ----------
#
# Author: Alexander Schulz (alexander@codekunst.com)
#
# Description: The YEP system allows for easy peasy box and provisioning configuration
#              with the help of a pretty, easy to read YAML configuration file!

# ATTENTION: please install the hostmanager plugin using the following command:
# vagrant plugin install vagrant-hostmanager
# Also recommended for no-pain vboxguestadditions management:
# vagrant plugin install vagrant-vbguest

require 'yaml'

root_path = File.expand_path(File.dirname(__FILE__))
basename = File.basename(root_path)

# Load Vagrantconfig.yml configuration, or assume defaults if not present
defaults = {
  'vagrant' => {
    'name'     => basename,
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
    'rootpass' => "test123",
    'databases' => []
  },
  'apache' => {
    'webroot'  => "web"
  },
  'php' => {
    'memory_limit' => "256M"
  }
}

def flatten_hash(hash, prefix = '', flat = nil)
  flat = Hash.new if flat.nil?
  if hash.kind_of?(Hash)
    hash.each do |key, val|
      if key.is_a?(Integer)
        pfx = "#{prefix}[#{key}]"
      else
        pfx = prefix.length > 0 ? "#{prefix}.#{key}" : key
      end
      flat = flatten_hash(val, pfx, flat)
    end
  elsif hash.kind_of?(Array)
    hash.each_with_index do |val, idx|
      pfx = "#{prefix}[#{idx}]"
      flat = flatten_hash(val, pfx, flat)
    end
  else
    prefix = "hash" if prefix.length < 1
    flat[prefix] = hash
  end

  return flat
end

def merge_recursively(a, b)
  a.merge(b) do |key, a_item, b_item|
    if a_item.is_a?(Hash) and b_item.is_a?(Hash)
      merge_recursively(a_item, b_item)
    else
      b_item.nil? ? a_item : b_item
    end
  end
end

$yep_said_intro = false
def yep_puts(msg, only_on_cmd = "up")
  only_on_cmd = only_on_cmd.to_s
  if only_on_cmd.length == 0 or ARGV[0] == only_on_cmd
    if !(defined? $yep_said_intro) or (!!$yep_said_intro) == false
      $yep_said_intro = true
      puts "YAML Easy Provision says:"
    end
    puts ' - ' + msg
  end
end


yaml_config = Hash.new
begin
  yaml_config = YAML.load_file "#{root_path}/Vagrantconfig.yml" if File.exist?("#{root_path}/Vagrantconfig.yml")
rescue Exception => e
  fail Vagrant::Errors::VagrantError.new, "The Vagrantconfig.yml config file contains syntax errors:\n" + e.message
end

if yaml_config.is_a?(Hash)
  config = merge_recursively(defaults, yaml_config)
  flat_config = flatten_hash(config).inject({}){|memo,(k,v)| memo[k.to_sym] = v; memo}
  if ARGV[0] == "up"
    puts "YAML Easy Provision enabled"
    puts "YAML Easy Provision couldn't find a 'Vagrantconfig.yml' file, so it assumed default values!" if !(File.exist?("#{root_path}/Vagrantconfig.yml"))
  end
else
  yaml_error = yaml_config.to_s
  fail Vagrant::Errors::VagrantError.new, "The Vagrantconfig.yml config file contains errors, vagrant will exit.\nThis is what the parser returned:\n#{yaml_error}"
end

is_provisioned = File.exist?("#{root_path}/.vagrant/machines/#{config['vagrant']['name']}/virtualbox/action_provision")


VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |vagrant|
  vagrant.vm.hostname = config['vm']['hostname']
  vagrant.vm.box = config['vagrant']['box']
  vagrant.vm.box_check_update = false
  vagrant.vm.define config['vagrant']['name']
  vagrant.vm.network "private_network", type: "dhcp"

  # Automatically manage /etc/hosts on hosts and guests
  if Vagrant.has_plugin? 'vagrant-hostmanager'
    vagrant.hostmanager.enabled = true
    vagrant.hostmanager.manage_host = true
    # Custom IP resolver to fetch the box's IP address, since hostmanager doesn't support DHCP
    # Cache box addresses for when ssh is not available
    cached_addresses = {}
    vagrant.hostmanager.ip_resolver = proc do |vm, resolving_vm|
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

  vagrant.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", config['vm']['memory']]
  end

  use_nfs = config['vm']['nfs']
  if use_nfs == "true" or (!!use_nfs == use_nfs and use_nfs == true)
    vagrant.vm.synced_folder "./", "/vagrant", nfs: true
  else
    vagrant.vm.synced_folder "./", "/vagrant"
  end

  # Shut the "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device" warnings up
  vagrant.vm.provision :shell, privileged: true, inline: "(grep -q 'mesg n' /root/.profile && sed -i '/mesg n/d' /root/.profile && echo 'Ignore the previous stdin/mesg error, fixing this now...') || exit 0;"

  if config['vm']['swap'].to_i > 0
    vagrant.vm.provision "Swap creation", type: "shell", run: "always", privileged: true, inline: <<-SWAPEOF.gsub(/^ {6}/, '')
      echo -n "Creating swap"
      [ ! -e /var/swap.1 ] && { /bin/dd if=/dev/zero of=/var/swap.1 bs=1M count=#{config['vm']['swap']} status=none && chmod 600 /var/swap.1 && /sbin/mkswap /var/swap.1 >/dev/null \
        && /sbin/swapon /var/swap.1 >/dev/null \
        || echo "error creating swap" >&2; } \
      || true
    SWAPEOF
  end

  vagrant.vm.provision "Base provisioning", type: "shell", inline: <<-PROVISIONEOF.gsub(/^ {4}/, '')
    echo "Configuring packages"
    debconf-set-selections <<< 'mysql-server mysql-server/root_password password #{config['mysql']['rootpass']}'
    debconf-set-selections <<< 'mysql-server mysql-server/root_password_again password #{config['mysql']['rootpass']}'
    debconf-set-selections <<< 'phpmyadmin phpmyadmin/dbconfig-install boolean true'
    debconf-set-selections <<< 'phpmyadmin phpmyadmin/app-password-confirm password'
    debconf-set-selections <<< 'phpmyadmin phpmyadmin/mysql/admin-pass password #{config['mysql']['rootpass']}'
    debconf-set-selections <<< 'phpmyadmin phpmyadmin/mysql/app-pass password'
    debconf-set-selections <<< 'phpmyadmin phpmyadmin/reconfigure-webserver multiselect apache2'

    echo "Installing base packages (this may take a while...)"
    apt-get update > /dev/null && apt-get -y install apache2 libapache2-mod-php php php-cli php-curl php-gd php-zip php-mcrypt php-mysql mysql-server phpmyadmin graphicsmagick unzip git > /dev/null

    echo "Linking web-root"
    rm -rf /var/www/html && ln -s /vagrant/#{config['apache']['webroot']} /var/www/html

    echo "Configuring php"
    # This assumes php7 obviously, which is default on ubuntu 16.04+
    sed -i 's/memory_limit = .*/memory_limit = #{config['php']['memory_limit']}/' /etc/php/7.0/apache2/php.ini
    sed -i 's/upload_max_filesize = .*/upload_max_filesize = 10M/' /etc/php/7.0/apache2/php.ini
    sed -i 's/upload_max_filesize = .*/upload_max_filesize = 10M/' /etc/php/7.0/cli/php.ini
    sed -i 's/post_max_size = .*/post_max_size = 10M/' /etc/php/7.0/apache2/php.ini
    sed -i 's/post_max_size = .*/post_max_size = 10M/' /etc/php/7.0/cli/php.ini
    sed -i 's/max_execution_time = .*/max_execution_time = 240/' /etc/php/7.0/apache2/php.ini
    sed -i 's/max_execution_time = .*/max_execution_time = 240/' /etc/php/7.0/cli/php.ini

    echo "Configuring apache"
    sed -e '14i\<Directory /var/www/html>\\n		AllowOverride All\\n	</Directory>\\n' /etc/apache2/sites-available/000-default.conf > /etc/apache2/sites-available/001-vagrant.conf
    a2enmod rewrite > /dev/null
    a2dissite 000-default > /dev/null
    a2ensite 001-vagrant > /dev/null
    service apache2 restart > /dev/null

    # automatically move into /vagrant when ssh'ing into the box
    echo 'cd /vagrant' >> /home/vagrant/.bashrc
  PROVISIONEOF

  if !(config['provision']['packages'].nil?) and config['provision']['packages'].length > 0
    package_count = config['provision']['packages'].split(" ").length
    yep_puts "Will install #{package_count} extra packages" if !is_provisioned
    vagrant.vm.provision "Extra packages", type: "shell", inline: <<-PACKAGESEOF.gsub(/ {6}/, '')
      echo 'Installing extra packages: #{config['provision']['packages']}'
      apt-get -y install #{config['provision']['packages']} > /dev/null
    PACKAGESEOF
  end

  if !(config['mysql']['databases'].nil?) and config['mysql']['databases'].kind_of?(Array)
    yep_puts "Will create database(s) %s" % config['mysql']['databases'].join(", ") if !is_provisioned
    config['mysql']['databases'].each do |db|
      vagrant.vm.provision "Creating Database: #{db}", type: "shell", keep_color: true, inline: "mysql -uroot -p'#{config['mysql']['rootpass']}' -e 'CREATE DATABASE IF NOT EXISTS `#{db}`;' >/dev/null"
    end
  end

  if !(config['provision']['commands'].nil?) and config['provision']['commands'].kind_of?(Hash)
    if !(config['provision']['commands']['once'].nil?) and config['provision']['commands']['once'].kind_of?(Array)
      once_count = config['provision']['commands']['once'].length
      yep_puts "Will run #{once_count} custom commands on provision" if !is_provisioned
      config['provision']['commands']['once'].each do |cmd|
        cmd = cmd % flat_config
        echoCmd = cmd.gsub("'", "'\"'\"'")
        vagrant.vm.provision "Shell command: #{cmd}", type: "shell", keep_color: true, inline: <<-CMDEOF.gsub(/^ {10}/, '')
          echo 'Running provisioning command: #{echoCmd}'
          #{cmd}
        CMDEOF
      end
    end
    if !(config['provision']['commands']['always'].nil?) and config['provision']['commands']['always'].kind_of?(Array)
      always_count = config['provision']['commands']['always'].length
      yep_puts "Will run #{always_count} custom commands on boot"
      config['provision']['commands']['always'].each_with_index do |cmd, idx|
        cmd = cmd % flat_config
        echoCmd = cmd.gsub("'", "'\"'\"'")
        vagrant.vm.provision "Shell command \##{idx}", type: "shell", keep_color: true, run: "always", inline: <<-CMDEOF.gsub(/^ {10}/, '')
          echo 'Running command: #{echoCmd}'
          #{cmd}
        CMDEOF
      end
    end
  end

  vagrant.vm.post_up_message = <<-UPMSGEOF.gsub(/^ {4}/, '')
    Your machine is available as http://#{config['vm']['hostname']}/
    PHPMyAdmin URL: http://#{config['vm']['hostname']}/phpmyadmin
    MySQL root password: #{config['mysql']['rootpass']}
  UPMSGEOF

end
