# Documentação

Guia pela ordem em que as coisas se fazem. Para cada VM nova do trabalho prático: Parte 3 (criar a VM), Parte 4 (rede do cluster) e depois a parte do serviço dessa VM (ex.: Parte 5 para a BD). A Parte 7 é só consulta.

- Parte 1. Preparar o Mac
- Parte 2. VM modelo (`ubuntu-base`)
- Parte 3. Criar uma VM nova a partir do modelo
- Parte 4. Rede do cluster (IP fixo e nomes)
- Parte 5. Cluster de BD: MariaDB + Galera
- Parte 6. HAProxy + Keepalived à frente da BD (slides 30 e 31)
- Parte 7. Consulta
- Falta fazer


# Parte 1. Preparar o Mac

## 1.1 Preparar o disco
1. Reformatar o SanDisk para APFS.
2. Criar a pasta `UTM` e, dentro, `ISOs` (com o ISO do Ubuntu).

## 1.2 Controlar as VMs pelo terminal
1. Criar o script `~/bin/vms` (usa o `utmctl`) com `vms up`, `vms down`, `vms ls`.
2. Acrescentar `~/bin` ao PATH no `~/.zshrc`.
3. O `--hide` do `utmctl` só esconde a janela principal do UTM, por isso o script também esconde o UTM com `osascript`.
4. O UTM tem de ficar aberto (se o fechar as VMs param) e o SanDisk ligado.


# Parte 2. VM modelo (`ubuntu-base`)
1. No UTM, criar uma VM com 12 GB de disco e instalar o Ubuntu Server 26.04 ARM64.
2. Na instalação usar 2 GB de RAM (com 1 GB o teclado perde teclas); depois voltar a 1 GB.
3. Instalar o `openssh-server` durante a instalação do Ubuntu (selecionar na lista de pacotes) ou depois com `sudo apt install openssh-server`.
4. Atualizar o sistema: `sudo apt update && sudo apt upgrade -y`.
5. Limpar o `/etc/machine-id` para que cada clone gere um ID único: `sudo truncate -s 0 /etc/machine-id` (será regenerado no próximo boot).
6. Desligar a `ubuntu-base`: só serve para clonar.


# Parte 3. Criar uma VM nova a partir do modelo
Repetir para cada VM (exemplo: `db01`).

## 3.1 Clonar e preparar
1. Botão direito na `ubuntu-base` desligada, Clone, nome `db01`.
2. Antes de ligar, gerar um MAC aleatório em Network (o UTM não o muda sozinho).
3. Confirmar com **Show in Finder** que a clone ficou no SanDisk (`OS Temp/UTM/VMs`); se não, mover antes de ligar (ver 3.2).
4. Ligar a VM e mudar o nome com `sudo hostnamectl set-hostname db01`.
5. Gerar chaves SSH novas da máquina: `sudo rm /etc/ssh/ssh_host_* && sudo dpkg-reconfigure openssh-server`, depois `sudo reboot`.
6. Confirmar o IP com `ip a`.

## 3.2 Problemas ao clonar

### Erro `The file "edk2-arm-vars.fd" doesn't exist` ao clonar ou ligar
O que é e o que se faz: depois de mover o `.utm` para o SanDisk, o UTM continuava a apontar para a pasta antiga; volta-se a abrir a VM a partir do SanDisk.
- Causa: o UTM guarda o caminho da VM. Se a pasta mudou, não encontra o ficheiro de arranque (`efi_vars.fd`, dentro de `NOME.utm/Data/`). O ficheiro está bem no SanDisk, o que falha é o caminho guardado.
- Sinal: **Show in Finder** na VM não abre nenhuma janela.
- Correção: botão direito na VM, **Remove** (não Delete: só tira da lista, os ficheiros ficam). Depois **File → Open…** e abrir `OS Temp/UTM/VMs/NOME.utm`.
- Confirmar que os ficheiros existem: `ls -la "/Volumes/OS Temp/UTM/VMs/NOME.utm/Data/"` (deve mostrar o `.qcow2` e o `efi_vars.fd`).

### A clone fica com o mesmo IP da NAT (`enp0s1`) que o original
O que é e o que se faz: a NAT do UTM dá o IP por DHCP, que identifica a máquina pelo `machine-id`; a clone herdou o do original, por isso gera-se um novo.
- Dentro da clone: `sudo rm /etc/machine-id` e `sudo systemd-machine-id-setup` (apaga o ID copiado e cria um novo e aleatório).
- Reiniciar (`sudo reboot`) e confirmar com `ip -br a` que a `enp0s1` tem um IP diferente do original.

## 3.3 Ligar por SSH com o Warp
1. Ligar à `db01` por SSH no Warp (`ssh rodrigo@192.168.64.X`).
2. O Warp mostra a árvore de ficheiros da VM e cria `.warp` e `.config/warp-terminal` na VM.
3. Sair de uma sessão sem desligar a VM: `exit`, Ctrl+D ou fechar o tab.

## 3.4 Login por chave SSH
1. A chave pública é mais segura e cómoda que a password.
2. Se já existirem chaves no Mac (`id_ed25519` e `id_rsa`), responder n ao `ssh-keygen` para não as substituir.
3. `ssh-copy-id rodrigo@192.168.64.6` para pôr a chave na `db01`.
4. `ssh db01` e `ssh ubuntu-base` entram sem pedir a password da VM.

## 3.5 Aviso SSH "REMOTE HOST IDENTIFICATION HAS CHANGED"
- Causa: o IP da NAT (ex.: 192.168.64.3) já tinha sido de outra máquina (ou a VM mudou de chaves SSH); o Mac guardou a chave antiga em `~/.ssh/known_hosts`.
- Correção, **no Mac** (não dentro da VM): `ssh-keygen -R 192.168.64.3` e depois `ssh rodrigo@192.168.64.3` (responder `yes` à pergunta da impressão digital).

Exemplo do aviso (com `ssh rodrigo@192.168.64.3`):

> @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
> @    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
> @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
> IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
> Someone could be eavesdropping on you right now (man-in-the-middle attack)!
> It is also possible that a host key has just been changed.
> The fingerprint for the ED25519 key sent by the remote host is
> SHA256:LvQ1wMu0J3wQQ8IC/HvVID9CoVCy3dpOi3aXK46AXzw.
> Please contact your system administrator.
> Add correct host key in /Users/rodrigobranco/.ssh/known_hosts to get rid of this message.
> Offending ECDSA key in /Users/rodrigobranco/.ssh/known_hosts:7
> Host key for 192.168.64.3 has changed and you have requested strict checking.
> Host key verification failed.


# Parte 4. Rede do cluster (IP fixo e nomes)
Em cada VM do cluster.

## Conceitos
- **Placa de rede** = ligação de uma máquina a uma rede. Na VM é emulada pelo UTM.
- Cada VM tem 2 placas: `enp0s1` = NAT (Internet, updates, SSH, IP automático 192.168.64.x); `enp0s2` = Host Only (rede privada do cluster, IP fixo 10.84.128.x). No fim desliga-se a NAT.
- `/etc/netplan/` = dar IP fixo à placa. `/etc/hosts` = tabela nome → IP (sem DNS). São duas coisas diferentes.

## IPs escolhidos (do slide 25)
| VM (hostname e `/etc/hosts`) | Nome no Galera | IP cluster (enp0s2) |
|---|---|---|
| db01 | db1 | 10.84.128.11 |
| db02 | db2 | 10.84.128.16 |
| (3.ª, mais tarde) | db3 | 10.84.128.13 |
| haproxydb01 | (haproxydb1 no `/etc/hosts`) | 10.84.128.17 |
| haproxydb02 (ainda não criada) | haproxydb2 | 10.84.128.21 |
| IP virtual da BD (Keepalived) | | 10.84.128.253 |

## 4.1 Segunda placa de rede (UTM, VM desligada)
1. Botão direito na VM → **Edit**.
2. Devices → **New…** → **Network**.
3. Network Mode = **Host Only** → **Save**.
4. A placa antiga (Shared Network) fica como está.

## 4.2 Ver as placas (dentro da VM)
```
ip -br a      # placas e IPs (-br = uma linha por placa; a = address)
ip -br link   # placas e MAC real de cada uma
```

## 4.3 Ficheiro novo do netplan com o IP fixo da `enp0s2`
```
sudo nano /etc/netplan/60-cluster.yaml
```
Conteúdo (db01; na db02 o IP passa a 10.84.128.16/24):
```yaml
network:
  version: 2
  ethernets:
    enp0s2:
      dhcp4: false
      dhcp6: false
      addresses:
        - 10.84.128.11/24
```
Guardar no nano: Ctrl+O, Enter, Ctrl+X. Os espaços do início das linhas importam.

## 4.4 Permissões do ficheiro
```
sudo chmod 600 /etc/netplan/60-cluster.yaml   # só o dono lê/escreve (o netplan exige)
```

## 4.5 Corrigir o MAC no ficheiro antigo do netplan (em todas as clones)
Sem isto, o `netplan apply` dá "Cannot find unique matching interface for enp0s1".
- Causa: o ficheiro antigo `/etc/netplan/00-installer-config.yaml` identifica a placa NAT pelo MAC do `ubuntu-base`; a clone tem MAC novo.
- Ver o MAC real: `ip -br link` (linha `enp0s1`).
- Corrigir:
```
sudo nano /etc/netplan/00-installer-config.yaml
```
  e trocar a linha `macaddress:` pelo MAC real da `enp0s1` dessa VM:
  - db01: `7e:ac:26:70:d7:d4`
  - db02: `02:d8:35:26:0d:d7`

## 4.6 Aplicar e confirmar
```
sudo netplan apply
ip -br a
```
- A SSH cai e a NAT muda de IP (db01: 192.168.64.6 → .5). Esperar uns 20 s e ligar ao IP novo: `ssh rodrigo@192.168.64.5` (se aparecer o aviso da 3.5, corrigir como lá diz).
- `enp0s2` deve mostrar `10.84.128.11/24` (db01) ou `10.84.128.16/24` (db02).
- **db01: feito e confirmado (NAT 192.168.64.5). db02: feito e confirmado, `enp0s2` = 10.84.128.16/24 (NAT passou a 192.168.64.3).**

## 4.7 Dar nomes aos IPs fixos (`/etc/hosts`)
O que é: sem DNS, o `/etc/hosts` é a tabela local nome → IP. O que se faz: acrescentar uma linha por máquina para poder usar `db01`/`db02` em vez dos IPs.
```
sudo nano /etc/hosts
```
- `sudo`: executar como administrador (o ficheiro é do sistema).
- `nano`: editor de texto no terminal.
- `/etc/hosts`: o ficheiro a editar.

Acrescentar no fim (igual em todas as VMs):
```
10.84.128.11 db01
10.84.128.16 db02
```
- Formato de cada linha: IP, espaço, nome.
- Guardar no nano: Ctrl+O, Enter, Ctrl+X.

Testar (a partir de cada VM):
```
ping -c 2 db02
```
- `ping`: envia pacotes de teste e espera resposta.
- `-c 2`: enviar só 2 pacotes (c = count) e parar.
- `db02`: o nome que o `/etc/hosts` traduz para 10.84.128.16 (na db02 testar `db01`).

**Confirmado: funciona nas duas VMs.**


# Parte 5. Cluster de BD: MariaDB + Galera
Aula de 2026-10-06, slides 24 a 29 do PDF `Cluster_HA_LOAD_B_CLOUD.pdf`. Só 2 nós (db01, db02); o 3.º fica para o trabalho prático.

## Conceitos
- **Nó** = cada máquina do cluster (cada VM de BD).
- **Galera** = extra do MariaDB que mantém os dados iguais em todos os nós (escreve-se num, aparece nos outros).
- **WSREP** = Write-Set REPlication (a replicação do Galera).

## 5.1 Instalar o MariaDB (em todos os nós, painéis sincronizados)
```
sudo apt-get update
sudo apt-get install apache2 mariadb-server rsync -y
```
Verificar: `mariadb --version`, `systemctl status mariadb`, `dpkg -l | grep -E 'mariadb-server|apache2|rsync'` (`ii` = instalado).

## 5.2 Segurança do MariaDB (slide 26)
- O comando agora chama-se `sudo mariadb-secure-installation` (antes `mysql_secure_installation`).
- Respostas: Enter nas duas perguntas da password atual; unix_socket = n; mudar password de root = Y (mesma em todos os nós); remover anónimos Y; root remoto não Y; remover base de teste Y; recarregar Y.
- Utilizador do cluster (slide 26): não é preciso com `wsrep_sst_method=rsync`.

## 5.3 Preparar o Galera (em todos os nós)
```
ls -l /usr/lib/galera/                      # confirmar libgalera_smm.so
sudo touch /var/log/mariadb.log
sudo chown mysql:mysql /var/log/mariadb.log
sudo systemctl stop mariadb
```

## 5.4 Ficheiro `/etc/mysql/conf.d/galera.cnf` (em cada nó, em separado)
Tem de acabar em `.cnf` (o MariaDB ignora outras extensões; em caso de engano na extensão: `sudo mv galera.dnf galera.cnf`).
```
[mysqld]
binlog_format=ROW
default-storage-engine=innodb
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0
log_error=/var/log/mariadb.log
wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so
wsrep_cluster_name="moodle_galera_cluster"
wsrep_cluster_address="gcomm://10.84.128.11,10.84.128.16"
wsrep_sst_method=rsync
wsrep_node_address="10.84.128.11"   # db02: 10.84.128.16
wsrep_node_name="db1"               # db02: db2
```

## 5.5 Firewall (nos nós com a ufw ativa; na db01 estava)
```
sudo ufw allow from 10.84.128.0/24 to any port 4567
sudo ufw allow from 10.84.128.0/24 to any port 4568 proto tcp
sudo ufw allow from 10.84.128.0/24 to any port 4444 proto tcp
sudo ufw allow from 10.84.128.0/24 to any port 3306 proto tcp
```
Sintoma sem isto: o 2.º nó fica a tentar ligar (`wsrep::connect ... failed: 7`; `nc -zv IP 4567` fica preso).

## 5.6 AppArmor (em todos os nós)
- Sintoma sem isto: db02 entra no cluster mas rebenta (SEGV) com `posix_spawnp(wsrep_sst_rsync) failed: 13 (Permission denied)`; o kernel regista `apparmor="DENIED" ... name="/usr/bin/dash"`.
- Solução do laboratório:
```
sudo apt-get install -y apparmor-utils
sudo aa-complain mariadbd      # com d no fim
```

## 5.7 Arrancar o cluster (slide 28)
```
sudo galera_new_cluster        # SÓ no 1.º nó (db01)
sudo systemctl start mariadb   # nos outros nós (db02)
mysql -u root -p -e "show status like 'wsrep_cluster_size'"   # deve dar 2
```
**Resultado: cluster com 2 nós a funcionar.**

## 5.8 Como ver se o cluster iniciou (dentro da VM)
```
mysql -u root -p -e "show status where Variable_name in ('wsrep_cluster_size','wsrep_cluster_status','wsrep_ready')"
sudo ss -tlnp | grep 4567       # o Galera está à escuta
```
Saudável: `wsrep_cluster_size` = nº de nós (2), `wsrep_cluster_status` = `Primary`, `wsrep_ready` = `ON`.

## 5.9 Teste de replicação (slide 29)
```
# db01:
mysql -u root -p -e "CREATE DATABASE teste_cluster;"
# db02:
mysql -u root -p -e "SHOW DATABASES;"          # teste_cluster aparece
# limpar:
mysql -u root -p -e "DROP DATABASE teste_cluster;"
```
`-u root` = utilizador root **da base de dados** (não o do Ubuntu); `-p` = pede a password.
**Confirmado: a replicação funciona. Slides 24 a 29 concluídos.**

## 5.10 Desligar e voltar a ligar o cluster
- Desligar: `sudo systemctl stop mariadb` primeiro na **db02**, depois na **db01** (a última); só depois desligar as VMs.
- Ligar: ligar as VMs; na **db01** (a última que parou) `sudo galera_new_cluster`; na **db02** `sudo systemctl start mariadb`; confirmar `wsrep_cluster_size` = 2.
- Regra: **o último a desligar é o primeiro a arrancar** com `galera_new_cluster`.
- Se falhar com "safe to bootstrap": `sudo cat /var/lib/mariadb/grastate.dat`; o nó com `safe_to_bootstrap: 1` é o que deve arrancar primeiro.


# Parte 6. HAProxy + Keepalived à frente da BD (slides 30 e 31)
Aula de 2026-10-12. Objetivo: duas máquinas HAProxy (haproxydb1 `.17` e haproxydb2 `.21`) à frente do cluster de BD, para alta disponibilidade e balanceamento de carga. Os clientes (mais tarde o Moodle) falam só com o IP virtual `10.84.128.253`, que passa de um HAProxy para o outro se um falhar. Hoje só a haproxydb01.

## 6.1 Criar a haproxydb01 (UTM)
O que é e o que se faz: uma VM nova clonada da `ubuntu-base`, com 512 MB, igual às outras (Parte 3).
1. `ubuntu-base` desligada e SanDisk ligado. Botão direito → **Clone**, nome `haproxydb01`.
2. **Show in Finder**: confirmar que ficou em `UTM/VMs` no SanDisk.
3. **Edit** → na placa existente, MAC **Random**; **New… → Network → Host Only**, também MAC **Random**.
4. **System** → memória **512 MB** → **Save**.

Engano evitado: o primeiro clone (`ha1db`) tinha sido feito a partir da **db02** e a `enp0s2` já vinha com o IP `10.84.128.16`, porque o IP fixo está no disco (`60-cluster.yaml`), não no MAC. Também trazia o MariaDB e o Galera da db02, que podiam tentar entrar no cluster. Solução: `sudo systemctl stop mariadb`, `sudo poweroff`, apagar essa VM (cuidado a escolher a certa) e clonar a `ubuntu-base`. Antes de apagar, confirmar na db01 que `wsrep_cluster_size` continua a 2.

## 6.2 Nome, MAC e IP fixo (.17)
O que é e o que se faz: repetir na haproxydb01 o que já se fez nas BD, com o IP `10.84.128.17`.
1. Ligar e ver as placas: `ip -br a` e `ip -br link`.
2. Nome: `sudo hostnamectl set-hostname haproxydb01` (como em 3.1).
3. MAC real da `enp0s1` no `00-installer-config.yaml` (como em 4.5).
4. `/etc/netplan/60-cluster.yaml` com `10.84.128.17/24` na `enp0s2`, `chmod 600` e `sudo netplan apply` (como em 4.3, 4.4 e 4.6).
5. Confirmar: a `enp0s2` mostra `10.84.128.17/24`.
- O IP da NAT mudou (192.168.64.10 → .3) depois do `netplan apply`. É normal: a NAT dá o IP por DHCP e a VM voltou a pedi-lo. Não afeta o cluster, porque a rede do cluster é a `enp0s2`, com IP fixo. Voltar a ver o IP com `ip -br a` depois de cada `netplan apply`.

## 6.3 Problema: a haproxydb01 recebeu o mesmo IP da NAT que a db02
O que é: a haproxydb01 ia sempre buscar o `192.168.64.3`, o da db02, mesmo com MACs diferentes.
- Causa provável: a clone herdou o `machine-id` da `ubuntu-base` e o DHCP do UTM identifica a máquina por ele.
- Ver: `cat /etc/machine-id` em cada VM (se for igual, é isto).
- Correção: a da secção 3.2 (`sudo rm /etc/machine-id`, `sudo systemd-machine-id-setup`, `sudo reboot`).
- Para o futuro: logo depois de clonar, gerar o `machine-id` novo, tal como as chaves SSH.

## 6.4 `/etc/hosts` da haproxydb01
O que é e o que se faz: tabela nome → IP com as BD e os HAProxy, como pede o slide 31 (sem DNS).
```
sudo nano /etc/hosts
```
- A linha `127.0.1.1 ubuntu-base` trocar por `127.0.1.1 haproxydb01` (é o nome da própria máquina; se ficar o antigo, o `sudo` pode queixar-se de "unable to resolve host").
- Acrescentar no fim:
```
10.84.128.11 db1
10.84.128.16 db2
10.84.128.13 db3
10.84.128.17 haproxydb1 haproxydb01
10.84.128.21 haproxydb2
```
- Uma linha por IP; vários nomes na mesma linha, separados por espaço.
- O `ping -c 2 haproxydb01` só funciona porque o nome `haproxydb01` está na linha do `.17`. Sem isso dá "Temporary failure in name resolution", mesmo sendo o hostname.
- Cada VM tem o seu `/etc/hosts`: nada se partilha sozinho. A ideia é ter a mesma tabela em todas as máquinas, menos a linha `127.0.1.1`, que leva sempre o nome da própria máquina.
- Testar: `ping -c 2 db1` (deve dar `0% packet loss` com a db01 ligada).

## 6.5 Instalar HAProxy, Keepalived e Apache (slide 31)
O que é e o que se faz: os 3 serviços que o slide pede nas máquinas HAProxy: balanceador (HAProxy), IP virtual (Keepalived) e servidor web (Apache). Precisa de Internet, por isso usa a NAT.
```
sudo apt-get update
sudo apt-get install haproxy keepalived apache2 -y
```
- `apt-get update`: atualiza a lista de pacotes (não instala nada).
- `install`: instala os pacotes indicados.
- `haproxy`: recebe os pedidos e reparte-os pelas BD.
- `keepalived`: cria o IP virtual e decide qual dos 2 HAProxy o tem (VRRP).
- `apache2`: servidor web. O slide não diz para quê; dedução: página de teste ou verificação de que a máquina está viva. A confirmar com o professor.
- `-y`: responde "sim" às perguntas.

Confirmar:
```
haproxy -v
keepalived --version
systemctl is-active apache2
```
- `-v` e `--version`: mostram a versão. Resultado: HAProxy 3.2.9, Keepalived 2.3.4.
- `systemctl is-active apache2`: diz `active` se o serviço está a correr. Resultou `active`.

## 6.6 Firewall da haproxydb01
```
sudo ufw status
```
- `ufw`: a firewall do Ubuntu; `status` só mostra o estado.
- Resultou `inactive`: não é preciso abrir portas nesta máquina.

## 6.7 Keepalived: ficheiro de configuração (MASTER)
O que é e o que se faz: define o IP virtual `10.84.128.253` e que esta máquina é a principal (MASTER). A haproxydb02 será a reserva (BACKUP).
```
sudo nano /etc/keepalived/keepalived.conf
```
- O ficheiro ainda não existe: o nano abre-o vazio.
```
vrrp_instance VI_DB {
    state MASTER
    interface enp0s2
    virtual_router_id 53
    priority 101
    advert_int 1
    virtual_ipaddress {
        10.84.128.253/24
    }
}
```
- `vrrp_instance VI_DB`: grupo VRRP (o protocolo que decide quem tem o IP virtual); `VI_DB` é só um nome.
- `state MASTER`: arranca como principal.
- `interface enp0s2`: a placa onde o IP virtual aparece (a Host Only, do cluster).
- `virtual_router_id 53`: número do grupo (1 a 255); tem de ser igual nas duas máquinas.
- `priority 101`: ganha o número mais alto; a haproxydb02 terá um mais baixo (ex.: 100).
- `advert_int 1`: o MASTER diz "estou vivo" de 1 em 1 segundo; se deixar de dizer, o BACKUP assume.
- `virtual_ipaddress`: o IP virtual e a máscara.

## 6.8 Arrancar o Keepalived e ver o IP virtual
```
sudo systemctl enable --now keepalived
ip -br a show enp0s2
```
- `enable`: arranca sempre que a VM liga. `--now`: arranca também já.
- `ip -br a show enp0s2`: IPs só da `enp0s2`.
- Resultado: a `enp0s2` tem `10.84.128.17/24` (próprio) e `10.84.128.253/24` (virtual, `secondary`). **Confirmado.**

## 6.9 Parar o Keepalived e testar o IP virtual
```
sudo systemctl stop keepalived       # pára já; o .253 desaparece da placa
sudo systemctl disable keepalived    # tira do arranque (inverso de enable)
sudo systemctl start keepalived      # volta a ligar
systemctl is-active keepalived       # active / inactive
```
- Parado, o `.253` deixa de existir na VM. É assim que se simula uma falha quando a haproxydb02 existir: parar o Keepalived da 01 e ver o `.253` passar para a 02.
- O ping do Mac ao `.253` **não serve de teste**: respondia com `ttl=56` e tempos de 88 a 860 ms, ou seja, saía por outro caminho (provavelmente VPN ou rede externa que usa também 10.84.128.x). Testar a partir de uma VM do cluster, por exemplo a db01: `ping -c 3 10.84.128.253` (parado: `100% packet loss`; ligado: responde em menos de 1 ms).
- Para ver por onde o Mac envia um IP: `route -n get 10.84.128.253` (linhas `gateway:` e `interface:`).
- `arp -n IP` mostra o MAC associado a um IP (a haproxydb01 tem `4e:c7:71:6b:47:8b`); `sudo arp -d IP` apaga a entrada da tabela.
- **Teste a partir da db01 feito e resultou.**

## 6.10 Engano: instalar `mailutils-mh` sem querer
O que é e o que se faz: pacote instalado por engano; remove-se com a configuração e as dependências que só ele puxou.
```
sudo apt-get purge mailutils-mh -y
sudo apt-get autoremove --purge -y
```
- `purge`: remove o pacote e os ficheiros de configuração dele.
- `autoremove --purge`: remove os pacotes que só estavam instalados por causa dele.
- Ler a lista antes do Enter: se aparecer `haproxy`, `keepalived` ou `apache2`, parar.

## 6.11 HAProxy: balancear as ligações às BD (por fazer)
O que é e o que se faz: o HAProxy fica à escuta na porta 3306 e reparte as ligações pela db1 e db2, à vez.
```
sudo nano /etc/haproxy/haproxy.cfg
```
Acrescentar no fim, sem apagar o que lá está:
```
listen mariadb_cluster
    bind *:3306
    mode tcp
    balance roundrobin
    option tcpka
    server db1 10.84.128.11:3306 check
    server db2 10.84.128.16:3306 check
```
- `listen mariadb_cluster`: bloco que junta a entrada e os servidores de trás; o nome é livre.
- `bind *:3306`: escuta na porta 3306 em todos os IPs da máquina, incluindo o virtual.
- `mode tcp`: passa as ligações tal como vêm (o MariaDB não é HTTP).
- `balance roundrobin`: distribui à vez.
- `option tcpka`: mantém as ligações vivas com pequenos sinais.
- `server db1 10.84.128.11:3306 check`: servidor de trás (nome, IP, porta); `check` faz o HAProxy testar se está vivo e deixar de lhe enviar ligações se falhar. A linha da db2 é igual com o `.16`.

Validar antes de usar:
```
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```
- `-c`: só verifica (check), não arranca; `-f`: o ficheiro a verificar. Esperado: `Configuration file is valid`.
- Falta confirmar se o bloco foi acrescentado e validado; depois reiniciar o HAProxy (`sudo systemctl restart haproxy`) e testar.


# Parte 7. Consulta

## 7.1 O que quer dizer `mysql -u root -p -e "..."`
- `-u root`: entrar como o utilizador `root` **da base de dados** (não é o root do Ubuntu; o MariaDB tem utilizadores próprios).
- `-p`: pede a password (a do `mariadb-secure-installation`), sem a escrever no comando.
- `-e "..."`: executa essa instrução SQL e sai.

## 7.2 Conceitos do `galera_new_cluster`
- É um comando (script do MariaDB), não sintaxe. Liga o interruptor `_WSREP_NEW_CLUSTER` e arranca o MariaDB: cria o cluster de raiz a partir deste nó.
- O nome do cluster (`moodle_galera_cluster`) vem do `galera.cnf` (`wsrep_cluster_name`), escolhido por nós.
- Só se corre quando o cluster tem de nascer de novo: criação, ou quando **todos** os nós estavam parados. Se um nó avaria e os outros estão vivos, basta `sudo systemctl start mariadb` nesse nó.
- Perigo: correr `galera_new_cluster` com outros nós ativos cria clusters separados (split-brain).

## 7.3 Permissões Linux (resumo)
- r=4 (ler), w=2 (escrever), x=1 (executar/entrar na pasta). Ordem: dono, grupo, outros.
- 600 = só o dono lê/escreve · 644 = dono lê/escreve, resto lê · 755 = dono tudo, resto lê/executa.
- `ls -l`: `-rw-r--r--` = ficheiro, dono `rw-`, grupo `r--`, outros `r--` (644).

## 7.4 Recursos (RAM) e número de VMs
- Os slides indicam o dimensionamento original (BD 3x16 GB, HAProxy BD 2x4 GB, Web 3x12 GB, NFS 2x8 GB, Ubuntu 18.04; ~110 GB no total). Não é para copiar num portátil de 16 GB.
- Plano reduzido: BD 1 GB, HAProxy e NFS 512 MB, Web 1 GB = ~9 GB para as 12 VMs; **não ligar todas ao mesmo tempo** enquanto se constrói (macOS precisa de 4 a 6 GB). Vigiar a pressão de memória no Activity Monitor (verde ok, amarelo limite, vermelho demais).
- Os slides mostram 12 VMs mas o professor falou em 16: perguntar quais são as 4 extra (talvez elasticidade) e se podemos usar menos RAM.
- Os slides usam Ubuntu 18.04 e nós o 26.04: alguns comandos mudaram (`mariadb-secure-installation`, nomes `mariadb`/`mariadbd`).


# Falta fazer
1. HAProxy da haproxydb01: confirmar o bloco `listen mariadb_cluster`, validar com `haproxy -c`, reiniciar e testar a ligação à BD pelo IP virtual (ver 6.11).
2. Keepalived da haproxydb01: deixar ligado (`sudo systemctl start keepalived`) e confirmar o `.253` na `enp0s2` (foi parado nos testes de 6.9).
3. Confirmar que o `machine-id` da haproxydb01 é diferente do da db02 (ver 6.3).
4. Criar a haproxydb02 (`10.84.128.21`): mesmo caminho da Parte 3 e 4 e da Parte 6, com Keepalived em `state BACKUP` e `priority 100`; depois simular a falha parando o Keepalived da 01.
5. `/etc/hosts` com a mesma tabela em todas as VMs (db01, db02, haproxydb01) e corrigir a linha `127.0.1.1 ubuntu-base` onde ainda houver.
6. Confirmar nomes `db01`/`db02` em todo o lado (hostname, UTM, `~/bin/vms`, `~/.ssh/config`, Warp).
7. Mais tarde (trabalho prático): 3.º nó (db3, 10.84.128.13), atualizar `wsrep_cluster_address` nos 3 nós.
