Vagrant.configure("2") do |config|
  # Базовый образ
  config.vm.box = "ubuntu/focal64"
  config.vm.hostname = "inception-k8s"

  # --- ПУТЬ А: Проброс портов (без sudo) ---
  # Запрос на localhost:8080 хоста пойдет на порт 80 виртуалки
  # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1", auto_correct: true
  # Запрос на localhost:8443 хоста пойдет на порт 443 виртуалки (Traefik)
  # config.vm.network "forwarded_port", guest: 443, host: 8443, host_ip: "127.0.0.1", auto_correct: true

  # --- ПУТЬ Б: Выделенный IP (Рекомендую использовать его) ---
  # Это позволит тестировать роутинг Traefik по доменным именам через /etc/hosts
  config.vm.network "private_network", ip: "192.168.56.10"

  # Конфигурация ресурсов ВМ
  config.vm.provider "virtualbox" do |vb|
    vb.name = "inception-k8s-vm"
    vb.memory = "5000" # 3 ГБ RAM (хватит для K3s, БД и WordPress)
    vb.cpus = 4
    
    # Отключаем GUI, нам нужна только консоль
    vb.gui = false
  end
end

