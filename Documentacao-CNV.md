# Documentação

Guia pela ordem em que as coisas se fazem. Para cada VM nova do trabalho prático: Parte 3 (criar a VM), Parte 4 (rede do cluster) e depois a parte do serviço dessa VM (ex.: Parte 5 para a BD). A Parte 6 é só consulta.

- Parte 1. Preparar o Mac
- Parte 2. VM modelo (`ubuntu-base`)
- Parte 3. Criar uma VM nova a partir do modelo
- Parte 4. Rede do cluster (IP fixo e nomes)
- Parte 5. Cluster de BD: MariaDB + Galera
- Parte 6. Consulta
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


# Parte 6. Consulta

## 6.1 O que quer dizer `mysql -u root -p -e "..."`
- `-u root`: entrar como o utilizador `root` **da base de dados** (não é o root do Ubuntu; o MariaDB tem utilizadores próprios).
- `-p`: pede a password (a do `mariadb-secure-installation`), sem a escrever no comando.
- `-e "..."`: executa essa instrução SQL e sai.

## 6.2 Conceitos do `galera_new_cluster`
- É um comando (script do MariaDB), não sintaxe. Liga o interruptor `_WSREP_NEW_CLUSTER` e arranca o MariaDB: cria o cluster de raiz a partir deste nó.
- O nome do cluster (`moodle_galera_cluster`) vem do `galera.cnf` (`wsrep_cluster_name`), escolhido por nós.
- Só se corre quando o cluster tem de nascer de novo: criação, ou quando **todos** os nós estavam parados. Se um nó avaria e os outros estão vivos, basta `sudo systemctl start mariadb` nesse nó.
- Perigo: correr `galera_new_cluster` com outros nós ativos cria clusters separados (split-brain).

## 6.3 Permissões Linux (resumo)
- r=4 (ler), w=2 (escrever), x=1 (executar/entrar na pasta). Ordem: dono, grupo, outros.
- 600 = só o dono lê/escreve · 644 = dono lê/escreve, resto lê · 755 = dono tudo, resto lê/executa.
- `ls -l`: `-rw-r--r--` = ficheiro, dono `rw-`, grupo `r--`, outros `r--` (644).

## 6.4 Recursos (RAM) e número de VMs
- Os slides indicam o dimensionamento original (BD 3x16 GB, HAProxy BD 2x4 GB, Web 3x12 GB, NFS 2x8 GB, Ubuntu 18.04; ~110 GB no total). Não é para copiar num portátil de 16 GB.
- Plano reduzido: BD 1 GB, HAProxy e NFS 512 MB, Web 1 GB = ~9 GB para as 12 VMs; **não ligar todas ao mesmo tempo** enquanto se constrói (macOS precisa de 4 a 6 GB). Vigiar a pressão de memória no Activity Monitor (verde ok, amarelo limite, vermelho demais).
- Os slides mostram 12 VMs mas o professor falou em 16: perguntar quais são as 4 extra (talvez elasticidade) e se podemos usar menos RAM.
- Os slides usam Ubuntu 18.04 e nós o 26.04: alguns comandos mudaram (`mariadb-secure-installation`, nomes `mariadb`/`mariadbd`).


# Falta fazer
1. Mais tarde (trabalho prático): 3.º nó (db3, 10.84.128.13), actualizar `wsrep_cluster_address` nos 3 nós, depois HAProxy + Keepalived à frente da BD.
