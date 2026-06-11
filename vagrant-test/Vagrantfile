Vagrant.configure("2") do |config|
  
  # 統一使用官方的 Rocky Linux 9 鏡像
  config.vm.box = "generic/rocky9"

  # ====================================================
  # 1. 主控節點 (Control-Node): 未來用來執行 Ansible
  # ====================================================
  config.vm.define "control" do |control|
    control.vm.hostname = "control-node"
    # 設定私有網路固定 IP，發揮你的網路規劃能力
    control.vm.network "private_network", ip: "192.168.56.10"
    
    # 設定虛擬機硬體規格 (1核心 CPU, 1GB 記憶體)
    control.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 1
    end
  end

  # ====================================================
  # 2. 網頁伺服器 (Web-Server): 未來部署 Docker & WordPress
  # ====================================================
  config.vm.define "web" do |web|
    web.vm.hostname = "web-server"
    web.vm.network "private_network", ip: "192.168.56.11"
    
    web.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 1
    end
  end

  # ====================================================
  # 3. 監控伺服器 (Monitor-Server): 未來部署 Prometheus + Grafana
  # ====================================================
  config.vm.define "monitor" do |monitor|
    monitor.vm.hostname = "monitor-server"
    monitor.vm.network "private_network", ip: "192.168.56.12"
    
    monitor.vm.provider "virtualbox" do |vb|
      vb.memory = "1024"
      vb.cpus = 1
    end
  end

end
