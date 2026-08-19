---
sidebar_position: 6
---
# Backups automatisch in die Cloud oder NAS hochladen

Es wird ein Beispiel für Google Drive gezeigt, andere (Cloud)Diense sind ähnlich über rclone möglich.

### Annahmen zum System:
*IP-Adresse*: 192.168.1.30

*Benutzername*: pi (ggf. in folgenden Befehlen anpassen)

### rclone einrichten
- Per SSH mit Teslalogger verbinden: `ssh -L 53682:localhost:53682 pi@192.168.1.30`
(`-L 53682:localhost:53682` ist für rclone nötig)
- Falls nicht vorhanden *rclone* installieren: `sudo apt-get install -y rclone`
- `rclone config` ausführen
- Verbindung mit **n** erstellen (z. B.: "tl_backup") und **ENTER** drücken
![Image](https://raw.githubusercontent.com/bassmaster187/TeslaLogger/master/TeslaLogger/screenshots/backup_gdrive1.png)
- Listennummer für Google Drive finden, eingeben und **ENTER**
![Image](https://raw.githubusercontent.com/bassmaster187/TeslaLogger/master/TeslaLogger/screenshots/backup_gdrive2.png)
![Image](https://raw.githubusercontent.com/bassmaster187/TeslaLogger/master/TeslaLogger/screenshots/backup_gdrive3.png)
- Dieses HowTo für *client_id* und *client_secret* verwenden:
  https://rclone.org/drive/#making-your-own-client-id
- **3** und **ENTER** für *scope*, **ENTER** bei *service_account_file*, **ENTER** bei *Edit advanced config*, **ENTER** bei *Use auto config*
![Image](https://raw.githubusercontent.com/bassmaster187/TeslaLogger/master/TeslaLogger/screenshots/backup_gdrive5.png)

- Angezeigten Link kopieren und im Browser öffnen. Mit Google-Konto anmelden und Zugriff erlauben.
![Image](https://raw.githubusercontent.com/bassmaster187/TeslaLogger/master/TeslaLogger/screenshots/backup_gdrive6.png)
Danach **ENTER** für *Shared Drive (Team Drive)*
![Image](https://raw.githubusercontent.com/bassmaster187/TeslaLogger/master/TeslaLogger/screenshots/backup_gdrive7.png)
Es wird Konfiguration mit Tokens angeziegt, mit **ENTER** bestätigen und mit **q** rclone beenden

### Backup scirpt erzeugen
- nano öffnen: `nano /home/pi/backup.sh` und folgendes einfügen:
```
#!/bin/bash

BACKUP_DIR="/etc/teslalogger/backup"
REMOTE_NAME="tl_backup:Backup/teslalogger"

/usr/bin/rclone copy --update --verbose --transfers 3 --contimeout 60s --timeout 300s --retries 3 --low-level-retries 10 --stats 1s "$BACKUP_DIR" "$REMOTE_NAME" --drive-use-trash=false
```

Die Pfade, besonders bei Docker Installation müssen angepasst werden!

`tl_backup:Backup/teslalogger` bedeutet: es wird ein Ordner Namens Backup und in diesem noch ein Ordner teslalogger angelegt.

Falls man mehrere Applikationen in dem selben "Backup" Ordner ablegen möchte, wird folgendes empfohlen:

`REMOTE_NAME="tl_backup,root_folder_id=abcdefg:/teslalogger"`


wobei `abcdef` die OrdnerID ist, die man in der Url sieht, wenn man im Google Drive über Web-Browser auf den Ordner zugreift:
![Image](https://raw.githubusercontent.com/bassmaster187/TeslaLogger/master/TeslaLogger/screenshots/backup_gdrive8.png)

Mit diesem Befehl werden alle Backup-Dateien über die zuvor erstellte rclone-*tl_backup*-Verbindung in den Ordner "Backup/teslalogger" kopiert. Bei Bedarf anpassen.
Nach dem ersten Test wird empfohlen statt `copy` `move` zu nutzen. So werden hochgeladen Dateien anschließen aus dem lokalem Ordner gelöscht.

- Datei speichern: **CTRL+X**, **y**, **ENTER**
- Ausführbar machen: `chmod +x /home/pi/backup.sh`
- Testen: `chmod +x /home/pi/backup.sh`
- Bei vielen Backups dauert der erste Lauf länger. Erfolg sieht so aus:
![Image](https://raw.githubusercontent.com/bassmaster187/TeslaLogger/master/TeslaLogger/screenshots/backup_gdrive9.png)

### Backup Script jede Nacht ausführen

#### Backup service einrichten:
`sudo nano /etc/systemd/system/backup-rclone.service`

Folgendes einfügen und ggf. User und Scriptpfad anassen:

```[Unit]
Description=Rclone Backup und Bereinigung
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=pi
ExecStart=/bin/bash /home/pi/backup.sh

[Install]
WantedBy=multi-user.target
```

#### Timer einrichten
`sudo nano /etc/systemd/system/backup-rclone.timer`

```
[Unit]
Description=Timer für Rclone Backup jede Nacht

[Timer]
OnCalendar=*-*-* 04:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Anschließend Systemd Manager neustarten um neue Einstellungen zu übernehmen:
`sudo systemctl daemon-reload`

Timer aktivieren:
`sudo systemctl enable --now backup-rclone.timer`

Und den Status prüfen: 
`systemctl status backup-rclone.timer`

"Active:" soll auf *active* stehen

Backupservice sofort manuell zum Testen ausfürhen:
`sudo systemctl start backup-rclone.service`