Vagrant.configure("2") do |config|

  config.vm.define "redismaster" do |node|
    node.vm.hostname = "redismaster"
    node.vm.box = "bento/ubuntu-18.04"
    node.vm.network "private_network", ip: "15.16.17.16"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "redismaster"
      vb.gui = false
      vb.memory = "512"
    end

    node.vm.provision "shell", path: "provision/redismaster.sh", privileged: false
  end

  (1..2).each do |i|
    config.vm.define "redisslave#{i}" do |node|
      node.vm.hostname = "redisslave#{i}"
      node.vm.box = "bento/ubuntu-18.04"
      node.vm.network "private_network", ip: "15.16.17.#{16+i}"

      node.vm.provider "virtualbox" do |vb|
        vb.name = "redisslave#{i}"
        vb.gui = false
        vb.memory = "512"
      end

      node.vm.provision "shell", path: "provision/redisslave#{i}.sh", privileged: false
    end
  end

  (1..2).each do |i|
    config.vm.define "wordpress#{i}" do |node|
      node.vm.hostname = "wordpress#{i}"
      node.vm.box = "bento/ubuntu-18.04"
      node.vm.network "private_network", ip: "15.16.17.#{18+i}"

      node.vm.provider "virtualbox" do |vb|
        vb.name = "wordpress#{i}"
        vb.gui = false
        vb.memory = "512"
      end

      node.vm.provision "shell", path: "provision/wordpress.sh", privileged: false
    end
  end

end