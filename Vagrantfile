# -*- mode: ruby -*-
# vi: set ft=ruby :

$ubuntu_mirrors = <<SCRIPT
sed -i 's@http://archive.ubuntu.com/ubuntu@mirror://mirrors.ubuntu.com/mirrors.txt@' /etc/apt/sources.list
SCRIPT

$install_consul = <<SCRIPT
echo "====> Updating Packages"
export DEBIAN_FRONTEND=noninteractive
# -qq is pointless, it doesn't work :S
apt-get update 2>&1 > /dev/null
echo "====> Installing Packages"
apt-get install -qq -y --no-install-recommends unzip
echo "====> Installing consul"
useradd --system consul
mkdir -p /var/lib/consul
chown consul:consul /var/lib/consul
cd /usr/local/bin
wget --quiet https://dl.bintray.com/mitchellh/consul/0.5.2_linux_amd64.zip
unzip *.zip
rm *.zip
cd /usr/local/share
mkdir -p consul/web-ui
cd consul/web-ui
wget --quiet https://dl.bintray.com/mitchellh/consul/0.5.2_web_ui.zip
unzip *.zip
rm *.zip
mv dist/* .
rm -rf dist
SCRIPT

$configure_consul_service = <<SCRIPT
cp -v /vagrant/systemd/consul.service /etc/systemd/system/consul.service
systemctl enable consul.service
systemctl start consul.service
SCRIPT

$install_docker = <<SCRIPT
echo "====> Installing docker"
wget -qO- https://experimental.docker.com/ | sh
usermod -aG docker vagrant
SCRIPT

$install_docker_compose = <<SCRIPT
echo "====> Installing docker compose"
curl -sSL https://github.com/docker/compose/releases/download/1.3.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
SCRIPT

$configure_docker_service = <<SCRIPT
cp -v /vagrant/systemd/docker.service /etc/systemd/system/docker.service
systemctl daemon-reload
systemctl enable docker.socket
systemctl enable docker.service
systemctl start docker.socket
SCRIPT

unless Vagrant.has_plugin?("vagrant-reload")
  raise 'vagrant-reload is not installed! (vagrant plugin install vagrant-reload)'
end


# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|

  num_nodes = (ENV['MULTIHOST_NODES'] || 3).to_i

  base_ip = "10.100.0."
  multihost_ips = num_nodes.times.collect { |n| base_ip + "#{n+10}" }

  num_nodes.times do |n|

    host_index = n + 1
    host_name = "node-#{host_index}"
    host_ip = multihost_ips[n]

    config.vm.define host_name do |host|

      host.vm.box = "ubuntu/vivid64"

      # In case you run into "SSL certificate problem: unable to get local issuer certificate"
      # and you do not have the time to address the issue properly, uncomment the following line
      # host.vm.box_download_insecure = true

      host.vm.network :private_network, ip: "#{host_ip}"

      host.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      end

      # This fails in ubuntu/vivid for now (https://github.com/mitchellh/vagrant/issues/5673)
      #host.vm.hostname = host_name
      host.vm.provision :shell, inline: "hostnamectl set-hostname #{host_name}"

      host.vm.provision :shell, inline: $ubuntu_mirrors

      host_config = {
        bind_addr: host_ip,
      }

      if host_index == 1
        host_config[:bootstrap_expect] = 1
      else
        host_config[:retry_join] = [ multihost_ips[0] ]
      end

      host.vm.provision :shell, inline: <<-SHELL
        mkdir -p /etc/consul.d
        cp -v /vagrant/consul.d/*.json /etc/consul.d
        echo '#{host_config.to_json}' > /etc/consul.d/10-node-config.json
      SHELL

      host.vm.provision :shell, inline: $install_consul
      host.vm.provision :shell, inline: $configure_consul_service

      # Installing docker
      host.vm.provision :shell, inline: $install_docker
      host.vm.provision :shell, inline: $install_docker_compose

      docker_opts="-H tcp://0.0.0.0:2375 --kv-store consul:localhost:8500 --default-network overlay:multihost --label com.docker.network.driver.overlay.bind_interface=eth1"

      if host_index != 1
        docker_opts += " --label com.docker.network.driver.overlay.neighbor_ip=#{multihost_ips[0]}"
      end

      host.vm.provision :shell, inline: <<-SHELL
        echo 'DOCKER_OPTS="#{docker_opts}"' >> /etc/default/docker
      SHELL

      # We need to restart the machine
      host.vm.provision :reload

      host.vm.provision :shell, inline: $configure_docker_service

      # Docker swarm
      host.vm.provision :shell, inline: <<-SHELL
        docker rm -f swarm-join || true
        docker rm -f swarm-manage || true
        docker run -d --restart=always --name swarm-join   --net=host swarm:latest join   --addr #{host_ip}:2375 consul://localhost:8500/
        docker run -d --restart=always --name swarm-manage --net=host swarm:latest manage --addr #{host_ip}:3375 -H tcp://0.0.0.0:3375 --replication  --strategy spread consul://localhost:8500/
      SHELL

    end
  end

end
