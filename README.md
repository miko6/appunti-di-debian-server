## My DEBIAN SERVER installation ![alt](http://gitea.local/miko/Owner-avatar-appunti-di-debian-server/raw/branch/main/immagini/debians.jpg)

0. ***Azioni preliminari***

- `ip addr` per trovare l'ip del server  
- loggarsi come *su* e poi `apt install sudo`  
- aggiungere l'utente nomeutente al file sudoers

`su -`  

`visudo`

- cerchiamo la riga *# Allow members of group sudo to execute any command*

e aggiungiamo questa riga `nomeutente ALL=(ALL:ALL) ALL`

---  

1. ***Installare e abilitare il firewall ufw***  

`sudo apt install ufw`  

`sudo ufw enable`  

`sudo ufw status`  

---

2. ***Installare ssh***  

`sudo apt install ssh`  

- verificare lo stato di SSH  

`sudo systemctl status ssh`  

- Qualora il servizio SSH sia ancora inattivo e non si avvii automaticamente neanche in seguito a un reboot, potrete modificare lo stato digitando queste due righe di comando:  

`sudo systemctl enable ssh`  

`sudo systemctl start ssh`  

- aprire la porta SSH nel firewall  

`sudo ufw allow ssh`  

---

3. ***Configurare IP statico***  

`ip link` per trovare il nome esatto dell'interfaccia di rete

`sudo nano /etc/network/interfaces`  

```
# The loopback network interface
auto lo
iface lo inet loopback
 
# The primary network interface
auto nomeinterfaccia
iface nomeinterfaccia  inet static
 address 192.168.1.xxx
 netmask 255.255.255.0
 gateway 192.168.1.1
 dns-nameservers 1.1.1.1 8.8.8.8
```

- Configurare i DNS

`sudo nano /etc/resolv.conf`

- aggiungere queste righe

```
nameserver 1.1.1.1
nameserver 8.8.8.8
```

- riavviamo

4. ***Disabilitare IPV6***  

`sudo nano /etc/default/grub`  

- Trova la riga che inizia con **GRUB_CMDLINE_LINUX_DEFAULT** e aggiungi *ipv6.disable=1* all'interno delle virgolette. Esempio di riga modificata: `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash ipv6.disable=1`  

`sudo update-grub` per rendere le modifiche efefttive

- riavviare  

`sudo reboot now`  

- Se disabilitiamo l'*IPV6* possiamo fare questa modifica nella configurazione del firewall *ufw*:  

`sudo nano /etc/default/ufw`  

e settiamo il valore *IPV6* su *no*  

---

5. ***Montaggio automatico disco secondario***

- identificare il disco ed il suo *UUID* con il comando  

`sudo blkid`  

- creiamo il punto di mount  

`sudo mkdir -p /mnt/xxxx`  

- modifica del file /etc/fstab  

`sudo nano /etc/fstab`  

- aggiungiamo questa riga alla fine del file  

`UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /mnt/nasm2  ext4  defaults,noatime,nofail  0  2`  

o per una partizione *ntfs*

`UUID=xxxxxxxxxxxxxx /mnt/seagate ntfs-3g defaults,noatime,nofail,uid=1000,gid=1000,dmask=022,fmask=133 0 0`

- verifiche, prima senza riavviare  

`sudo mount -a` (se non si sono errori il file funziona)  

- riavviamo  

`sudo systemctl reboot`  

- al riavvio `lsblk` per conferma  

---

6. ***Installazione di Samba e condivisione del secondo disco***  

`sudo apt update`  

`sudo apt install cifs-utils`

`sudo apt install samba`  

- Gestione degli utenti Samba - Crea un utente Samba  

`sudo smbpasswd -a nomeutente` digitare due volte la password per l'utente samba (diversa da quella di root)  

- Riavviare Samba per applicare le modifiche:  

`sudo service smbd restart`  

- Abilita il servizio Samba all'avvio del server  

`sudo systemctl enable smbd`  

- Configurazione di Samba  

`sudo nano /etc/samba/smb.conf`  

- Aggiungi una nuova sezione alla fine del file per la condivisione dei punti di mount creati nello step precedente  

```
[NASm2]  
path = /mnt/nasm2  
valid users = nomeutente  
browsable = yes  
read only = no  
guest ok = no  

[seagate]
path = /mnt/seagate
valid users = nomeutente
browsable = yes
read only = no
guest ok = no
```  
- Consentire il traffico Samba se UFW è attivo:  

`sudo ufw allow 'Samba'`  

---

7. ***Installare Docker engine + Portainer + stacks***  

[Guida per installare l'engine](https://docs.docker.com/engine/install/debian/)  

- Portainer  

`sudo apt update && sudo apt upgrade -y`  

`sudo docker volume create portainer_data`  

`sudo docker run -d -p 8000:8000 -p 9443:9443 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest`  

- Accedi alla Web UI:  
Apri il browser e vai su https://<IP_del_tuo_server>:9443  

- [Script di aggiornamento per Portainer](https://github.com/mynameismaurizio/portainer-updater) 

`mkdir -p ~/.local/bin`

`nano ~/.local/bin/update-portainer`  

copiare e incollare lo script:  

```
##############################################################################################
#                                                                                            #
#  _____           _        _                   _    _           _       _                   #
# |  __ \         | |      (_)                 | |  | |         | |     | |                  #
# | |__) |__  _ __| |_ __ _ _ _ __   ___ _ __  | |  | |_ __   __| | __ _| |_ ___ _ __        #
# |  ___/ _ \| '__| __/ _` | | '_ \ / _ \ '__| | |  | | '_ \ / _` |/ _` | __/ _ \ '__|       #
# | |  | (_) | |  | || (_| | | | | |  __/ |    | |__| | |_) | (_| | (_| | ||  __/ |          #
# |_|   \___/|_|   \__\__,_|_|_| |_|\___|_|     \____/| .__/ \__,_|\__,_|\__\___|_|          #
#                                                     | |                                    #
#                                                     |_|                                    #
#                                                                                            #
#                                    Portainer Updater                                       #
#                                    by mynameismaurizio                                     #
#                                    for the community                                       #
#                                                                                            #
##############################################################################################


#!/bin/bash
# Define color codes
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
RED='\033[0;31m'
NC='\033[0m' # No color
# Define Portainer version (default to latest if not provided)
target_version=${1:-"latest"}
# Pull the specified Portainer image
echo -e "${YELLOW}Pulling Portainer CE version $target_version...${NC}"
if sudo docker pull portainer/portainer-ce:$target_version; then
echo -e "${GREEN}Successfully pulled Portainer CE version $target_version${NC}"
else
echo -e "${RED}Error pulling Portainer version $target_version. Please check your connection and try again.${NC}"
exit 1
fi
# Stop and remove the existing Portainer container
echo -e "${YELLOW}Stopping and removing the current Portainer container...${NC}"
if sudo docker stop portainer && sudo docker rm portainer; then
echo -e "${GREEN}Successfully stopped and removed the current Portainer container.${NC}"
else
echo -e "${RED}Error stopping/removing the Portainer container. Please ensure the container is running.${NC}"
exit 1
fi
# Start the new Portainer container
echo -e "${YELLOW}Starting Portainer version $target_version...${NC}"
if sudo docker run --name portainer --restart=unless-stopped -d \
-p 8000:8000 -p 9000:9000 -p 9443:9443 \
-v /var/run/docker.sock:/var/run/docker.sock \
-v portainer_data:/data portainer/portainer-ce:$target_version \
--trusted-origins portainer.local; then
echo -e "${GREEN}Portainer has been updated to version $target_version!${NC}"
else
echo -e "${RED}Error starting the new Portainer container. Please check Docker logs for details.${NC}"
exit 1
fi
```

`chmod +x ~/.local/bin/update-portainer`

`sudo mv ~/.local/bin/update-portainer /usr/local/bin/update-portainer`  

*update-portainer* nel terminale per lanciare l'aggiornamento

- Installa gli stack preferiti (es. PiHole), loggati sulla pagina di Portainer, vai in Home, Environments, local, Stacks, Add stacks, inserisci un nome per lo stack, nell'area di editing assicurati che sia selezionato l'editor "Web editor", incolla il compose, clicca sul pulsante "Deploy the stack"

> :memo: **Note:** Prima di procedere, modifica le parti fondamentali del codice che hai appena incollato:  

    TZ: 'Europe/Rome': Imposta il tuo fuso orario. Se sei in Italia, Europe/Rome è corretto. Puoi trovare la lista completa qui .

    FTLCONF_webserver_api_password: 'latuapassword': Sostituisci la_tua_password_sicura con una password complessa a tua scelta. Questa ti servirà per accedere all'interfaccia di amministrazione di Pi-hole  

- PiHole compose script  

```version: "3"

# More info on https://github.com/pi-hole/docker-pi-hole/ and https://docs.pi-hole.net/
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    dns: 127.0.0.1  # <--- AGGIUNGI QUESTA RIGA QUI
    ports:
      # DNS Ports (must be free on the host)
      - "53:53/tcp"
      - "53:53/udp"
      # Web Admin Interface Port (choose a port if 80 is busy, e.g., "8080:80")
      - "8053:80/tcp"
      # Uncomment the line below if you want to use Pi-hole as a DHCP server (advanced)
      #- "67:67/udp"
    environment:
      # Set your timezone: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
      TZ: 'Europe/Rome'
      # Set a password for the web interface. CHANGE THIS!
      FTLCONF_webserver_api_password: 'latuapassword'
      # Tells Pi-hole to listen on all interfaces, necessary for Docker's default network
      FTLCONF_dns_listeningMode: 'ALL'
    # Volumes store your data between container upgrades (important!)
    volumes:
      - './etc-pihole:/etc/pihole'
      - './etc-dnsmasq.d:/etc/dnsmasq.d'
    # Capabilities: NET_ADMIN is only needed if you uncomment the DHCP port above
    # cap_add:
    #   - NET_ADMIN
    restart: unless-stopped  
```  

> :memo: Per aggiornare la versione di *PiHole* da *Portainer* **Home/Containers/**selezionare **pihole/Recreate/**selezionare **Pull the latest/Recreate** done.  

- BentoPDF

```
services:
  bentopdf:
    # GitHub Container Registry (Recommended)
    # Self-Hosted build - ghcr.io/alam00000/bentopdf-simple:latest
    # Commercial build  - ghcr.io/alam00000/bentopdf:latest
    # Docker Hub (Alternative)
    # Self-Hosted build - bentopdfteam/bentopdf-simple:latest
    # Commercial build  - bentopdfteam/bentopdf:latest
    image: ghcr.io/alam00000/bentopdf-simple:latest
    container_name: bentopdf
    restart: unless-stopped
    ports:
      - '8080:8080'
    # For IPv4-only environments
    #environment:
    #  - DISABLE_IPV6=true
```

- nginx proxy manager

```
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      # Questa è la porta HTTP standard che prenderà il controllo dei tuoi domini .local
      - '80:80'
      # Questa è la porta HTTPS standard (per il futuro)
      - '443:443'
      # Questa è la porta per entrare nel pannello di controllo di Nginx Proxy Manager
      - '81:81'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

 adesso l'interfaccia di *nginx* è raggiungibile da *http://192.168.1.192:81*

> :memo: ci sono delle opzioni aggiuntive da aggiungere alle opzioni avanzate di pihole, portainer e webimn quando si crea il proxy e sono le seguenti:

- per *pihole*

```
location = / {
    return 301 $scheme://$host/admin/;
}
```
- per *portainer* (è necessario anche attivare l'opzione Websockets nella prima schermata di creazione del proxy)

```
proxy_ssl_verify off;
```

per *webmin*

```
proxy_ssl_verify off;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

Entra in Webmin usando il vecchio metodo: https://192.168.1.192:10000.  
Vai su Webmin (nel menu a sinistra) -> Webmin Configuration.  
Clicca sull'icona Trusted referers (Referenti fidati).  
Nel campo Trusted websites or hosts, aggiungi webmin.local. Clicca su Save.  
Da quel momento http://webmin.local funzionerà perfettamente!

---

8. ***Webmin***

[1] Install required packages  

`sudo apt -y install python3 shared-mime-info unzip apt-show-versions libapt-pkg-perl libauthen-pam-perl libio-pty-perl libnet-ssleay-perl`

[2] Install Webmin  

`sudo wget https://www.webmin.com/download/deb/webmin-current.deb`  

`sudo dpkg -i webmin-current.deb`  

`sudo nano /etc/webmin/miniserv.conf`  

[3]  # add to last line : add access permission (include anche comandi per la corretta gestione di webmin in nginx)

```
allow=192.168.1.0/24 172.20.0.2  
trust_real_ip=1
no_testing_cookie=1
```

- se riceviamo un errore nell'interfaccia di webmin relativo ad una directory */tmp/.webmin* procediamo così:

`sudo mkdir -p /var/tmp/.webmin` e nella configurazione di webmin cambiamo la cartella con quella appena creata */var/tmp/.webmin*

[4] Per evitare conflitti tra le *WebUi* dei servizi installati nel server andiamo a modificare il file /etc/hosts nel sistema client **(nel mio caso Linux Mint LMDE)** nel seguente modo: 

`sudo nano /etc/hosts`  

aggiungiamo al file le seguenti linee  

```
192.168.1.xxx   pi.hole  
192.168.1.xxx   webmin.local  
192.168.1.xxx   portainer.local  
192.168.1.xxx   bentopdf.local
```

> :memo: **Note:** Se dall'interfaccia riceviamo un avviso che relativo alla Temporary files directory apriamo il terminale di webmin e diamo il comando `sudo mkdir -p /var/tmp/.webmin` poi nella sezione *Webmin configuration* nella sezione *Temporary files directory* personalizziamo il percorso con */var/tmp/.webmin* e salviamo  

---

9. abilitare riconoscimento partizioni *ntfs*

`sudo apt install ntfs-3g`

10. ***Sensori temperatura***  

`sudo apt install lm-sensors` richiamo delle temp con `sensors`  

---

11. ***htop***  

`sudo apt install htop`  

---

12. ***fastfetch***

`sudo apt install fastfetch`  

---  

13. ***rsync***

`sudo apt install rsync` - [GUIDA](https://supporthost.com/it/comando-rsync-linux/) - [man page](https://manpages.debian.org/bookworm/rsync/rsync.1.en.html) 

---

14. ***sensori***

`sudo apt install lm-sensors`
`sudo sensors-detect` rispondere si a tutte le domande

15. **fail2ban**  

 `sudo apt install fail2ban`  

 configurare fail2ban per ssh  

 `sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`  

 modifichiamo il file di configurazione appena creato  

 `sudo nano /etc/fail2ban/jail.local`  

 nella jail [DEFAULT] troviamo la linea con #ignoreip, decommentiamola e rendiamola così:

 ```
 ignoreip = 127.0.0.1/8 ::1 192.168.1.0/24
 ```

 scorriamo in basso fino a trovare la sezione [sshd] e settiamola così  

```
[sshd]

enabled = true
port    = ssh
backend = systemd
maxretry = 3
findtime = 300
bantime = 3600
logpath = %(dropbear_log)s  
```  

riavviamo il servizio con:  

`sudo systemctl restart fail2ban`  

verifichiamo lo stato del servizio:  

`sudo systemctl status fail2ban`  

monitoriamo il servizio:  

`sudo fail2ban-client status sshd`  

comandi utili:  

`sudo fail2ban-client unban --all` sbannare tutti  
`sudo fail2ban-client unban <ip-address>`  sbannare singolo ip  
`sudo tail -f /var/log/fail2ban.log` per monitorare l'attività di f2b  
`sudo fail2ban-client -x start` utile per capire un errore che impedisce l'avvio del servizio  
`sudo fail2ban-client status` vadere tutte le jail attive  
`sudo fail2ban-client status sshd` vedere lo status della jail ssh

fonti utili:  
[Link 1](https://guide.debianizzati.org/index.php/Fail2ban#Introduzione) - [Link 2](https://linuxiac.com/how-to-protect-ssh-with-fail2ban/) - [Link 3](https://www.evemilano.com/blog/fail2ban/)  

> :memo: Ho riscontrato un problema con gli IP della rete locale che venivano bannati nonostante fossero inseriti nella stringa *ignoreip* nella sezione *[DEFAULT]* del file *jail.local* in */etc/fail2ban*. Per ovviare al problema controllare il file *sshd.conf* presente in */etc/fail2ban/jail.d*. Eliminare la riga *ignoreip* se presente.  