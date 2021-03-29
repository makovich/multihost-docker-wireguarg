N = 3

Vagrant.configure("2") do |config|
  N.times do |n|
    config.vm.define "barge#{n+1}" do |node|
      node.vm.provider "virtualbox" do |vb|
        vb.customize ["modifyvm", :id, "--memory", "256"]
      end
      node.vm.box = "ailispaw/barge"
      node.vm.hostname = "barge#{n+1}"
      node.vm.network "private_network", ip: "10.10.10.10#{n+1}"

      node.vm.provision "shell", inline: install()
      node.vm.provision "shell", inline: setup_wgconf(n)
      node.vm.provision "shell", inline: setup_docker(n)
      node.vm.provision "shell", inline: setup_interface()
      node.vm.provision "shell", inline: build_image()
      node.vm.provision "docker" do |d|
        rand(1..5).times do |c|
          d.run "serf#{c}",
            image: "serf",
            args: "--hostname=barge#{n+1}-serf#{c} --net=ovl0"
        end
      end
    end
  end
end

def pk(n)
  [
    "mNt5SBnjtZKKxERK8M5cjNbIw2Zzw1vlHrEfR1wpjVc=",
    "yDLjejIl+yyKtB+AnfY3rEGV6bBdZyXhElZtEUZRXVQ=",
    "mCZjHr567vXHe6J3qnNhpyJUFDJKLKZE+dXHVS3y1EQ=",
    "iKaPZEMDMUL+ZI99xLILH9gJolkxhRsvSlf/ut2S7WA=",
    "oFjNIucL38A3tP3+MR2YyZYI2M7VHoijQ7phTZIn/Vk=",
  ][n]
end

def pub(n)
  [
    "buG9gK++Fj4jXRXKX0CF55IANhCVQRL6/WswgzFGp10=",
    "XMwIuy2+dd4fh1lw1dMJH7VlV3xKNv/D37+X7M1yliM=",
    "oS10xs91rvL8z66aVqgkLF4XCvBHT1sRf/M5jTmkowA=",
    "vXAAuDPouRTNPmSqjLgmKjoWRujeoYlwUIaHMmcPNgk=",
    "KsHREmKP6w2CSNS7T2b73HS66YIDkedl8Ro+M3ujYD8="
  ][n]
end

def install()
  %{
echo "Installing WireGuard"
# tcpdump for debugging
# wget https://raw.githubusercontent.com/yunchih/static-binaries/master/tcpdump -O /bin/tcpdump && chmod +x /bin/tcpdump

# find out your instructions at https://www.wireguard.com/install/
# this is specific to barge-os
pkg install kmod
pkg install wireguard
depmod -a
modprobe wireguard

umask 077
mkdir /etc/wireguard
  }
end

def setup_wgconf(n)
  ret = %{
echo "Setting up wg0.conf"

cat > /etc/wireguard/wg0.conf <<EOF
[Interface]
PrivateKey = #{pk(n)}
ListenPort = 12345
  }

  (1...N).each do |p| ret << %{
[Peer]
PublicKey = #{pub((n+p)%N)}
Endpoint = 10.10.10.10#{(n+p)%N+1}:12345
AllowedIPs = 172.20.10#{(n+p)%N+1}.0/24
  }
  end

  ret << %{
EOF
  }
end

def setup_docker(n)
  %{
echo "Setting up Docker network ovl0"

docker network create \
  --driver=bridge \
  --subnet=172.20.0.0/16 \
  --ip-range=172.20.10#{n+1}.0/24 \
  --opt "com.docker.network.bridge.name"="ovl0" \
  ovl0

ip route del 172.20.0.0/16 dev ovl0
ip route add 172.20.10#{n+1}.0/24 dev ovl0
  }
end

def setup_interface()
  %{
echo "Setting up wg0 interface"

cat >> /etc/network/interfaces <<EOF
auto wg0
iface wg0 inet static
  address 0.0.0.0
  netmask 255.255.255.255
  pre-up ip link add dev wg0 type wireguard
  pre-up wg setconf wg0 /etc/wireguard/wg0.conf
  post-up sysctl net.ipv4.conf.ovl0.proxy_arp=1
  post-up ip route add 172.20.0.0/16 dev wg0
  post-up iptables -t nat -I POSTROUTING -o wg0 -s 172.20.0.0/16 -j ACCEPT
  post-down iptables -t nat -D POSTROUTING -o wg0 -s 172.20.0.0/16 -j ACCEPT
  post-down sysctl net.ipv4.conf.ovl0.proxy_arp=0
  post-down ip link delete dev wg0
EOF
ifup wg0
  }
end

def build_image()
  %{
echo "Building Docker image 'serf'"

cat > Dockerfile <<EOF
FROM busybox

RUN wget https://releases.hashicorp.com/serf/0.8.2/serf_0.8.2_linux_amd64.zip \
 && unzip -d /bin serf_0.8.2_linux_amd64.zip

CMD ["/bin/serf", "agent", "-retry-join", "172.20.101.1"]
EOF

docker build -t serf .
  }
end
