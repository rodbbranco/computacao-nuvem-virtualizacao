# Documentação

## 1. Preparar o disco
1. Reformatar o SanDisk para APFS.
2. Criar a pasta `UTM` e, dentro, `ISOs` (com o ISO do Ubuntu).

## 2. VM modelo (`ubuntu-base`)
4. No UTM, criar uma VM com 12 GB de disco e instalar o Ubuntu Server 26.04 ARM64.
5. Na instalação usar 2 GB de RAM (com 1 GB o teclado perde teclas); depois voltar a 1 GB.
6. Instalar o `openssh-server` durante a instalação do Ubuntu (selecionar na lista de pacotes) ou depois com `sudo apt install openssh-server`.
7. Atualizar o sistema: `sudo apt update && sudo apt upgrade -y`.
8. Limpar o `/etc/machine-id` para que cada clone gere um ID único: `sudo truncate -s 0 /etc/machine-id` (será regenerado no próximo boot).
8. Desligar a `ubuntu-base`: só serve para clonar.

## 3. Primeira clone (`srv01`)
9. Botão direito na `ubuntu-base` desligada, Clone, nome `srv01`.
10. Antes de ligar, gerar um MAC aleatório em Network (o UTM não o muda sozinho).
11. Mudar o nome com `sudo hostnamectl set-hostname srv01`.
12. Gerar chaves SSH novas da máquina: `sudo rm /etc/ssh/ssh_host_* && sudo dpkg-reconfigure openssh-server`, depois `sudo reboot`.
13. Confirmar o IP com `ip a`.

## 4. SSH com o Warp
14. Ligar à `srv01` por SSH no Warp (`ssh rodrigo@192.168.64.X`).
15. O Warp mostra a árvore de ficheiros da VM e cria `.warp` e `.config/warp-terminal` na VM.

## 5. Controlar as VMs pelo terminal
16. Criar o script `~/bin/vms` (usa o `utmctl`) com `vms up`, `vms down`, `vms ls`.
17. Acrescentar `~/bin` ao PATH no `~/.zshrc`.
18. O `--hide` do `utmctl` só esconde a janela principal do UTM, por isso o script também esconde o UTM com `osascript`.
19. O UTM tem de ficar aberto (se o fechar as VMs param) e o SanDisk ligado.

## 6. Login por chave SSH
20. A chave pública é mais segura e cómoda que a password.
21. Se já existirem chaves no Mac (`id_ed25519` e `id_rsa`), responder n ao `ssh-keygen` para não as substituir.
22. `ssh-copy-id rodrigo@192.168.64.6` para pôr a chave na `srv01`.
23. `ssh srv01` e `ssh ubuntu-base` entram sem pedir a password da VM.
24. Sair de uma sessão sem desligar a VM: `exit`, Ctrl+D ou fechar o tab.

## Por fazer
- Confirmar com `ssh -o PasswordAuthentication=no srv01` que só a chave entra.
- Desligar o login por password (`/etc/ssh/sshd_config.d/00-no-password.conf`), com a janela do UTM como plano B.
- Pôr a chave na `ubuntu-base` antes de criar as restantes clones (`srv02` a `srv16`).
- Atalhos no `~/.ssh/config` e Launch Configuration do Warp.


# Percurso da aula, 2026-10-06 (Cluster de BD com MariaDB + Galera)

Objetivo da aula: slides 24 a 29 do PDF `Cluster_HA_LOAD_B_CLOUD.pdf`. Hoje só 2 nós (db01, db02); o 3.º fica para o trabalho prático.

## Conceitos
- **Nó** = cada máquina do cluster (cada VM de BD).
- **Galera** = extra do MariaDB que mantém os dados iguais em todos os nós (escreves num, aparece nos outros).
- **Placa de rede** = ligação de uma máquina a uma rede. Na VM é emulada pelo UTM.
- Cada VM tem 2 placas: `enp0s1` = NAT (Internet, updates, SSH, IP automático 192.168.64.x); `enp0s2` = Host Only (rede privada do cluster, IP fixo 10.84.128.x). No fim desliga-se a NAT.
- `/etc/netplan/` = dar IP fixo à placa. `/etc/hosts` = tabela nome → IP (sem DNS). São duas coisas diferentes.

## IPs escolhidos (do slide 25)
| VM | Nome | IP cluster (enp0s2) |
|---|---|---|
| db01 | db1 | 10.84.128.11 |
| db02 | db2 | 10.84.128.16 |
| (3.ª, mais tarde) | db3 | 10.84.128.13 |

## Passos feitos

### 1. Segunda placa de rede (UTM, VM desligada), em cada VM
1. Botão direito na VM → **Edit**.
2. Devices → **New…** → **Network**.
3. Network Mode = **Host Only** → **Save**.
4. A placa antiga (Shared Network) fica como está.

### 2. Ver as placas (dentro da VM)
```
ip -br a      # placas e IPs (-br = uma linha por placa; a = address)
ip -br link   # placas e MAC real de cada uma
```

### 3. Ficheiro novo do netplan com o IP fixo da `enp0s2`
```
sudo nano /etc/netplan/60-cluster.yaml
```
Conteúdo (db01; na db02 troca o IP por 10.84.128.16/24):
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

### 4. Permissões do ficheiro
```
sudo chmod 600 /etc/netplan/60-cluster.yaml   # só o dono lê/escreve (o netplan exige)
```

### 5. Problema: `netplan apply` deu "Cannot find unique matching interface for enp0s1"
- Causa: o ficheiro antigo `/etc/netplan/00-installer-config.yaml` identifica a placa NAT pelo MAC do `ubuntu-base`; a clone tem MAC novo.
- Ver o MAC real: `ip -br link` (linha `enp0s1`).
- Corrigir:
```
sudo nano /etc/netplan/00-installer-config.yaml
```
  e trocar a linha `macaddress:` pelo MAC real da `enp0s1` dessa VM:
  - db01: `7e:ac:26:70:d7:d4`
  - db02: `02:d8:35:26:0d:d7`

### 6. Aplicar e confirmar
```
sudo netplan apply
ip -br a
```
- A SSH cai e a NAT muda de IP (db01: 192.168.64.6 → .5). Esperar uns 20 s e ligar ao IP novo: `ssh rodrigo@192.168.64.5`.
- `enp0s2` deve mostrar `10.84.128.11/24` (db01) ou `10.84.128.16/24` (db02).
- **db01: feito e confirmado (NAT 192.168.64.5). db02: feito e confirmado, `enp0s2` = 10.84.128.16/24 (NAT passou a 192.168.64.3).**

### 7. Aviso SSH "REMOTE HOST IDENTIFICATION HAS CHANGED"
- Causa: o IP da NAT (192.168.64.3) já tinha sido de outra máquina (ou a db02 mudou de chaves SSH); o Mac guardou a chave antiga em `~/.ssh/known_hosts`.
- Correção, **no Mac** (não dentro da VM): `ssh-keygen -R 192.168.64.3` e depois `ssh rodrigo@192.168.64.3` (responder `yes` à pergunta da impressão digital).

### 8. Instalar o MariaDB (nas duas VMs, painéis sincronizados)
```
sudo apt-get update
sudo apt-get install apache2 mariadb-server rsync -y
```
Verificar: `mariadb --version`, `systemctl status mariadb`, `dpkg -l | grep -E 'mariadb-server|apache2|rsync'` (`ii` = instalado).

### 9. Segurança do MariaDB (slide 26)
- O comando agora chama-se `sudo mariadb-secure-installation` (antes `mysql_secure_installation`).
- Respostas: Enter nas duas perguntas da password atual; unix_socket = n; mudar password de root = Y (mesma nas duas VMs); remover anónimos Y; root remoto não Y; remover base de teste Y; recarregar Y.
- Utilizador do cluster (slide 26): não é preciso com `wsrep_sst_method=rsync`.

### 10. Preparar o Galera (nas duas VMs)
```
ls -l /usr/lib/galera/                      # confirmar libgalera_smm.so
sudo touch /var/log/mariadb.log
sudo chown mysql:mysql /var/log/mariadb.log
sudo systemctl stop mariadb
```

### 11. Ficheiro `/etc/mysql/conf.d/galera.cnf` (em cada VM, em separado)
Tem de acabar em `.cnf` (o MariaDB ignora outras extensões; se te enganares, `sudo mv galera.dnf galera.cnf`).
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

### 12. Firewall (ufw ativa na db01)
```
sudo ufw allow from 10.84.128.0/24 to any port 4567
sudo ufw allow from 10.84.128.0/24 to any port 4568 proto tcp
sudo ufw allow from 10.84.128.0/24 to any port 4444 proto tcp
sudo ufw allow from 10.84.128.0/24 to any port 3306 proto tcp
```
Sintoma sem isto: a db02 fica a tentar ligar (`wsrep::connect ... failed: 7`; `nc -zv IP 4567` fica preso).

### 13. AppArmor (nas duas VMs)
- Sintoma: db02 entra no cluster mas rebenta (SEGV) com `posix_spawnp(wsrep_sst_rsync) failed: 13 (Permission denied)`; o kernel regista `apparmor="DENIED" ... name="/usr/bin/dash"`.
- Solução do laboratório:
```
sudo apt-get install -y apparmor-utils
sudo aa-complain mariadbd      # com d no fim
```

### 14. Arrancar o cluster (slide 28)
```
sudo galera_new_cluster        # SÓ no 1.º nó (db01)
sudo systemctl start mariadb   # no 2.º nó (db02)
mysql -u root -p -e "show status like 'wsrep_cluster_size'"   # deve dar 2
```
**Resultado: cluster com 2 nós a funcionar.**

### 15. Teste de replicação (slide 29)
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

### 16. O que quer dizer `mysql -u root -p -e "..."`
- `-u root`: entrar como o utilizador `root` **da base de dados** (não é o root do Ubuntu; o MariaDB tem utilizadores próprios).
- `-p`: pede a password (a do `mariadb-secure-installation`), sem a escrever no comando.
- `-e "..."`: executa essa instrução SQL e sai.

### 17. Conceitos do `galera_new_cluster`
- É um comando (script do MariaDB), não sintaxe. Liga o interruptor `_WSREP_NEW_CLUSTER` e arranca o MariaDB: cria o cluster de raiz a partir deste nó.
- O nome do cluster (`moodle_galera_cluster`) vem do `galera.cnf` (`wsrep_cluster_name`), escolhido por nós.
- Só se corre quando o cluster tem de nascer de novo: criação, ou quando **todos** os nós estavam parados. Se um nó avaria e os outros estão vivos, basta `sudo systemctl start mariadb` nesse nó.
- Perigo: correr `galera_new_cluster` com outros nós ativos cria clusters separados (split-brain).
- WSREP = Write-Set REPlication (a replicação do Galera).

### 18. Como ver se o cluster iniciou (dentro da VM)
```
mysql -u root -p -e "show status where Variable_name in ('wsrep_cluster_size','wsrep_cluster_status','wsrep_ready')"
sudo ss -tlnp | grep 4567       # o Galera está à escuta
```
Saudável: `wsrep_cluster_size` = nº de nós (2), `wsrep_cluster_status` = `Primary`, `wsrep_ready` = `ON`.

### 19. Desligar e voltar a ligar o cluster
- Desligar: `sudo systemctl stop mariadb` primeiro na **db02**, depois na **db01** (a última); só depois desligar as VMs.
- Ligar: ligar as VMs; na **db01** (a última que parou) `sudo galera_new_cluster`; na **db02** `sudo systemctl start mariadb`; confirmar `wsrep_cluster_size` = 2.
- Regra: **o último a desligar é o primeiro a arrancar** com `galera_new_cluster`.
- Se falhar com "safe to bootstrap": `sudo cat /var/lib/mariadb/grastate.dat`; o nó com `safe_to_bootstrap: 1` é o que deve arrancar primeiro.

### 20. Recursos (RAM) e número de VMs
- Os slides indicam o dimensionamento original (BD 3x16 GB, HAProxy BD 2x4 GB, Web 3x12 GB, NFS 2x8 GB, Ubuntu 18.04; ~110 GB no total). Não é para copiar num portátil de 16 GB.
- Plano reduzido: BD 1 GB, HAProxy e NFS 512 MB, Web 1 GB = ~9 GB para as 12 VMs; **não ligar todas ao mesmo tempo** enquanto se constrói (macOS precisa de 4 a 6 GB). Vigiar a pressão de memória no Activity Monitor (verde ok, amarelo limite, vermelho demais).
- Os slides mostram 12 VMs mas o professor falou em 16: perguntar quais são as 4 extra (talvez elasticidade) e se podemos usar menos RAM.
- Os slides usam Ubuntu 18.04 e nós o 26.04: alguns comandos mudaram (`mariadb-secure-installation`, nomes `mariadb`/`mariadbd`).

## Permissões Linux (resumo)
- r=4 (ler), w=2 (escrever), x=1 (executar/entrar na pasta). Ordem: dono, grupo, outros.
- 600 = só o dono lê/escreve · 644 = dono lê/escreve, resto lê · 755 = dono tudo, resto lê/executa.
- `ls -l`: `-rw-r--r--` = ficheiro, dono `rw-`, grupo `r--`, outros `r--` (644).

## Falta fazer
1. (feito) Teste de replicação do slide 29.
2. Confirmar nomes `db01`/`db02` em todo o lado (hostname, UTM, `~/bin/vms`, `~/.ssh/config`, Warp).
3. Mais tarde (trabalho prático): 3.º nó (db3, 10.84.128.13), actualizar `wsrep_cluster_address` nos 3 nós, depois HAProxy + Keepalived à frente da BD.


### O que fazer quando:
``ssh rodrigo@192.168.64.3``

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

``ssh-keygen -R 192.168.64.3``