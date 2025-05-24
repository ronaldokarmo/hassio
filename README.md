# 🏠 Tutorial: Instalando o Home Assistant no Orange Pi 5

Este tutorial ensina, passo a passo, como instalar o sistema operacional necessário para rodar o **Home Assistant** no dispositivo **Orange Pi 5**, incluindo dicas úteis e correções no processo de instalação.

> ⚠️ **Atenção:** Este guia assume que você está utilizando um computador com **Windows** para preparar a imagem do sistema e que você possui conhecimento básico de terminal e redes.

---

## 🧰 Pré-requisitos

* Orange Pi 5 com cartão SD ou SSD/HD conectado
* Leitor de cartão SD (caso utilize SD)
* Computador com Windows/Linux
* Acesso à internet
* Balena Etcher instalado ([link](https://etcher.io))

---

## 📥 Passo 1: Baixando e preparando o sistema

1. **Baixe a imagem do sistema Operacional**: [Imagem Debian Bullseye](http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/service-and-support/Orange-pi-5.html)

2. **Verifique o hash da imagem (Windows)**:

```powershell
certUtil -hashfile "C:\Users\USER\Downloads\Orangepi5_1.2.0_debian_bullseye_server_linux5.10.160.img" SHA256
```

3. **Grave a imagem no cartão SD ou HD/SSD usando o Balena Etcher**:

> 💡 **Dica:** Verifique se o dispositivo está corretamente selecionado no Balena Etcher para evitar sobrescrever dados importantes.

---

## 🔌 Passo 2: Primeira inicialização e atualização do sistema

Conecte o Orange Pi 5 na energia, aguarde o boot e conecte via SSH:

```bash
ssh orangepi@192.168.15.15
sudo su -
```

### Atualize o sistema

```bash
apt update && apt upgrade -y
sudo apt list --upgradable
apt dist-upgrade -y
lsb_release -a
```

### Faça o upgrade do Debian 11 (Bullseye) para o Debian 12 (Bookworm)

```bash
sed -i 's/bullseye/bookworm/g' /etc/apt/sources.list
apt update && apt full-upgrade -y
reboot
```

> 💡 **Dica:** Após o reboot, conecte novamente via SSH e continue com os comandos.

---

## 🧱 Passo 3: Instalação de pacotes essenciais

```bash
sudo apt install apparmor jq wget curl udisks2 libglib2.0-bin network-manager dbus lsb-release systemd-journal-remote -y
```

### Configure o cgroup

```bash
mkdir /boot/firmware
sudo touch /boot/firmware/cmdline.txt
sudo vi /boot/firmware/cmdline.txt
chmod 777 /boot/firmware/cmdline.txt
```

Adicione as linhas:

```
cgroup_enable=cpuset
cgroup_enable=memory
cgroup_memory=1
```

### Instale o "equivs" e crie pacote para systemd-resolved

```bash
sudo apt install equivs

# Crie e edite o controle
equivs-control systemd-resolved.control
sed -i 's/<package name; defaults to equivs-dummy>/systemd-resolved/g' systemd-resolved.control

# Construa e instale o pacote
equivs-build systemd-resolved.control
sudo dpkg -i systemd-resolved_1.0_all.deb
```

### Atualize o boot

```bash
echo "extraargs=apparmor=1 security=apparmor systemd.unified_cgroup_hierarchy=false systemd.legacy_systemd_cgroup_controller=false" >> /boot/orangepiEnv.txt
sudo update-initramfs -u
sudo reboot
```

---

## 🐳 Passo 4: Instalando Docker e Home Assistant

```bash
sudo su -
curl -fsSL get.docker.com | sh
```

### Instale os agentes necessários

```bash
wget https://github.com/home-assistant/os-agent/releases/download/1.7.2/os-agent_1.7.2_linux_aarch64.deb
dpkg -i os-agent_1.7.2_linux_aarch64.deb

wget https://github.com/home-assistant/supervised-installer/releases/latest/download/homeassistant-supervised.deb
dpkg -i homeassistant-supervised.deb
apt --fix-broken install
```

> ⚠️ **NOTA:** Use este comando para instalar o Home Assistant ignorando a verificação de OS

```bash
BYPASS_OS_CHECK=true apt install ./homeassistant-supervised.deb
```

### Reinicie o supervisor (se necessário)

```bash
ha supervisor restart
ha jobs reset
```

> ⚠️ **NOTA:** Pode ser necessário ajustar o arquivo `/boot/orangepiEnv.txt` com os seguintes parâmetros:

```bash
verbosity=1
bootlogo=false
extraargs=cma=128M
overlay_prefix=rk3588
fdtfile=rockchip/rk3588s-orangepi-5.dtb
rootdev=UUID=<sua UUID>
rootfstype=ext4
```

> 💡 Use `sudo blkid` para identificar a UUID correta do seu disco.

---

## 🧭 Boot via USB (opcional)

```bash
sudo dd if=/usr/lib/linux-u-boot-current-orangepi5_1.1.8_arm64/u-boot.itb of=/dev/sda bs=1024 seek=8
```

Vídeos de referência:

* [Boot USB #1](https://www.youtube.com/watch?v=kSSh8ADPsTw&t=41s)
* [Boot USB #2](https://www.youtube.com/watch?v=Z5-sZt_O51w&t=2s)

---

## 🧑‍💻 Acesso e configuração do sistema

### Acesso via SSH

```bash
ssh root@192.168.15.15
ssh orangepi@192.168.15.15
```

### Alterar senha

```bash
passwd
passwd orangepi
```

### Listar usuários

```bash
users
```

---

### 📌 Tabela de Referência: Paths, Usuários e IPs

| **Item**                               | **Valor/Exemplo**                                                                                                   |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| **IP do Orange Pi 5**                  | `192.168.15.15`                                                                                                     |
| **Usuário padrão SSH**                 | `orangepi` ou `root`                                                                                                |
| **Imagem do sistema**                  | `Orangepi5_1.2.0_debian_bullseye_server_linux5.10.160.img`                                                          |
| **Link da imagem**                     | [Baixar imagem](http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/service-and-support/Orange-pi-5.html) |
| **Comando para verificar hash**        | `certUtil -hashfile "CAMINHO\DA\IMAGEM.img" SHA256` (no PowerShell)                                                 |
| **Usuário do Home Assistant (Docker)** | `homeassistant`                                                                                                     |
| **UUID do disco**                      | Identificado via `sudo blkid`                                                                                       |
| **Comando para boot via USB**          | `sudo dd if=/usr/lib/linux-u-boot-current-orangepi5_1.1.8_arm64/u-boot.itb of=/dev/sda bs=1024 seek=8`              |
| **Endereço do CasaOS**                 | `http://192.168.15.101:4357/` ou `http://orangepi5.local:8123/`                                                     |
| **Comando de backup do InfluxDB**      | `influxd backup -portable <path-to-backup>`                                                                         |
| **Comando de consulta InfluxDB**       | `influx -database homeassistant -execute "SELECT \"value\" FROM ...`                                                |

Perfeito! Com base na versão mais recente do seu tutorial, aqui estão algumas sugestões de melhoria e correções:

---

### ✅ Sugestões de melhoria

#### 1. Versão da imagem está desatualizada no tutorial

* A imagem usada no tutorial é a `Orangepi5_1.2.0_debian_bullseye_server_linux5.10.160.img`.
* Mas você está utilizando a `Orangepi5_1.1.8_debian_bookworm_server_linux5.10.160.img`.
* 👉 **Sugestão:** Atualize a referência da imagem para refletir a versão `1.1.8` e `bookworm`, para manter coerência com o restante do guia.

### 2. Atualização do sistema desnecessária

* Se você já está usando a imagem Bookworm (`1.1.8`), este trecho se torna redundante:

```bash
sed -i 's/bullseye/bookworm/g' /etc/apt/sources.list
apt update && apt full-upgrade -y
```

* 👉 **Sugestão:** Condicionalmente oriente o usuário a verificar sua versão do Debian com `lsb_release -a` antes de executar esse trecho, ou remova caso já esteja em Bookworm.

**3. Comando `mkdir /boot/firmware` pode ser desnecessário**

* Se `/boot/firmware/` já existe, o comando pode gerar erro.
* 👉 **Sugestão:** Use `mkdir -p /boot/firmware` para evitar erro caso o diretório já exista.

#### 4. Adicionar passo para habilitar o NetworkManager

* Alguns usuários relatam que o `NetworkManager` não vem habilitado por padrão.
* 👉 **Sugestão:** Adicione:

```bash
sudo systemctl enable NetworkManager
sudo systemctl start NetworkManager
```

**5. Clareza na parte do `equivs`**

* Instruções do `equivs` estão corretas, mas poderiam confundir iniciantes.
* 👉 **Sugestão:** Informe o que exatamente o pacote `systemd-resolved` fictício está resolvendo (ex: dependência do Home Assistant).

**6. Confusão na configuração do `/boot/orangepiEnv.txt`**

* Você dá duas instruções para `extraargs`. Uma antes com `apparmor=1...` e outra depois com `cma=128M`.
* 👉 **Sugestão:** Unifique ou esclareça que os parâmetros devem ser combinados, algo como:

```bash
extraargs=apparmor=1 security=apparmor systemd.unified_cgroup_hierarchy=false systemd.legacy_systemd_cgroup_controller=false cma=128M
```

**7. Comando do `dd` para boot USB assume que `/dev/sda` é o disco certo**

* Pode causar perda de dados.
* 👉 **Sugestão:** Adicione aviso claro antes do `dd`:

```markdown
⚠️ **Cuidado!** Confirme que `/dev/sda` é realmente o disco onde deseja instalar o bootloader. Use `lsblk` para verificar.
```

---

## 🏠 Instalando CasaOS

```bash
curl -fsSL https://get.casaos.io | sudo bash
```

Acesse **Orangepi 5** nos linsks abaixo:

* [http://orangepi5.local:8080/](http://orangepi5.local:8080/)
* [http://orangepi5.local:4357/](http://orangepi5.local:4357/)
* [http://orangepi5.local:8123/](http://orangepi5.local:8123/)

> 💡 **Dica:** Você pode instalar `rclone` se precisar:

```bash
apt install rclone -y
```

---

## 🔐 Instalando PiVPN

```bash
curl -L https://install.pivpn.io | bash
pivpn -a
pivpn -c
pivpn -l
```

Vídeo de referência: [Instalação PiVPN](https://www.youtube.com/watch?v=ZKFN3lvEJTo&t=444s)

---

## 🔎 Instalando Pi-hole

```bash
curl -sSL https://install.pi-hole.net | bash
```

### Alterar senha de acesso

```bash
sudo pihole -a -p
```

---

## 📊 Backup e Acesso ao InfluxDB

### Executar comandos

```bash
docker exec -it <container_id> /bin/bash
influxd backup -portable <path-to-backup>
```

### Consultas InfluxDB

```bash
influx -database homeassistant -execute "SELECT \"value\" FROM \"homeassistant\".\"autogen\".\"kWh\" WHERE \"entity_id\"='energy_meter_energia_total'" -format csv > /home/orangepi/influxdb/output.csv
```

> 📘 **Dica de uso**: Use `influx -username homeassistant -password homeassistant` para autenticar, se necessário.

---

## ✅ Conclusão

Agora seu Orange Pi 5 está preparado com o **Home Assistant**, **CasaOS**, **Pi-hole**, **PiVPN** e backup do **InfluxDB** configurado. Aproveite a automação residencial com performance e controle!

---

🎯 **Links úteis**:

* [Home Assistant](https://www.home-assistant.io/)
* [CasaOS](https://casaos.io/)
* [Pi-hole](https://pi-hole.net/)
* [PiVPN](https://pivpn.io/)
* [Doc Installing Home Assistant on OrangePi 5 Board](https://gist.github.com/renatoccosta/c30f0b4216c8caaf1f202b0a0561b5d3?permalink_comment_id=4583001)

---

🛟 **Precisa de ajuda?**
Deixe suas dúvidas nos comentários ou busque suporte na [comunidade oficial do Home Assistant](https://community.home-assistant.io/).
