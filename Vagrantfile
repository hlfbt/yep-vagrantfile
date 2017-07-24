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
require 'ipaddr'

root_path = File.expand_path(File.dirname(__FILE__))
basename = File.basename(root_path)

# Load Vagrantconfig.yml configuration, or assume defaults if not present
defaults = {
  'vagrant' => {
    'name'      => "default",
    'box'       => "boxcutter/ubuntu1604"  # The ubuntu/xenial64 box has a bunch of problems with interfaces etc.
  },
  'network' => {
    'dhcp'      => true
  },
  'vm' => {
    'hostname'  => "local.dev",
    'memory'    => 1024,
    'swap'      => 1024
  },
  'synced_folders' => {
    'folders'   => []
  },
  'provision' => {
    'packages'  => "",
    'commands'  => {
      'always'  => [],
      'once'    => []
    }
  },
  'mysql' => {
    'rootpass'  => "test123",
    'databases' => []
  },
  'apache' => {
    'webroot'   => "web"
  },
  'php' => {
    'memory_limit' => "256M"
  },
  'misc' => {
    'upmessage' => ""
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

$debug_yep = (ARGV.length > 1 and ARGV[1] == '--debug-yep')
$yep_said_intro = false
def yep_puts(msg, only_on_cmd = "up")
  only_on_cmd = only_on_cmd.to_s
  if only_on_cmd.length == 0 or ARGV[0] == only_on_cmd or $debug_yep
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

yaml_local_config = Hash.new
begin
  yaml_local_config = YAML.load_file "#{root_path}/Vagrantconfig.local.yml" if File.exist?("#{root_path}/Vagrantconfig.local.yml")
rescue Exception => e
  fail Vagrant::Errors::VagrantError.new, "The Vagrantconfig.local.yml config file contains syntax errors:\n" + e.message
end

if yaml_config.is_a?(Hash)
  config = merge_recursively(defaults, yaml_config)
  config = merge_recursively(config, yaml_local_config) if yaml_local_config.is_a?(Hash)
  flat_config = flatten_hash(config).inject({}){|memo,(k,v)| memo[k.to_sym] = v; memo}
  if ARGV[0] == "up"
    puts "YAML Easy Provision enabled"
    puts "YAML Easy Provision couldn't find a 'Vagrantconfig.yml' or 'Vagrantconfig.local.yml' file, so it assumed default values!" if !(File.exist?("#{root_path}/Vagrantconfig.yml") || File.exist?("#{root_path}/Vagrantconfig.local.yml"))
  end
else
  yaml_error = yaml_config.to_s
  fail Vagrant::Errors::VagrantError.new, "The Vagrantconfig.yml config file contains errors, vagrant will exit.\nThis is what the parser returned:\n#{yaml_error}"
end

is_provisioned = File.exist?("#{root_path}/.vagrant/machines/#{config['vagrant']['name']}/virtualbox/action_provision")

if $debug_yep
  require 'pp'
  puts "root_path: #{root_path}"
  puts "basename: #{basename}"
  puts "is_provisioned: #{is_provisioned}"
  puts "\nVagrantconfig.yml:"
  if File.exist?("#{root_path}/Vagrantconfig.yml") then pp yaml_config else puts "File not found." end
  puts "\nVagrantconfig.local.yml:"
  if File.exist?("#{root_path}/Vagrantconfig.local.yml") then pp yaml_local_config else puts "File not found." end
  puts "\nconfig:"
  pp config
  puts "\nflat_config:"
  pp flat_config
  puts ""
end

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |vagrant|
  vagrant.vm.hostname = config['vm']['hostname']
  vagrant.vm.box = config['vagrant']['box']
  vagrant.vm.box_check_update = false
  vagrant.vm.define config['vagrant']['name']

  # Parse and set the correct network settings
  use_dhcp = config['network']['dhcp']
  use_dhcp = (use_dhcp == "true" or (!!use_dhcp == use_dhcp and use_dhcp == true))
  network_config = { :type => "static" }
  network_config[:type] = "dhcp" if use_dhcp
  if !(config['network']['ip'].nil?)
    network_ip = IPAddr.new(config['network']['ip'])
    ip_addresses = network_ip.to_range().first(4)
    if ip_addresses.size == 1
      if use_dhcp
        network_config[:adapter_ip] = ip_addresses[0].to_s
        network_config[:dhcp_ip] = ip_addresses[0].succ.to_s
        network_config[:dhcp_lower] = ip_addresses[0].succ.succ.to_s
        network_config[:dhcp_upper] = ip_addresses[0].mask(24).to_range().last(2)[0].to_s
      else
        network_config[:ip] = ip_addresses[0].to_s
      end
    elsif ip_addresses.size == 4
    main_ip = config['network']['ip'].split("/")[0]
      main_ip = ip_addresses[1].to_s if main_ip == ip_addresses[0].to_s
      network_config[:netmask] = network_ip.inspect.split("/")[1].chomp(">")
      if use_dhcp
        network_config[:adapter_ip] = main_ip
        network_config[:dhcp_ip] = ((ip_addresses[1].to_s == main_ip) ? ip_addresses[2].to_s : ip_addresses[1].to_s)
        network_config[:dhcp_lower] = ((ip_addresses[2].to_s == network_config[:dhcp_ip]) ? ip_addresses[3].to_s : ip_addresses[2].to_s)
        network_config[:dhcp_upper] = network_ip.to_range().last(2)[0].to_s
      else
        network_config[:ip] = main_ip
      end
    elsif ip_addresses.size != 0
      fail Vagrant::Errors::VagrantError.new, "The network.ip parameter only accepts either a single IP, or a range of at least 4 IP addresses."
    end
  elsif !use_dhcp
    fail Vagrant::Errors:VagrantError.new, "Network set to static but no IP address was provided."
  end
  vagrant.vm.network "private_network", **network_config

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

  synced_folders_default = {}
  synced_folders_default[:type] = config['synced_folders']['type'] if !(config['synced_folders']['type'].nil?)
  vagrant.vm.synced_folder "./", "/vagrant", **synced_folders_default
  synced_folders_msg = "Will mount folder#{'s' if config['synced_folders']['folders'].size > 1}: "
  parse_folder = lambda do |key, val = nil, args: {}|
    host = nil
    guest = nil
    if val.nil?
      if key.kind_of?(Hash)
        host = key['host'] if !(key['host'].nil?)
        guest = key['guest'] if !(key['guest'].nil?)
        args[:type] = key['type'] if !(key['type'].nil?)
      elsif key.kind_of?(Array)
        host = key[0] if key.size > 0
        guest = key[1] if key.size > 1
        args[:type] = key[2] if key.size > 2
      end
    elsif val.kind_of?(String)
      host = key
      guest = val
    else
      return parse_folder.call val.unshift(key)
    end
    [host, guest, args]
  end
  add_folder = lambda do |key, val|
    (host, guest, args) = parse_folder.call(key, val, :args => synced_folders_default)
    if host.nil? or guest.nil?
      fail Vagrant::Errors::VagrantError.new, "Invalid folder parameter: " + ((val.nil?) ? "#{key}" : "#{key}: #{val}") + ". Expects either an array (['/path/on/host', '/path/on/guest', 'optional mount type']), a set ({'host': '/path/on/host', 'guest': '/path/on/guest', 'type': 'optional mount type'}) or a key / value pair ('/path/on/host': '/path/on/guest')."
    end
    synced_folders_msg += "#{host} => #{guest} [" + args.map{|k,v| "#{k}=#{v}"}.join(', ') + "], "
    vagrant.vm.synced_folder host, guest, **args
  end
  if config['synced_folders']['folders'].kind_of?(Hash)
    config['synced_folders']['folders'].each { |key, val| add_folder.call key, val }
  elsif config['synced_folders']['folders'].kind_of?(Array)
    config['synced_folders']['folders'].each { |key| add_folder.call key, nil }
  end
  yep_puts synced_folders_msg.chomp(", ") if !(config['synced_folders']['folders'].empty?)

  # Shut the "stdin: is not a tty" and "mesg: ttyname failed : Inappropriate ioctl for device" warnings up
  vagrant.vm.provision :shell, privileged: true, inline: "(grep -q 'mesg n' /root/.profile && sed -i '/mesg n/d' /root/.profile && echo 'Ignore the previous stdin/mesg error, fixing this now...') || exit 0;"

  config['vm']['swap'] = config['vm']['swap'].to_i
  if config['vm']['swap'] > 0
    vagrant.vm.provision "Swap creation", type: "shell", run: "always", privileged: true, inline: <<-SWAPEOF.gsub(/^ {6}/, '')
      { \
        { \
          [ ! -e /var/swap.1 -o "$(du -m /var/swap.1 | grep -o '^[0-9]*' | head -c-2)" != "$(head -c-2 <<< "#{config['vm']['swap']}")" ] \
          && { \
            echo -n "Creating swap of size #{config['vm']['swap']}M" \
            && /bin/dd if=/dev/zero of=/var/swap.1 bs=1M count=#{config['vm']['swap']} status=none \
            && chmod 600 /var/swap.1 \
            && /sbin/mkswap /var/swap.1 >/dev/null; \
          } \
          || true; \
        } \
        && /sbin/swapoff -a > /dev/null \
        && /sbin/swapon /var/swap.1 >/dev/null \
        || echo "error creating/enabling swap" >&2; \
      } \
      || true;
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
    once_cmds = config['provision']['commands']['once']
    if !(once_cmds.nil?) and (once_cmds.kind_of?(Array) or once_cmds.kind_of?(Hash)) and once_cmds.length > 0
      is_array = once_cmds.kind_of?(Array)
      if !is_provisioned
        if is_array
          yep_puts "Will run #{once_cmds.length} custom commands on provision"
        else
          yep_puts "Will run custom commands on provision: #{once_cmds.keys.join(', ')}"
        end
      end
      once_cmds.each_with_index do |(name,cmd),idx|
        if is_array
          cmd = name
          prov_name = "shell_\##{idx}"
        else
          prov_name = name
        end
        cmd = cmd % flat_config
        echo_cmd = "echo '" + cmd.gsub("'", "'\"'\"'").split("\n").join("'\necho '") + "'"
        vagrant.vm.provision prov_name, type: "shell", keep_color: true, inline: <<-CMDEOF.gsub(/^ {10}/, '')
          echo 'Running provisioning command(s):'
          #{echo_cmd}
          #{cmd}
        CMDEOF
      end
    end

    always_cmds = config['provision']['commands']['always']
    if !(always_cmds.nil?) and (always_cmds.kind_of?(Array) or always_cmds.kind_of?(Hash)) and always_cmds.length > 0
      is_array = always_cmds.kind_of?(Array)
      if is_array
        yep_puts "Will run #{always_cmds.length} custom commands on boot"
      else
        yep_puts "Will run custom commands on boot: #{always_cmds.keys.join(', ')}"
      end
      always_cmds.each_with_index do |(name,cmd),idx|
        if is_array
          cmd = name
          prov_name = "shell_\##{idx}"
        else
          prov_name = name
        end
        cmd = cmd % flat_config
        echo_cmd = "echo '" + cmd.gsub("'", "'\"'\"'").split("\n").join("'\necho '") + "'"
        vagrant.vm.provision prov_name, type: "shell", keep_color: true, run: "always", inline: <<-CMDEOF.gsub(/^ {10}/, '')
          echo 'Running command(s):'
          #{echo_cmd}
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

  if !(config['misc']['upmessage'].nil?) and config['misc']['upmessage'].to_s.length > 0
    upmessage = config['misc']['upmessage'].to_s % flat_config
    vagrant.vm.post_up_message << "\n#{upmessage}"
  end

  exit 0 if $debug_yep
end
