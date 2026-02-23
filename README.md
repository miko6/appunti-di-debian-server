## My DEBIAN SERVER installation ![alt](https://github.com/miko6/appunti-di-debian-server/blob/main/immagini/debians.jpg)

0. ***Azioni preliminari***

`ip addr` per trovare l'ip del server  
- loggarsi come *su* e poi `apt install sudo`  
- aggiungere l'utente nomeutente al file sudoers `sudo usermod -aG sudo nomeutente`  

---  

1. ***Installare e abilitare il firewall ufw***  

`sudo apt install enable`  
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

3. ***Montaggio automatico disco secondario***

- identificare il disco ed il suo UUID con il comando  

`sudo blkid`  

- creaiamo il punto di mount  

`sudo mkdir -p /mnt/xxxx`  

- modifica del file /etc/fstab  

`sudo nano /etc/fstab`  

- aggiungiamo questa riga alla fine del file  

`UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /mnt/xxxx  ext4  defaults,noatime,nofail  0  2`  

- verifiche, prima senza riavviare  

`sudo mount -a` (se non si sono errori il file funziona)  

- riavviamo  

`sudo systemctl reboot`  

- al riavvio `lsblk` per conferma  

---

4. ***Installazione di Samba e condivisione del secondo disco***  

`sudo apt update`  

`sudo apt install samba`  

- Gestione degli utenti Samba - Crea un utente Samba  

`sudo smbpasswd -a nomeutente` digitare due volte la password per l'utente samba (diversa da quella di root)  

- Riavviare Samba per applicare le modifiche:  

`sudo service smbd restart`  

- Abilita il servizio Samba all'avvio del server  

`sudo systemctl enable smbd`  

- Configurazione di Samba  

`sudo nano /etc/samba/smb.conf`  

- Aggiungi una nuova sezione alla fine del file per la condivisione del punto di mount creato nello step precedente  

```
[NASm2]  
path = /mnt/xxxx  
valid users = nomeutente  
browsable = yes  
read only = no  
guest ok = no  
```  
- Consentire il traffico Samba se UFW è attivo:  

`sudo ufw allow 'Samba'`  

---

5. ***Installare Docker engine + Portainer + stacks***  

[Guida per installare l'engine](https://docs.docker.com/engine/install/debian/)  

- Portainer  

`sudo apt update && sudo apt upgrade -y`  
`sudo docker volume create portainer_data`  
`sudo docker run -d -p 8000:8000 -p 9443:9443 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest`  

- Accedi alla Web UI:

Apri il browser e vai su https://<IP_del_tuo_server>:9443  

- Installa gli stack preferiti (es. PiHole), loggati sulla pagina di Portainer, vai in Home, Environments, local, Staks, Add stacks, inserisci un nome per lo stack, nell'area di editing assicurati che sia selezionato l'editor "Web editor", incolla il compose, clicca sul pulsante "Deploy the stack"

> :memo: **Note:** Prima di procedere, modifica le parti fondamentali del codice che hai appena incollato:  

    TZ: 'Europe/Rome': Imposta il tuo fuso orario. Se sei in Italia, Europe/Rome è corretto. Puoi trovare la lista completa qui .

    FTLCONF_webserver_api_password: 'latuapassword': Sostituisci la_tua_password_sicura con una password complessa a tua scelta. Questa ti servirà per accedere all'interfaccia di amministrazione di Pi-hole  

- PiHole  

```version: "3"

# More info on https://github.com/pi-hole/docker-pi-hole/ and https://docs.pi-hole.net/
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    ports:
      # DNS Ports (must be free on the host)
      - "53:53/tcp"
      - "53:53/udp"
      # Web Admin Interface Port (choose a port if 80 is busy, e.g., "8080:80")
      - "80:80/tcp"
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
---

6. ***Sensori temperatura***  

`sudo apt install lm-sensors` richiamo delle temp con `sensors`  

---

7. ***btop***  

`sudo apt install btop`  

---

8. ***fastfetch***

`sudo apt install fastfetch`  
