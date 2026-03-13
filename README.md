# ansible_labkey_install

Ansible playbook pro automatizovanou instalaci **LabKey Server v24+** (embedded) na Debian.

## Požadavky

**Řídicí stanice** (odkud se Ansible spouští):
- Ansible 2.9+
- SSH klíč s přístupem na cílový server

**Cílový server:**
- Debian 12
- Uživatel se sudo právy
- Kořenový adresář `/labkey/` existuje
- Bez přístupu k internetu — binární soubory se kopírují lokálně

## Příprava

### 1. Binární soubory

Stáhnout a uložit do `files/`:

| Soubor | Zdroj |
|--------|-------|
| `OpenJDK17U-jdk_x64_linux_hotspot_*.tar.gz` | [adoptium.net](https://adoptium.net) |
| `LabKey*-community-embedded.tar.gz` | [labkey.com/download](https://www.labkey.com/download-community-edition/) |

### 2. Inventory

Upravit `inventory/hosts.ini` — nastavit IP adresu cílového serveru:

```ini
[labkey_servers]
labkey-debian ansible_host=<IP adresa>

[labkey_servers:vars]
ansible_user=<uživatel se sudo>
```

### 3. Konfigurace

Upravit `inventory/group_vars/labkey_servers.yml`:

```yaml
# Názvy souborů musí odpovídat tomu, co je v files/
jdk_archive:     "OpenJDK17U-jdk_x64_linux_hotspot_17.0.18_8.tar.gz"
jdk_dir:         "jdk-17.0.18+8"
labkey_archive:  "LabKey24.7.0-1-community-embedded.tar.gz"
labkey_dist_dir: "LabKey24.7.0-1-community"

# Před ostrým nasazením změnit:
labkey_db_pass:        "silné-heslo"
labkey_encryption_key: "přesně-32-znaků-dlouhý-klíč!!"
```

### 4. SSH klíč

```bash
ssh-copy-id -i ~/.ssh/id_ed25519 uživatel@<IP adresa>
```

## Spuštění

```bash
# Test konektivity a sudo
ansible-playbook playbooks/ping.yml

# Plná instalace
ansible-playbook playbooks/install.yml
```

## Co playbook nainstaluje

| Krok | Co se provede |
|------|--------------|
| APT | postgresql, r-base, texinfo, uuid-runtime a další závislosti |
| Adresáře | `/labkey/{labkey,src,apps,backupArchive}` |
| Java | JDK 17 → `/labkey/apps/`, symlink `/usr/local/java` |
| PostgreSQL | Přesun datového adresáře do `/usr/local/pgsql/data`, vytvoření DB a uživatele |
| LabKey | Rozbalení distribuce, deploy JAR + konfigurace do `/labkey/labkey/` |
| Systemd | Instalace a spuštění služby `labkey_server` |

## Adresářová struktura po instalaci

```
/labkey/
  labkey/              # LABKEY_HOME — JAR, konfigurace, tmp
    labkeyServer.jar
    config/
      application.properties
    labkey-tmp/
  src/                 # rozbalená distribuce
  apps/                # JDK
  backupArchive/       # zálohy

/usr/local/java        # symlink na JDK
/usr/local/pgsql/data  # PostgreSQL data
```

## Správa služby

```bash
systemctl status labkey_server
systemctl restart labkey_server
journalctl -u labkey_server -f
```

## Před ostrým nasazením

1. **Hesla** — změnit `labkey_db_pass` a `labkey_encryption_key` v `group_vars/labkey_servers.yml`, poté znovu spustit playbook
2. **Firewall** — otevřít port 80 (výchozí) nebo jiný dle `server.port` v `application.properties`
3. **SMTP** — nakonfigurovat v `/labkey/labkey/config/application.properties`
4. **Zálohování** — nastavit zálohy PostgreSQL a adresáře `/labkey/labkey/`
