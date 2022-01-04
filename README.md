# TIM HUB ROOT - VERSION AGTHP 2.3.3


## WEBSITES
- [Hacking Technicolor Gateways: Material for MkDocs](https://hack-technicolor.readthedocs.io/en/stable/)
- [IlPuntoTecnico](https://www.ilpuntotecnico.com/forum/index.php?topic=81461.0)


## FILE UTILI
- Path: *TIM_HUB_root.zip/autoflashgui-master_timhub.zip/autoflashgui-master/firmware*


---


## GUIDE - PART 1
- Aggiornare TIM HUB alla versione 2.3.3
- Dalla prima scheda nella GUI web, eseguire backup configurazione in `.bin` se necessario
- Eseguire reset modem
- Al riavvio, login nella pagina web (`admin/admin`), non cambiare la password e attivare le funzionalità estese
- Per rifare il login nella GUI, la password è la **ACCESS KEY** sull'etichetta posteriore del modem (sotto l'ultimo codice a barre nella colonna a sinistra)
- Entrare nella prima scheda ed eseguire dalla terza tab il downgrade alla versione 1.0.3 caricando il file `AGTHP_1.0.3_CLOSED.rbi`
- Al riavvio, non sarà possibile fare il login. Eseguire reset modem dal tasto sul retro (tenere premuto per 10-12 sec.)
- Al riavvio, login nella pagina web (`admin/admin`) senza cambiare la password
- Eseguire il programma `autoflashgui.exe` contenuto nella cartella *autoflashgui-master*


### AUTOFLASHGUI.EXE
- Load default: *Generic (Advanced DDNS)*
- Target IP: ip modem
- Username: user GUI web
- Password: password GUI web
- **NON** selezionare *Firmware File Name* e la spunta *Flash firmware*
- Attivare *Split the given command on semicolons [...]* se non selezionato
- Lasciare invariato il resto delle impostazioni
- Cliccare su *Run*
- Attendere risultato sulla shell
- Chiudere il programma
- Collegarsi in SSH al modem e provare a autenticarsi con `root/root`


---


## GUIDE - PART 2
- Dalla shell root del modem abilitare la Serial Console Port
    - `sed -i -e 's/#//' -e 's#askconsole:.*\$#askconsole:/bin/ash#' /etc/inittab`
- Verificare lo stato delle bank
    - `find /proc/banktable -type f -print -exec cat {} ';' -exec echo ';'`
- Prendere nota dei seguenti paramentri

        ...
	    /proc/banktable/booted
	    <take note of this>
	    proc/banktable/active
	    <take note of this>
	    ...

- E' necessario che il risultato del comando precedente diventi come segue

        /proc/banktable/active
	    bank_1
	    /proc/banktable/activeversion
	    Unknown
	    /proc/banktable/booted
	    bank_2

- Proseguire quindi al passaggio successivo per impostare come active il `bank_1` per poi cancellarlo e fare in modo che vada in boot il `bank_2`


### SCRIPT
- Creare con il comando `vim` uno script con i seguenti comandi

```bash
# Ensure two banks match in sizes
[ $(grep -c bank_ /proc/mtd) = 2 ] && \
[ "$(grep bank_1 /proc/mtd | cut -d' ' -f2)" = \
"$(grep bank_2 /proc/mtd | cut -d' ' -f2)" ] && {
# Clone and verify firmware into bank_2 if applicable
[ "$(cat /proc/banktable/booted)" = "bank_1" ] && {
mtd -e bank_2 write /dev/$(grep bank_1 /proc/mtd | cut -d: -f1) bank_2 && \
mtd verify /dev/$(grep bank_1 /proc/mtd | cut -d: -f1) bank_2 || \
{ echo Clone verification failed, retry; exit; } }
# Make a temp copy of overlay for booted firmware
cp -rf /overlay/$(cat /proc/banktable/booted) /tmp/bank_overlay_backup
# Clean up jffs2 space by removing existing old overlays
rm -rf /overlay/*
# Use the previously made temp copy as overlay for bank_2
cp -rf /tmp/bank_overlay_backup /overlay/bank_2
# Activate bank_1
echo bank_1 > /proc/banktable/active
# Make sure above changes get written to flash
sync
# Erase firmware in bank_1
mtd erase bank_1;
# Emulate system crash to hard reboot
echo c > /proc/sysrq-trigger; }
# end
```

- Lanciare il seguente comando per renderlo eseguibile
    - `chmod +x script.sh`


---


## GUIDE - PART 3
- E' possibile proseguire con l'update del firmware per tornare alla versione 2.3.3
- Aprire WinSCP e collegarsi con protocollo SCP al modem con credenziali `root/root`
- Caricare nella directory `/tmp` del modem il file `AGTHP_2.3.3_CLOSED.rbi` rinominandolo in `new.rbi`
- Eseguire dalla shell il seguente comando
    - `cat "/tmp/new.rbi" | (bli_parser && echo "Please wait..." && (bli_unseal | dd bs=4 skip=1 seek=1 of="/tmp/new.bin"))`
- E' necessario fare un clean-up di file e configurazioni
- Creare un backup con il seguente comando e salvarlo sul proprio PC tramite WinSCP
    - `tar -C /overlay -cz -f /tmp/backup-$(date -I).tar.gz $(cat /proc/banktable/booted)`
- Eseguire il comando seguente per cancellare completamente l'overlay della bank attualmente bootata
    - `rm -rf /overlay/$(cat /proc/banktable/booted)`
- Cambiando versione del firmware il root potrebbe andare perso


### PRESERVING ROOT ACCESS
- Eseguire il blocco di comandi seguente per preparare uno script che andrà eseguito una volta sola al boot successivo per garantire l'accesso con root

> COPIA E INCOLLA NEL TERMINALE. PREMERE INVIO PER ESEGUIRE L'ULTIMO COMANDO

```bash
mkdir -p /overlay/$(cat /proc/banktable/booted)/etc
chmod 755 /overlay/$(cat /proc/banktable/booted) /overlay/$(cat /proc/banktable/booted)/etc
echo -e "echo root:root | chpasswd
sed -i 's#/root:.*\$#/root:/bin/ash#' /etc/passwd
sed -i -e 's/#//' -e 's#askconsole:.*\$#askconsole:/bin/ash#' /etc/inittab
uci -q set \$(uci show firewall | grep -m 1 \$(fw3 -q print | \
egrep 'iptables -t filter -A zone_lan_input -p tcp -m tcp --dport 22 -m comment --comment \"!fw3: .+\" -j DROP' | \
sed -n -e 's/^iptables.\+fw3: \(.\+\)\".\+/\1/p') | \
sed -n -e \"s/\(.\+\).name='.\+'$/\1/p\").target='ACCEPT'
uci add dropbear dropbear
uci rename dropbear.@dropbear[-1]=afg
uci set dropbear.afg.enable='1'
uci set dropbear.afg.Interface='lan'
uci set dropbear.afg.Port='22'
uci set dropbear.afg.IdleTimeout='600'
uci set dropbear.afg.PasswordAuth='on'
uci set dropbear.afg.RootPasswordAuth='on'
uci set dropbear.afg.RootLogin='1'
uci set dropbear.lan.enable='0'
uci commit dropbear
/etc/init.d/dropbear enable
/etc/init.d/dropbear restart
rm /overlay/\$(cat /proc/banktable/booted)/etc/rc.local
source /rom/etc/rc.local
" > /overlay/$(cat /proc/banktable/booted)/etc/rc.local
chmod +x /overlay/$(cat /proc/banktable/booted)/etc/rc.local
sync
```

- Se la password di root è stata cambiata, questa verrà resettata a `root/root`
- Il gateway adesso è pulito. L'accesso con root tramite SSH verrà abilitato di nuovo permanentemente al boot successivo


### FLASHING FIRMWARE
- Eseguire i seguenti comandi per scrivere il file `/tmp/new.bin` nella bank booted e per provocare un hard reboot
    - `mtd -e $(cat /proc/banktable/booted) write "/tmp/new.bin" $(cat /proc/banktable/booted)`
    - `echo c > /proc/sysrq-trigger`


### HARDENING GAINED ACCESS
- Eseguire i seguenti comandi nel terminale SSH per prevenire che il modem perda inaspettatamente l'accesso root

> COPIA E INCOLLA NEL TERMINALE. PREMERE INVIO PER ESEGUIRE L'ULTIMO COMANDO

```bash
# Disable CWMP
uci delete cwmpd.cwmpd_config
uci delete firewall.cwmpd
uci del_list watchdog.@watchdog[0].pidfile='/var/run/cwmpd.pid'
uci del_list watchdog.@watchdog[0].pidfile='/var/run/cwmpevents.pid'
uci commit
/etc/init.d/watchdog-tch reload
/etc/init.d/cwmpd disable
/etc/init.d/cwmpd stop
/etc/init.d/cwmpdboot disable
/etc/init.d/cwmpdboot stop
/etc/init.d/zkernelpanic disable
/etc/init.d/zkernelpanic stop

# Disable CWMP - extra, in case you think it may resurrect
uci set cwmpd.cwmpd_config.state=0
uci set cwmpd.cwmpd_config.acs_url='https://127.0.1.1:7547/'
uci set cwmpd.cwmpd_config.use_dhcp=0
uci set cwmpd.cwmpd_config.interface=loopback
uci set cwmpd.cwmpd_config.enforce_https=1
uci commit cwmpd

# Disable Telstra monitoring
uci delete tls-vsparc.Config
uci delete tls-vsparc.Passive
uci delete autoreset.vsparc_enabled
uci delete autoreset.thor_enabled
uci delete wifi_doctor_agent.acs
uci delete wifi_doctor_agent.config
uci delete wifi_doctor_agent.as_config
uci commit

# Disable Telstra Air/Fon WiFi
/etc/init.d/hotspotd stop
/etc/init.d/hotspotd disable
uci delete dhcp.hotspot
uci delete dhcp.fonopen
uci commit

# Remove any default SSH pubkey
echo > /etc/dropbear/authorized_keys
# Disable SSH access over wan
uci set dropbear.wan.enable='0'
uci commit dropbear

# Free space for gateways with small flash
find /rom/usr/lib/ipk -type f |xargs -n1 basename | cut -f 1 -d '_' |xargs opkg --force-removal-of-dependent-packages remove
```

- Se ricevi messaggi di errore da un comando, è possibile ignorarli: significa che il comando non era necessario per la tua versione del firmware


### GUI ANSUEL
- Collegarsi con WinSCP al modem come descritto in precedenza
- Copiare il file `GUI.tar.bz2` nella directory `/tmp`
- Collegarsi in SSH al modem con root
- Eseguire il seguente comando per estrarre la GUI
    - `bzcat /tmp/GUI.tar.bz2 | tar -C / -xvf - && /etc/init.d/rootdevice force`
- Attendere fino al termine della procedura. Se necessario il modem potrebbe riavviarsi da solo. Ignorare gli ultimi messaggi di errore
- In caso di Errore 9 riavviare il modem e il problema sarà risolto


### ROOT PASSWORD RESET
- Eseguire il comando `passwd` per cambiare la password di accesso dell'utente root



