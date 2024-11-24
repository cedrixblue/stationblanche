Station Blanche

Ce projet est ambitieux mais réalisable. Nous allons le diviser en étapes bien structurées, chacune correspondant à un objectif précis.

---

### **1. Installation et configuration d'Ubuntu 24.04**
#### Objectifs :
- Préparer une station blanche avec Ubuntu 24.04 GNOME pour scanner les clés USB.
- Configurer l'environnement réseau pour permettre la mise à jour des antivirus.

#### Étapes :
1. **Installation d'Ubuntu 24.04 :**
   - Télécharger la dernière ISO depuis le site officiel d’Ubuntu.
   - Créer une clé USB bootable avec un outil comme *Rufus* ou `dd` sous Linux :
     ```bash
     sudo dd if=ubuntu-24.04-desktop-amd64.iso of=/dev/sdX bs=4M status=progress
     ```
   - Installer Ubuntu avec les paramètres par défaut. Créez un utilisateur dédié (par ex. `stationblanche`).

2. **Mise à jour initiale :**
   - Après installation, mettez à jour le système :
     ```bash
     sudo apt update && sudo apt upgrade -y
     ```

3. **Configurer le réseau et le pare-feu :**
   - Activer et configurer `ufw` :
     ```bash
     sudo ufw enable
     sudo ufw allow out
     sudo ufw default deny incoming
     ```

4. **Verrouiller l’environnement utilisateur :**
   - Restreindre l’accès aux clés USB et aux ressources système aux administrateurs :
     ```bash
     sudo chmod 700 /home/stationblanche
     ```

---

### **2. Installation et configuration de ClamAV et Eset**
#### Objectifs :
- Installer deux antivirus : ClamAV (open source) et Eset (propriétaire).
- Configurer les deux pour fonctionner de manière complémentaire.

#### Étapes :
1. **Installer ClamAV :**
   ```bash
   sudo apt install clamav clamav-daemon -y
   sudo systemctl stop clamav-freshclam
   sudo freshclam  # Mise à jour de la base virale
   sudo systemctl start clamav-freshclam
   ```

2. **Configurer ClamAV :**
   - Modifier le fichier `/etc/clamav/clamd.conf` si nécessaire pour ajuster les performances.

3. **Installer Eset :**
   - Télécharger l’installateur depuis [ESET Endpoint Antivirus pour Linux](https://www.eset.com/int/business/download/endpoint-antivirus-linux/).
   - Installer le paquet .deb :
     ```bash
     sudo dpkg -i eset_nod32av_*.deb
     sudo apt --fix-broken install  # En cas de dépendances manquantes
     ```
   - Activer Eset :
     ```bash
     sudo /opt/eset/ees/bin/eav-service --register
     ```

4. **Tester les antivirus :**
   - Scanner un fichier test (exemple [eicar.com](https://www.eicar.org/download/eicar-com/?wpdmdl=8840&refresh=6743b65ea68471732490846)) :
     ```bash
     clamscan eicar.com
     /opt/eset/ees/bin/eav_scan eicar.com
     ```

---

### **3. Développement du script shell avec Zenity**
#### Objectifs :
- Créer un script pour gérer l’interface utilisateur et automatiser les tâches.

#### Exemple de script pour l'interface graphique avec Zenity :

```bash
#!/bin/bash
# station_blanche.sh - Script principal

# Configuration de la quarantaine et de l'historique
QUARANTINE_DIR="/home/stationblanche/quarantaine"
SCAN_LOG_DIR="/home/stationblanche/logs"
HISTORY_DAYS=15

# Préparer les répertoires nécessaires
mkdir -p "$QUARANTINE_DIR" "$SCAN_LOG_DIR"

# Fonction pour afficher une boîte de dialogue principale
main_menu() {
    zenity --list --radiolist \
        --title="Station Blanche" \
        --text="Choisissez une action :" \
        --column="Choix" --column="Action" \
        TRUE "Scanner une clé USB" \
        FALSE "Afficher l'historique des scans" \
        FALSE "Configurer les options" \
        FALSE "Quitter"
}

# Lancement séquentiel des antivirus
scan_usb() {
    USB_MOUNT_POINT="/media/usb"
    zenity --info --title="Scan en cours" --text="Lancement des scans..."

    # ClamAV scan
    clamscan --recursive --infected --move="$QUARANTINE_DIR" "$USB_MOUNT_POINT" \
        | tee "$SCAN_LOG_DIR/clamav_scan_$(date +%Y%m%d).log"

    # Eset scan
    /opt/eset/ees/bin/eav_scan --scan /media/usb \
        | tee -a "$SCAN_LOG_DIR/eset_scan_$(date +%Y%m%d).log"

    zenity --info --text="Scans terminés. Consultez les logs pour plus de détails."
}

# Afficher les options du menu principal
while true; do
    choice=$(main_menu)
    case $choice in
        "Scanner une clé USB") scan_usb ;;
        "Afficher l'historique des scans")
            zenity --text-info --title="Historique des scans" \
                --filename="$SCAN_LOG_DIR" --editable ;;
        "Quitter") break ;;
        *) zenity --error --text="Option inconnue. Réessayez." ;;
    esac
done
```

---

### **4. Mise en place de la journalisation sécurisée**
#### Étapes :
1. Configurer `logrotate` pour gérer la rotation des logs des scans :
   - Créez un fichier `/etc/logrotate.d/station_blanche` :
     ```bash
     /home/stationblanche/logs/*.log {
         daily
         missingok
         rotate 15
         compress
         delaycompress
         notifempty
         create 640 stationblanche stationblanche
     }
     ```

2. Activer la configuration :
   ```bash
   sudo logrotate -f /etc/logrotate.conf
   ```

---

### **5. Configuration des mises à jour automatiques**
#### Étapes :
1. **Automatiser les mises à jour de ClamAV :**
   - Ajouter une tâche `cron` pour `freshclam` :
     ```bash
     echo "0 2 * * * /usr/bin/freshclam" | sudo tee -a /etc/crontab
     ```

2. **Configurer les mises à jour d’Eset :**
   - Utiliser l’outil intégré dans Eset pour automatiser les mises à jour via l’interface graphique ou ligne de commande.

---

### **6. Automatisation du démarrage de l’application**
#### Étapes :
1. Ajouter le script au démarrage via `systemd` :
   - Créez un fichier `/etc/systemd/system/station_blanche.service` :
     ```ini
     [Unit]
     Description=Station Blanche USB Scanner
     After=multi-user.target

     [Service]
     ExecStart=/path/to/station_blanche.sh
     Restart=always
     User=stationblanche

     [Install]
     WantedBy=multi-user.target
     ```

   - Activer le service :
     ```bash
     sudo systemctl enable station_blanche.service
     sudo systemctl start station_blanche.service
     ```

---

### Suggestions :
**a.** Ajouter des tests pour vérifier que ClamAV et Eset scannent les clés USB comme prévu.  
**b.** Implémenter la phase 2 pour l'envoi de rapports par email.