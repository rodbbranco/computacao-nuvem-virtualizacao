# Documentação

Guia pela ordem em que as coisas se fazem. Para cada VM nova do trabalho prático: Parte 3 (criar a VM), Parte 4 (rede do cluster) e depois a parte do serviço dessa VM (ex.: Parte 5 para a BD). A Parte 7 é só consulta.

- Parte 1. Preparar o Mac
- Parte 2. VM modelo (`ubuntu-base`)
- Parte 3. Criar uma VM nova a partir do modelo
- Parte 4. Rede do cluster (IP fixo e nomes)
- Parte 5. Cluster de BD: MariaDB + Galera
- Parte 6. HAProxy + Keepalived à frente da BD (slides 30 a 41)
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
| haproxydb02 | (haproxydb2 no `/etc/hosts`) | 10.84.128.21 |
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
- Atenção ao `bind-address`: o ficheiro `/etc/mysql/mariadb.conf.d/50-server.cnf` traz `bind-address = 127.0.0.1` e é lido depois deste, por isso ganha. Sem corrigir, o Galera funciona mas o HAProxy não consegue ligar à porta 3306 (ver 6.12). Corrigir em todos os nós de BD:
```
sudo sed -i 's/^bind-address .*/bind-address = 0.0.0.0/' /etc/mysql/mariadb.conf.d/50-server.cnf
```
- `sed -i`: edita o ficheiro diretamente.
- `s/^bind-address .*/bind-address = 0.0.0.0/`: troca a linha que começa por `bind-address` por `bind-address = 0.0.0.0` (escutar em todos os IPs).

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

## 5.11 Arrancar o cluster quando nenhum nó é "seguro" (2026-10-21)
O que é e o que se faz: as VMs foram desligadas sem parar o MariaDB, por isso nenhum nó ficou marcado como o último a sair. Descobre-se qual tem os dados mais recentes e marca-se esse à mão.

Sintoma, na db01:
```
sudo galera_new_cluster
# Job for mariadb.service failed ...
sudo tail -n 30 /var/log/mariadb.log
# [ERROR] WSREP: It may not be safe to bootstrap the cluster from this node.
#   It was not the last one to leave the cluster ...
```
- O `systemctl status` e o `journalctl` cortam ou não mostram a causa; o erro verdadeiro está no `/var/log/mariadb.log` (o `log_error` do `galera.cnf`).

1. Ver o estado nos dois nós (MariaDB parado, sem o arrancar):
```
sudo cat /var/lib/mariadb/grastate.dat
```
- Resultado nos dois: `seqno: -1` e `safe_to_bootstrap: 0`. O `-1` quer dizer "não sei até onde cheguei" (paragem não limpa).

2. Ver até onde cada nó chegou:
```
sudo grep -h "Recovered position" /var/log/mariadb.log; sudo journalctl -u mariadb --no-pager | grep "Recovered position"
```
- `grep -h`: procura a frase, sem escrever o nome do ficheiro.
- `journalctl -u mariadb --no-pager`: registo do serviço, todo seguido.
- Só interessa a **última** linha de cada nó: `Recovered position 99233bbe-...:88`. O número do fim é a posição. Resultou `88` nos dois, ou seja, dados iguais.
- O `sudo mariadbd --wsrep-recover --user=mysql` não mostra nada no ecrã: o resultado vai para o log.

3. No nó escolhido (o de posição mais alta; empatados, tanto faz, escolheu-se a db01):
```
sudo sed -i 's/safe_to_bootstrap: 0/safe_to_bootstrap: 1/' /var/lib/mariadb/grastate.dat
sudo galera_new_cluster
```
- `sed -i 's/A/B/'`: troca o texto A por B no próprio ficheiro.
- O ficheiro certo é `/var/lib/mariadb/grastate.dat` (não `/var/lib/mysql/...`).

4. Nos outros nós: `sudo systemctl start mariadb` (nunca `galera_new_cluster`).

**Confirmado: cluster de volta com `wsrep_cluster_size` = 2.**

Para evitar isto: parar o MariaDB pela ordem da 5.10 antes de desligar as VMs.

## 5.12 Reiniciar um nó com o cluster a correr
O que é e o que se faz: aplicar uma mudança de configuração sem deitar o cluster abaixo, reiniciando um nó de cada vez.
```
sudo systemctl restart mariadb
sudo mariadb -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```
- Reiniciar um nó, esperar que o `wsrep_cluster_size` volte a 2 e só depois reiniciar o outro.
- A ordem não importa (a db01 foi reiniciada primeiro por engano e voltou ao cluster sem problemas), desde que nunca estejam os dois parados ao mesmo tempo.


# Parte 6. HAProxy + Keepalived à frente da BD (slides 30 a 41)
Aulas de 2026-10-12 (haproxydb01, Keepalived) e 2026-10-21 (HAProxy, haproxydb02, testes de falha). Objetivo: duas máquinas HAProxy (haproxydb1 `.17` e haproxydb2 `.21`) à frente do cluster de BD, para alta disponibilidade e balanceamento de carga. Os clientes (mais tarde o Moodle) falam só com o IP virtual `10.84.128.253`, que passa de um HAProxy para o outro se um falhar.

Resumo da cadeia (slide 40): IP virtual (Keepalived) → 2 HAProxy → nós MariaDB Galera. Para crescer (slide 41), acrescentam-se nós ao par Keepalived+HAProxy ou ao cluster de BD.

## 6.1 Criar a haproxydb01 (UTM)
O que é e o que se faz: uma VM nova clonada da `ubuntu-base`, com 512 MB, igual às outras (Parte 3).
1. `ubuntu-base` desligada e SanDisk ligado. Botão direito → **Clone**, nome `haproxydb01`.
2. **Show in Finder**: confirmar que ficou em `UTM/VMs` no SanDisk.
3. **Edit** → na placa existente, MAC **Random**; **New… → Network → Host Only**, também MAC **Random**.
4. **System** → memória **512 MB** → **Save**.

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
- `apache2`: servidor web. O slide 38 diz que é preciso "um webserver à escolha" para aceder às estatísticas do HAProxy. Na prática a página de estatísticas da 6.13 é servida pelo próprio HAProxy, por isso o papel do Apache fica a confirmar com o professor.
- `-y`: responde "sim" às perguntas.
- O slide 32 pede também o cliente MariaDB nas duas HAProxy (o slide escreve `mariadb-cliente`, com "e" a mais): `sudo apt-get install mariadb-client -y`. Serve para testar a ligação à BD a partir daqui (6.12).

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
- No fim dos testes, voltar a ligar: `sudo systemctl start keepalived`.

## 6.10 Engano: instalar `mailutils-mh` sem querer
O que é e o que se faz: pacote instalado por engano; remove-se com a configuração e as dependências que só ele puxou.
```
sudo apt-get purge mailutils-mh -y
sudo apt-get autoremove --purge -y
```
- `purge`: remove o pacote e os ficheiros de configuração dele.
- `autoremove --purge`: remove os pacotes que só estavam instalados por causa dele.
- Ler a lista antes do Enter: se aparecer `haproxy`, `keepalived` ou `apache2`, parar.

## 6.11 HAProxy na haproxydb01 (slides 36 a 38)
O que é e o que se faz: o HAProxy recebe as ligações à BD no IP virtual, porta 3306, e reparte-as pelos nós do Galera. Também mostra uma página de estatísticas. Organização do ficheiro (`global`, `defaults`, `frontend`, `backend`, `listen`) na 7.5.

1. Cópia de segurança do ficheiro original (o slide 38 pede-a em cada nó; fazer também na haproxydb02):
```
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
```
- `cp`: copia um ficheiro. O primeiro caminho é o original, o segundo o nome da cópia (`.bak` = backup).

2. Acrescentar no fim do ficheiro, sem apagar o que lá está:
```
sudo nano /etc/haproxy/haproxy.cfg
```
```
listen stats
    bind 10.84.128.17:1936
    mode http
    stats enable
    stats hide-version
    stats refresh 5s
    stats show-node
    stats auth admin:PASSWORD_DOS_STATS
    stats uri /stat

frontend srv-haproxydb
    bind 10.84.128.253:3306
    mode tcp
    option tcplog
    default_backend galera-cluster

backend galera-cluster
    mode tcp
    balance roundrobin
    server db1 10.84.128.11:3306 check
    server db2 10.84.128.16:3306 check
    server db3 10.84.128.13:3306 check
```
- `listen stats`: página web de estatísticas. `bind 10.84.128.17:1936` escuta no IP próprio da máquina, porta 1936.
- `stats enable`: liga a página. `hide-version`: esconde a versão do HAProxy. `refresh 5s`: atualiza de 5 em 5 segundos. `show-node`: mostra o nome da máquina.
- `stats auth admin:PASSWORD_DOS_STATS`: login da página (utilizador:password). A password verdadeira não fica escrita aqui.
- `stats uri /stat`: endereço da página.
- `frontend srv-haproxydb`: a porta de entrada. `bind 10.84.128.253:3306` escuta no **IP virtual**, na porta do MariaDB.
- `mode tcp`: passa as ligações sem as ler (o MariaDB não é HTTP).
- `option tcplog`: formato de log para TCP. Sem esta linha aparece o aviso `'option httplog' not usable with frontend 'srv-haproxydb' (needs 'mode http')`, porque o `defaults` traz `option httplog`. É só um aviso: o HAProxy passa sozinho para `tcplog`.
- `default_backend galera-cluster`: manda tudo para o grupo `galera-cluster`.
- `balance roundrobin`: cada ligação **nova** vai para o servidor seguinte, à vez.
- `server db1 10.84.128.11:3306 check`: nome, IP e porta de cada nó. `check` testa se está vivo e deixa de lhe mandar ligações se falhar. A db3 ainda não existe: fica DOWN e é ignorada.
- Diferença em relação ao slide 38: o slide tem `option httpchk` no backend. Esse teste faz um pedido web à porta 3306, onde só responde o MariaDB, por isso as BD deviam ficar todas DOWN. Ficou só `check`. A confirmar com o professor.

3. Validar, reiniciar e ver as portas:
```
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy && sudo ss -tlnp | grep haproxy
```
- `haproxy -c -f ficheiro`: só verifica a configuração (`-c` = check; `-f` = ficheiro).
- `restart`: pára e volta a arrancar o serviço, para ler o ficheiro novo.
- `ss -tlnp`: portas à escuta (`t` TCP, `l` à escuta, `n` números, `p` programa). `| grep haproxy`: só as do HAProxy.
- Resultado: `10.84.128.253:3306` e `10.84.128.17:1936`. **Confirmado.**

## 6.12 Testar a ligação à BD pelo IP virtual (slide 39)
O que é e o que se faz: ligar ao MariaDB através do `.253` e ver qual nó responde. O `root` da BD só entra a partir da própria máquina, por isso cria-se um utilizador só para este teste.

1. Na db01 (o Galera copia-o para a db02):
```
sudo mariadb -u root -p
```
```
CREATE USER 'teste'@'10.84.128.%' IDENTIFIED BY 'PASSWORD_DE_TESTE';
GRANT USAGE ON *.* TO 'teste'@'10.84.128.%';
exit
```
- `CREATE USER 'teste'@'10.84.128.%'`: utilizador `teste`, que só pode entrar a partir da rede 10.84.128.x (`%` = qualquer).
- `IDENTIFIED BY`: a password.
- `GRANT USAGE`: só o direito de ligar, sem acesso a dados.
- Para mudar a password: `ALTER USER 'teste'@'10.84.128.%' IDENTIFIED BY 'nova';`.

2. Na haproxydb01, repetir várias vezes:
```
mariadb -h 10.84.128.253 -u teste -p -e "SELECT @@hostname;"
```
- `-h 10.84.128.253`: liga a esse IP (o virtual) em vez da própria máquina.
- `SELECT @@hostname`: pergunta o nome da máquina que respondeu.
- Resultado: alterna entre `db01` e `db02` (roundrobin). **Confirmado.**

### Problema: `ERROR 2013 ... Lost connection to server at 'handshake: reading initial communication packet'`
- Significado: o HAProxy aceitou a ligação mas não conseguiu falar com nenhuma BD por trás.
- Primeira causa: o cluster estava em baixo (VMs desligadas sem parar o MariaDB). Resolvido com a 5.11.
- Segunda causa: o MariaDB só escutava em `127.0.0.1`.
  - Separar o problema, ligando diretamente a um nó sem passar pelo HAProxy: `mariadb -h 10.84.128.11 -u teste -p -e "SELECT @@hostname;"` dava `ERROR 2002 ... (115)`.
  - `ping -c 2 10.84.128.11; nc -zv -w 3 10.84.128.11 3306`: o ping respondia mas o `nc` dava `Connection refused` (a máquina está viva mas ninguém escuta nessa porta). `-w 3` = desistir ao fim de 3 segundos.
  - Na db01, `sudo ss -tlnp | grep -E "3306|mariadb"` mostrava `127.0.0.1:3306`, e `sudo grep -rn "bind-address" /etc/mysql/` mostrou o `50-server.cnf` com `127.0.0.1`.
  - Correção: o `sed` da 5.4 nas duas BD e reiniciar uma de cada vez (5.12). Depois, `ss` mostra `0.0.0.0:3306` nas duas. **Confirmado.**
- Uma password errada daria `Access denied`, não estes erros.

## 6.13 Página de estatísticas
O que é e o que se faz: confirmar que a página do `listen stats` responde (o slide 39 mostra-a com as sessões repartidas por db1, db2 e db3).
```
curl -u admin:PASSWORD_DOS_STATS http://10.84.128.17:1936/stat | head
```
- `curl`: pede uma página web a partir do terminal.
- `-u utilizador:password`: o login do `stats auth`. Com caracteres especiais, a password vai entre aspas.
- `http://10.84.128.17:1936/stat`: IP próprio da máquina, porta e caminho da página.
- `| head`: só as primeiras 10 linhas. O `curl: Failed writing body` do fim é normal (o `head` fechou).
- Resultado: HTML com `Statistics Report for HAProxy on haproxydb01`. **Confirmado.** Vista no browser do Mac: por testar (o Mac pode não chegar à rede 10.84.128.x, ver 6.9).

## 6.14 Criar a haproxydb02 (.21)
O que é e o que se faz: a segunda máquina HAProxy, igual à haproxydb01 (6.1 a 6.5), com o IP `10.84.128.21`.
- Clonar da `ubuntu-base`, MAC novo, `machine-id` novo, hostname `haproxydb02`, chaves SSH novas, MAC no `00-installer-config.yaml`, `60-cluster.yaml` com `10.84.128.21/24` (Partes 3 e 4).
- Confirmar tudo de uma vez:
```
hostname; ip -br a; cat /etc/machine-id
```
  - `;` corre os comandos seguidos. O `machine-id` tem de ser diferente do da haproxydb01.
- Instalar: `sudo apt-get update && sudo apt-get install haproxy keepalived apache2 mariadb-client -y` (`&&` só corre o segundo se o primeiro correr bem).
- `/etc/hosts` usado na haproxydb02:
```
127.0.1.1 haproxydb02
...
10.84.128.11 db01
10.84.128.16 db02
10.84.128.13 db03
10.84.128.17 haproxydb01
10.84.128.21 haproxydb02
```
  - Os nomes das BD aqui (`db01`) não são os mesmos da haproxydb01 (`db1`). Funciona, porque a configuração usa IPs, mas fica por uniformizar (ver Falta fazer).

### Problema: `ping haproxydb01` respondia a partir de `127.0.1.1`
- Sintoma: `PING haproxydb01 (127.0.1.1)`, com 0,08 ms. A máquina estava a falar consigo própria.
- Causa: a linha `127.0.1.1` do `/etc/hosts` tinha ficado com `haproxydb01`.
- Correção: `127.0.1.1 haproxydb02`. Ver o ficheiro com números de linha: `grep -n "" /etc/hosts`.

### Problema: ping à haproxydb01 rápido demais
- Sintoma: depois de corrigir o `/etc/hosts`, o ping ao `10.84.128.17` dava 0,08 ms. Entre duas VMs é normal 0,5 ms ou mais.
- Causa provável: a `enp0s2` da haproxydb02 ainda tinha o `.17`. Ver com `ip -br a show enp0s2`, corrigir o `60-cluster.yaml` para `.21` e `sudo netplan apply`.
- Depois da correção, o ping deu 1,4 ms e 12 ms. **Confirmado.**

### O "cluster" dos HAProxy
- Não é como o Galera: as duas HAProxy não trocam dados. São duas máquinas com a mesma configuração, feita à mão em cada uma, e o Keepalived decide qual tem o IP virtual.
- O nome `60-cluster.yaml` não tem nada a ver com o Galera: é só o ficheiro do IP fixo da rede interna, que todas as VMs têm.

## 6.15 Keepalived na haproxydb02 (BACKUP, slides 33 e 34)
O que é e o que se faz: a mesma configuração da 6.7, mas como reserva.
```
sudo nano /etc/keepalived/keepalived.conf
```
```
vrrp_instance VI_DB {
    state BACKUP
    interface enp0s2
    virtual_router_id 53
    priority 100
    advert_int 1
    virtual_ipaddress {
        10.84.128.253/24
    }
}
```
- `state BACKUP`: fica à espera, sem o `.253`.
- `priority 100`: abaixo dos 101 da haproxydb01, por isso a 01 ganha enquanto estiver viva. Se o MASTER tiver prioridade mais baixa que o BACKUP, perde o estatuto (slide 33).
- `virtual_router_id 53`: igual nas duas, senão não se reconhecem.
- Arrancar: `sudo systemctl enable --now keepalived`.
- Confirmar: `ip -br a show enp0s2` mostra só o `.21`. **Confirmado.**

## 6.16 Teste de falha do IP virtual (slide 35)
O que é e o que se faz: parar o Keepalived da 01 e ver o `.253` passar para a 02.
1. haproxydb01: `ip -br a show enp0s2` → `.17` e `.253`.
2. haproxydb01: `sudo systemctl stop keepalived` → fica só com o `.17`.
3. haproxydb02, uns segundos depois: `ip -br a show enp0s2` → `.21` e `.253`.
4. haproxydb01: `sudo systemctl start keepalived` → recupera o `.253` (prioridade maior) e a 02 volta a ficar só com o `.21`.

**Confirmado.**

## 6.17 Deixar o HAProxy escutar num IP que a máquina ainda não tem
O que é e o que se faz: na haproxydb02 o `.253` só existe quando ela é MASTER. Sem isto, o HAProxy recusa o `bind 10.84.128.253:3306` e não arranca. Faz-se nas duas HAProxy, para tudo arrancar depois de um failover ou de um reboot.
```
echo "net.ipv4.ip_nonlocal_bind = 1" | sudo tee /etc/sysctl.d/99-haproxy.conf
sudo sysctl --system
sysctl net.ipv4.ip_nonlocal_bind
```
- `echo "..." | sudo tee ficheiro`: escreve o texto no ficheiro com permissões de administrador (com `sudo echo > ficheiro` não funciona, porque o `>` corre sem sudo).
- `/etc/sysctl.d/`: definições do kernel lidas em cada arranque.
- `ip_nonlocal_bind = 1`: permite escutar em IPs que a máquina não tem.
- `sysctl --system`: aplica já todas as definições, sem reiniciar.
- `sysctl net.ipv4.ip_nonlocal_bind`: mostra o valor. Resultado: `= 1` nas duas. **Confirmado.**

## 6.18 HAProxy na haproxydb02
O que é e o que se faz: a mesma configuração da 6.11, com uma diferença: no `listen stats`, `bind 10.84.128.21:1936` (o IP próprio da 02). A password dos stats é a mesma nas duas.
```
sudo haproxy -c -f /etc/haproxy/haproxy.cfg && sudo systemctl restart haproxy && sudo ss -tlnp | grep haproxy
```
- Resultado: à escuta em `10.84.128.21:1936` e `10.84.128.253:3306`, mesmo sem ter o IP virtual (graças à 6.17). **Confirmado.**

## 6.19 Teste final: cai uma HAProxy e a BD continua acessível (slide 40)
O que é e o que se faz: simular a queda da haproxydb01 e ligar à BD pelo mesmo IP virtual.
1. haproxydb01: `sudo systemctl stop keepalived haproxy` (pára os dois serviços).
2. haproxydb02: `ip -br a show enp0s2` → `.21` e `.253`.
3. haproxydb02: `mariadb -h 10.84.128.253 -u teste -p -e "SELECT @@hostname;"`, várias vezes → alterna entre `db01` e `db02`.
4. haproxydb01: `sudo systemctl start haproxy keepalived` → as ligações continuam a funcionar.

**Confirmado: slides 30 a 41 concluídos, com 2 nós de BD (a db3 fica para o trabalho prático).**


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

## 7.5 Organização do `haproxy.cfg` (slides 36 e 37)
- `global`: regras para o HAProxy inteiro. Ex.: `maxconn 50000` (máximo de ligações, protege a memória), `log` (para onde vão os logs).
- `defaults`: valores aplicados a todos os frontends e backends, para não os repetir. Ex.: `mode http` (nível 7, aplicação: percebe páginas web) ou `mode tcp` (nível 4, transporte: só passa os dados); `option forwardfor` (passa aos servidores o IP do cliente).
- `frontend`: onde os clientes ligam. `bind IP:porta` (onde escuta) e `default_backend` (para que grupo manda). Pode ter o seu próprio `mode`, que se sobrepõe ao do `defaults`.
- `backend`: o grupo de servidores que trata os pedidos. `balance` (algoritmo) e uma linha `server nome IP:porta` por servidor. O nome tem de ser igual ao do `default_backend` do frontend.
- `listen`: junta frontend e backend num só bloco. Mais curto, mas mais difícil de ler (o professor prefere separar).
- Algoritmos de `balance`: `roundrobin` (à vez, o do slide), `leastconn` (para o servidor com menos ligações abertas), `source` (o mesmo cliente vai sempre para o mesmo servidor).
- O HAProxy suporta TCP e HTTP. Para a BD usa-se `mode tcp`.


# Falta fazer
1. `/etc/hosts` com a mesma tabela em todas as VMs (db01, db02, haproxydb01, haproxydb02): hoje a haproxydb01 usa `db1`/`db2`/`db3` e a haproxydb02 `db01`/`db02`/`db03`. Escolher um e acrescentar `haproxydb02` na linha do `.21` das outras. Corrigir a linha `127.0.1.1 ubuntu-base` onde ainda houver.
2. Confirmar nomes `db01`/`db02` em todo o lado (hostname, UTM, `~/bin/vms`, `~/.ssh/config`, Warp).
3. Apagar o utilizador de teste da BD (na db01: `DROP USER 'teste'@'10.84.128.%';`).
4. Ver a página de estatísticas no browser do Mac, se o Mac chegar à rede 10.84.128.x (ver 6.9 e 6.13).
5. Perguntar ao professor: para que serve o Apache nas HAProxy (6.5) e se o `option httpchk` do slide 38 é para usar (6.11).
6. Mais tarde (trabalho prático): 3.º nó (db3, 10.84.128.13), atualizar `wsrep_cluster_address` nos 3 nós. O HAProxy já tem a linha da db3.
7. Próxima aula: nós web (slide 42: srv-moodleweb1/2/3, .24, .26, .27).
