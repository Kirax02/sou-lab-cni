# SOU-LAB-CNI
### Questa repository contiene un progetto vagrant ansible, per configurare 2 nodi.  
1. Nel primo nodo chiamato soufe1, andiamo a installare podman e a creare un container con il rispettivo file di configurazione per configurarlo in modo che faccia da reverse proxy per grafana e prometheus installati sul secondo nodo chiamato soufe2.
2. Nel secondo nodo chiamato soufe2, andiamo a installare podman e a creare 2 container, uno per grafana e uno per prometheus.
## Test del progetto
Per testare questo progetto bisogna clonare questa repository con il comando:  
`git clone https://github.com/Kirax02/sou-lab-cni | cd sou-lab-cni`e inviare il comando `vagrant up`.
## Struttura delle directory
```bash
sou-lab-cni
├── README.md
├── Vagrantfile
├── playbook.yml
└── roles
    └── sou_podman
        ├── tasks
        │   └── main.yml
        └── templates
            ├── grafana.ini.j2
            ├── haproxy.cfg.j2
            └── prometheus.yml.j2
```


## Vagrantfile
```vagrantfile
Vagrant.configure("2") do |config|
  config.vm.define "soufe1" do |soufe1|
    soufe1.vm.hostname = "soufe1"
    soufe1.vm.box = "almalinux/9"
    soufe1.vm.network "private_network", ip: "192.168.10.101", netmask: "255.255.0.0"
    soufe1.vbguest.auto_update = false
    soufe1.vm.provision "ansible" do |ansible|
      ansible.playbook = "playbook.yml"
      ansible.tags = "soufe1"
    end
  end

  config.vm.define "soufe2" do |soufe2|
    soufe2.vm.hostname = "soufe2"
    soufe2.vm.box = "almalinux/9"
    soufe2.vm.network "private_network", ip: "192.168.10.102", netmask: "255.255.0.0"
    soufe2.vbguest.auto_update = false
    soufe2.vm.provision "ansible" do |ansible|
      ansible.playbook = "playbook.yml"
      ansible.tags = "soufe2"
    end
  end
end
```
Con il seguente Vagrantfile andiamo a creare 2 nodi sotto la rete `192.168.10.0/16` che saranno configurati con il provisioner **ansible**. Aggiungiamo dei tag ai nodi cosi che possiamo scegliere quali tasks eseguire.
## Roles sou_podman 
```yaml
---
- name: installing podman
  package:
    name: "podman"
    state: present
  tags:
    - soufe1
    - soufe2

- name: Create a directories for soufe1
  file:
    path: "{{ item }}"
    state: directory
  loop:
    - "/etc/haproxy"
    - "/etc/ssl/crt"
    - "/etc/ssl/private"
    - "/etc/ssl/csr"
  tags:
    - soufe1

- name: Install Python cryptography
  package:
    name: python3-cryptography
    state: present
  tags:
    - soufe1

- name: Generate configuration file for haproxy container
  template:
    src: haproxy.cfg.j2
    dest: /etc/haproxy/haproxy.cfg
  tags:
    - soufe1

- name: Generate a private key for haproxy
  openssl_privatekey:
    path: /etc/ssl/private/haproxy.pem
    cipher: auto
    size: 2048
  tags:
    - soufe1

- name: Generate a CSR for haproxy
  openssl_csr:
    path: /etc/ssl/csr/haproxy.csr
    privatekey_path: /etc/ssl/private/haproxy.pem
    common_name: "haproxy.local"
    country_name: "IT"
    organization_name: "lupacchiotti"
    state_or_province_name: "Lazio"
  tags:
    - soufe1
      
- name: Generate a Self Signed OpenSSL certificate for haproxy
  openssl_certificate:
    path: /etc/ssl/crt/haproxy.crt
    privatekey_path: /etc/ssl/private/haproxy.pem
    csr_path: /etc/ssl/csr/haproxy.csr
    provider: selfsigned
  tags:
    - soufe1

- name: Create concatenated certificate
  shell: cat /etc/ssl/private/haproxy.pem /etc/ssl/crt/haproxy.crt > /etc/ssl/crt/haproxy.pem
  args:
    executable: /bin/bash
  tags:
    - soufe1

- name: Pull image for haproxy
  podman_image:
    name: docker.io/library/haproxy
  tags:
    - soufe1

- name: Create haproxy container
  containers.podman.podman_container:
    name: haproxy
    image: haproxy
    state: started
    user: root
    ports: 
      - "80:80"
      - "443:443"
    volumes:
      - "/etc/haproxy:/usr/local/etc/haproxy:Z"
      - "/etc/ssl/crt:/usr/local/etc/haproxy/certs:Z"
  tags:
    - soufe1

- name: Create a directories for soube2
  file:
    path: "{{ item }}"
    state: directory
  loop:
    - /etc/prometheus
    - /var/lib/prometheus
    - /etc/grafana
    - /var/lib/grafana
  tags:
    - soufe2

- name: Generate configuration file for prometheus
  template:
    src: prometheus.yml.j2
    dest: /etc/prometheus/prometheus.yml
  tags:
    - soufe2

- name: Pull prometheus image
  podman_image:
    name: docker.io/prom/prometheus
  tags:
    - soufe2

- name: Create prometheus container
  containers.podman.podman_container:
    name: prometheus
    image: prom/prometheus
    state: started
    ports:
      - "9090:9090"
    volumes: 
      - "/etc/prometheus:/etc/prometheus:Z"
      - "/var/lib/prometheus:/var/data/prometheus:Z"
  tags:
    - soufe2

- name: Generate configuration file for grafana
  template:
    src: grafana.ini.j2
    dest: /etc/grafana/grafana.ini
  tags:
    - soufe2

- name: Pull grafana image
  podman_image:
    name: docker.io/grafana/grafana
  tags:
    - soufe2

- name: Create grafana container
  containers.podman.podman_container:
    name: grafana
    image: grafana/grafana
    state: started
    ports: 
      - "3000:3000"
    volumes:
      - "/etc/grafana:/etc/grafana:Z"
      - "/var/lib/grafana:/var/data/grafana:Z"
  tags:
    - soufe2
```
Con le seguenti tasks andiamo a installare podman su entrambi i nodi e a creare i vari container solo sul nodo interessato grazie ai tags.

