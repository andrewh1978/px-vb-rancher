vb_dir = "#{ENV['HOME']}/VirtualBox\ VMs"
nodes = 3
disk_size = 20480
name = "px-test-cluster"
version = "2.1.0"
template_version = "2.1.0.0-oci"

if !File.exist?("id_rsa") or !File.exist?("id_rsa.pub")
    abort("Please create SSH keys before running vagrant up.")
end

open("hosts", "w") do |f|
  f << "192.168.99.99 master\n"
  (1..nodes).each do |n|
    f << "192.168.99.10#{n} node#{n}\n"
  end
end

Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.box_check_update = true
  config.vm.provision "shell", inline: <<-SHELL
    setenforce 0
    sed -i s/SELINUX=enforcing/SELINUX=disabled/g /etc/selinux/config
    swapoff -a
    rm -f /swapfile
    rpm -e linux-firmware
    sed -i /swap/d /etc/fstab
    sed -i s/enabled=1/enabled=0/ /etc/yum/pluginconf.d/fastestmirror.conf
    mkdir -p /root/.ssh
    cp /vagrant/hosts /etc
    cp /vagrant/id_rsa /root/.ssh
    cp /vagrant/id_rsa.pub /root/.ssh/authorized_keys
    chmod 600 /root/.ssh/id_rsa
  SHELL

  config.vm.provider "virtualbox" do |vb|
    vb.cpus = 2
  end

  config.vm.define "master" do |master|
    master.vm.hostname = "master"
    master.vm.network "private_network", ip: "192.168.99.99", virtualbox__intnet: true
    master.vm.network "forwarded_port", guest: 8080, host: 8080
    master.vm.provider :virtualbox do |vb| 
      vb.customize ["modifyvm", :id, "--name", "master"]
      vb.memory = 4096
    end
    master.vm.provision "shell", inline: <<-SHELL
      ( curl -sL https://rpm.nodesource.com/setup_10.x | bash -
        yum install -y docker nodejs
        systemctl enable docker
        systemctl start docker
        docker run --name rs --restart always -d -v /rancher/mysql:/var/lib/mysql --restart=unless-stopped -p 8080:8080 rancher/server
        npm install -g json
        curl -sSL https://github.com/rancher/cli/releases/download/v2.2.0/rancher-linux-amd64-v2.2.0.tar.gz | tar xzf - --strip-components=2 -C /usr/bin
        docker run --name etcd --restart always -d -v /etcd:/etcd-data -p 2379:2379 -p 2380:2380 quay.io/coreos/etcd:latest /usr/local/bin/etcd --advertise-client-urls http://192.168.99.99:2379 --listen-client-urls http://0.0.0.0:2379 --listen-peer-urls http://0.0.0.0:2380 --data-dir=/etcd-data --initial-advertise-peer-urls http://192.168.99.99:2380 --initial-cluster node1=http://192.168.99.99:2380 --name node1
        while : ; do
          curl -s -X PUT http://localhost:8080/v1/settings/api.host -H 'content-type: application/json' -d '{"value":"http://192.168.99.99:8080"}'
          [ $? -eq 0 ] && break
          sleep 1
        done
        while : ; do
          template_id=$(curl -s http://192.168.99.99:8080/v2-beta/projectTemplates?name=kubernetes | json data[0].id)
          [ $template_id ] && break
          sleep 1
        done
        curl -X POST -H 'Accept: application/json' -H 'Content-Type: application/json' -d '{"description":"kubernetes","name":"kubernetes","projectTemplateId":"'$template_id'","allowSystemRole":false,"members":[],"virtualMachine":false,"servicesPortRange":null}' "http://192.168.99.99:8080/v2-beta/projects" | json
        default_env_id=$(curl -s "http://192.168.99.99:8080/v2-beta/project?name=Default" | json data[0].id)
        while : ; do
          curl -s -X POST -H 'Content-Type: application/json' -H 'accept: application/json' -d '{"type":"registrationToken"}' "http://192.168.99.99:8080/v2-beta/projects/$default_env_id/registrationtoken"
          [ $? -eq 0 ] && break
        done
        curl -X POST -H 'Accept: application/json' -H 'Content-Type: application/json' http://192.168.99.99:8080/v2-beta/projects/$default_env_id/stack -d@- <<\EOF | json
{"system":true,"type":"stack","name":"portworx","startOnCreate":true,"environment":{"INSTALL_SCALE":"global","CLUSTER_ID":"#{name}","KVDB":"etcd://192.168.99.99:2379","USE_DISKS":"-a","EXTRA_OPTS":""},"dockerCompose":"version: '2'\\nservices:\\n  portworx:\\n    image: portworx/oci-monitor:#{version}\\n    container_name: px-oci-mon\\n    privileged: true\\n    labels:\\n      io.rancher.container.dns: 'true'\\n      {{- if eq .Values.INSTALL_SCALE \\"global\\" }}\\n      io.rancher.scheduler.global: 'true'\\n      {{- end }}\\n      io.rancher.container.pull_image: always\\n      io.rancher.container.hostname_override: container_name\\n      io.rancher.scheduler.affinity:container_label_ne: io.rancher.stack_service.name=\\$\\${stack_name}/\\$\\${service_name}\\n      io.rancher.scheduler.affinity:host_label_ne: px/enabled=false\\n    environment:\\n      CLUSTER_ID: \\${CLUSTER_ID}\\n      KVDB: \\${KVDB}\\n      USE_DISKS: \\${USE_DISKS}\\n      EXTRA_OPTS: \\${EXTRA_OPTS}\\n      ENABLE_ATTR_CACHE: true\\n    volumes:\\n      - /var/run/docker.sock:/var/run/docker.sock:rprivate\\n      - /etc/pwx:/etc/pwx:rprivate\\n      - /opt/pwx:/opt/pwx:rprivate\\n      - /etc/systemd/system:/etc/systemd/system:rprivate\\n      - /proc/1/ns:/host_proc/1/ns:rprivate\\n      - /var/run/dbus:/var/run/dbus:rprivate\\n    command: -c \\${CLUSTER_ID} -k \\${KVDB} \\${USE_DISKS} -m eth1 -d eth1 -x rancher --endpoint 0.0.0.0:9015 \\${EXTRA_OPTS}\\n","rancherCompose":".catalog:\\n  name: \\"Portworx\\"\\n  version: \\"#{template_version}\\"\\n  description: \\"Portworx Docker Volume Driver\\"\\n  questions:\\n\\n    - variable: INSTALL_SCALE\\n      label: Install scale\\n      description: \\"Determine if Portworx should install globally on all the nodes, or scaled via stacks (starting with Scale=1).\\"\\n      required: true\\n      default: \\"global\\"\\n      type: enum\\n      options:\\n      - \\"global\\"\\n      - \\"scaled\\"\\n    - variable: \\"CLUSTER_ID\\"\\n      description: \\"Arbitrary Cluster ID, common to all nodes in PX cluster \\"\\n      label: \\"Cluster ID\\"\\n      type: \\"string\\"\\n      required: true\\n      default: \\"\\"\\n    - variable: \\"KVDB\\"\\n      description: \\"Key-value database, accessibe to all nodes in PX cluster\\"\\n      label: \\"Key-Value Database\\"\\n      type: \\"string\\"\\n      required: true\\n      default: \\"\\"\\n    - variable: \\"USE_DISKS\\"\\n      description: \\"Specify which disks to use (Ex: '-a' for all available unformatted disks, or '-s /dev/sdX' for each individual disk)\\"\\n      label: \\"Use Disks\\"\\n      type: \\"string\\"\\n      required: true\\n      default: \\"-a\\"\\n    - variable: \\"EXTRA_OPTS\\"\\n      description: \\"Extra portworx options (see https://docs.portworx.com/runc/#command-line-arguments)\\"\\n      label: \\"Extra Options\\"\\n      type: \\"string\\"\\n      required: false\\n      default: \\"\\"\\n\\nportworx:\\n  metadata:\\n    version: 1\\n    storageDriverName: pxd\\n    debug: true\\n    volumeDriverName: pxd\\n  storage_driver:\\n    name: pxd\\n    scope: environment\\n    volume_access_mode: multiHostRW\\n    blockDevicePath: null\\n  start_on_create: true\\n  health_check:\\n    request_line: GET /health HTTP/1.0\\n    port: 9015\\n    interval: 10000\\n    response_timeout: 5000\\n    unhealthy_threshold: 3\\n    healthy_threshold: 2\\n    initializing_timeout: 840000\\n    strategy: none\\n\\n","externalId":"catalog://library:infra*portworx:50"}
EOF
        echo End
      ) &>/var/log/vagrant.bootstrap &
    SHELL
  end

  (1..nodes).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.hostname = "node#{i}"
      node.vm.network "private_network", ip: "192.168.99.10#{i}", virtualbox__intnet: true
      if i === 1
        node.vm.network "forwarded_port", guest: 32678, host: 32678
      end
      node.vm.provider "virtualbox" do |vb| 
        vb.memory = 2048
        vb.customize ["modifyvm", :id, "--name", "node#{i}"]
        if File.exist?("#{vb_dir}/disk#{i}.vdi")
          vb.customize ['closemedium', "#{vb_dir}/disk#{i}.vdi", "--delete"]
        end
        vb.customize ['createmedium', 'disk', '--filename', "#{vb_dir}/disk#{i}.vdi", '--size', disk_size]
        vb.customize ['storageattach', :id, '--storagectl', 'IDE', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', "#{vb_dir}/disk#{i}.vdi"]
      end
      node.vm.provision "shell", inline: <<-SHELL
        ( curl -sL https://rpm.nodesource.com/setup_10.x | bash -
          yum install -y docker nodejs
          systemctl enable docker
          systemctl start docker
          npm install -g json
          while : ; do
            env_id=$(curl http://192.168.99.99:8080/v2-beta/project?name=Default | json data[0].id)
            [ $env_id ] && break
            sleep 5
          done
          while : ; do
            command=$(curl "http://192.168.99.99:8080/v2-beta/projects/$env_id/registrationtokens/?state=active" | json data[0].command)
            [ "$command" ] && break
            sleep 1
          done
          echo $command | sh
          echo End
        ) &>/var/log/vagrant.bootstrap &
      SHELL
    end
  end

end
