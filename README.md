# ansible_labkey_install

Ansible playbook pro automatizovanou instalaci **LabKey Server v24+** na Debian.

## Architektura

```
Klient → nginx:443 (SSL, certifikát)
              ↓
         LabKey:8080 (plain HTTP, interní)
```

Nginx zajišťuje SSL a přesměrování HTTP→HTTPS. LabKey tomcat běží pouze interně na portu 8080.

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

| Soubor | Zdroj | Potřebný pro |
|--------|-------|-------------|
| `OpenJDK17U-jdk_x64_linux_hotspot_*.tar.gz` | [adoptium.net](https://adoptium.net) | `install.yml` |
| `LabKey*-community*.tar.gz` | [labkey.com/download](https://www.labkey.com/download-community-edition/) | `install.yml` |
| `rstudio-server-*.deb` | [posit.co/download/rstudio-server](https://posit.co/download/rstudio-server/) | `rstudio.yml` |
| `ROracle_*.tar.gz` | [CRAN archive](https://cran.r-project.org/src/contrib/Archive/ROracle/) | `oracle.yml` |
| `instantclient-basic-linux.x64-*.zip` | [oracle.com](https://www.oracle.com/database/technologies/instant-client/linux-x86-64-downloads.html) (vyžaduje login) | `oracle.yml` |
| `instantclient-sdk-linux.x64-*.zip` | [oracle.com](https://www.oracle.com/database/technologies/instant-client/linux-x86-64-downloads.html) (vyžaduje login) | `oracle.yml` |

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
# Odkomentovat požadovanou verzi LabKey:
labkey_archive:  "LabKey25.11.0-1-community.tar.gz"
labkey_dist_dir: "LabKey25.11.0-1-community"

# Před ostrým nasazením změnit:
labkey_db_pass:        "silné-heslo"
labkey_encryption_key: "přesně-32-znaků-dlouhý-klíč!!"

# Nginx / SSL — vyplnit před spuštěním ssl.yml:
labkey_hostname: "labkey.firma.cz"   # hostname přidělený správcem sítě
ssl_cert_file:   "labkey.crt"        # soubory uložit do files/
ssl_key_file:    "labkey.key"        # privátní klíč — není v gitu!
```

### 4. SSH klíč

```bash
ssh-copy-id -i ~/.ssh/id_ed25519 uživatel@<IP adresa>
```

## Spuštění

```bash
# Test konektivity a sudo
ansible-playbook playbooks/ping.yml

# Kompletní instalace (vše najednou)
ansible-playbook playbooks/install.yml playbooks/ssl.yml playbooks/rstudio.yml playbooks/oracle.yml playbooks/profile.yml
```

Jednotlivé playboky lze spouštět i samostatně — viz sekce níže.

### Čistá reinstalace (smazání dat)

```bash
# POZOR: smaže všechna data v DB, vyžádá potvrzení
ansible-playbook playbooks/drop-db.yml

# Poté znovu nainstalovat
ansible-playbook playbooks/install.yml
```

### HTTPS (certifikát ve formátu PEM)

1. Uložit certifikát a klíč do `files/`:
   ```
   files/labkey.crt
   files/labkey.key
   ```
   Privátní klíč (`.key`) je v `.gitignore` — do gitu se nedostane.

2. Nastavit hostname v `group_vars/labkey_servers.yml`:
   ```yaml
   labkey_hostname: "labkey.firma.cz"
   ```

3. Spustit:
   ```bash
   ansible-playbook playbooks/ssl.yml
   ```

## Co playboky nainstalují

### `install.yml`

| Krok | Co se provede |
|------|--------------|
| APT | postgresql, r-base, texinfo, uuid-runtime a další závislosti |
| Adresáře | `/labkey/{labkey,src,apps,backupArchive}` |
| Java | JDK 17 → `/labkey/apps/`, symlink `/usr/local/java` |
| PostgreSQL | Přesun datového adresáře do `/usr/local/pgsql/data`, vytvoření DB a uživatele |
| LabKey | Rozbalení distribuce, deploy JAR + konfigurace do `/labkey/labkey/` |
| Systemd | Instalace a spuštění služby `labkey_server` na portu 8080 |

### `ssl.yml`

| Krok | Co se provede |
|------|--------------|
| Nginx | Instalace nginx |
| Certifikát | Zkopírování `.crt` a `.key` do `/etc/ssl/labkey/` |
| Konfigurace | Nasazení reverse proxy — HTTPS:443 → LabKey:8080, HTTP:80 → redirect na HTTPS |

### `drop-db.yml`

Smaže a znovu vytvoří databázi LabKey. Zastaví LabKey, odstraní lock soubor, dropne DB a uživatele, vytvoří je znovu. **Vyžádá interaktivní potvrzení.** Použít před čistou reinstalací.

### `rstudio.yml` (volitelné)

| Krok | Co se provede |
|------|--------------|
| Závislosti | lib32gcc, libclang, libssl a další systémové knihovny |
| Pandoc | Instalace z Debian repozitáře |
| RStudio Server | Instalace z lokálního `.deb`, spuštění služby `rstudio-server` |

RStudio Server bude dostupný na `http://<server>:8787`. Přihlášení přes systémové uživatelské účty.

Před spuštěním uložit `.deb` do `files/` a zkontrolovat název souboru v `group_vars`:
```yaml
rstudio_deb: "rstudio-server-2024.09.1-394-amd64.deb"
```

> **Poznámka k Pandocu:** Pandoc se instaluje z Debian repozitáře (nevyžaduje lokální soubor).

### `oracle.yml` (volitelné)

Instaluje Oracle Instant Client a zkompiluje R balíček ROracle pro přístup k Oracle DB z R scriptů v LabKey.

| Krok | Co se provede |
|------|--------------|
| APT | libaio1, unzip |
| Oracle IC | Rozbalení Basic + SDK do `/opt/oracle/instantclient_*/` |
| ldconfig | Registrace sdílených knihoven |
| profile.d | `/etc/profile.d/oracle_client.sh` — ORACLE_HOME, LD_LIBRARY_PATH |
| ROracle | `R CMD INSTALL` s OCI_LIB/OCI_INC, výsledek v `/usr/local/lib/R/site-library/ROracle` |

Před spuštěním uložit do `files/`:
- `instantclient-basic-linux.x64-*.zip` a `instantclient-sdk-linux.x64-*.zip` — z [oracle.com](https://www.oracle.com/database/technologies/instant-client/linux-x86-64-downloads.html) (vyžaduje účet Oracle)
- `ROracle_*.tar.gz` — z [CRAN archive](https://cran.r-project.org/src/contrib/Archive/ROracle/)

### `backup.yml` / `restore.yml`

**Záloha** (pg_dump + konfigurační soubory):
```bash
ansible-playbook playbooks/backup.yml
```
Zálohy se ukládají do `{{ labkey_backup }}/` (výchozí `/labkey/backupArchive/`):
- `db_YYYYMMDD_HHMMSS.sql.gz` — pg_dump
- `files_YYYYMMDD_HHMMSS.tar.gz` — adresář `config/`

**Obnova:**
```bash
# Zobrazit dostupné zálohy
ansible-playbook playbooks/restore.yml

# Obnovit konkrétní zálohu (POZOR: přepíše stávající DB)
ansible-playbook playbooks/restore.yml -e backup_db_file=db_20260313_153045.sql.gz
```

### `upgrade.yml`

Upgrade LabKey na novou verzi (zachová `application.properties`):

1. V `group_vars/labkey_servers.yml` odkomentovat novou verzi
2. Nový tarball uložit do `files/`
3. Spustit:
   ```bash
   ansible-playbook playbooks/upgrade.yml
   ```

### `profile.yml`

Nastaví prostředí pro administrátory z linuxové skupiny `labkey`:
- `/etc/profile.d/labkey_env.sh` — LABKEY_HOME, JAVA_HOME, aliasy
- `/labkey/admin/` — administrační skripty (backup.sh, restart.sh, status.sh, log.sh)

```bash
ansible-playbook playbooks/profile.yml
```

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
/etc/ssl/labkey/       # SSL certifikát a klíč (po spuštění ssl.yml)
```

## Správa služeb

```bash
# LabKey
systemctl status labkey_server
systemctl restart labkey_server
journalctl -u labkey_server -f

# Nginx
systemctl status nginx
systemctl reload nginx

# RStudio Server
systemctl status rstudio-server
systemctl restart rstudio-server
```

## Před ostrým nasazením

1. **Hesla** — změnit `labkey_db_pass` a `labkey_encryption_key` v `group_vars/labkey_servers.yml`, až poté spustit `install.yml`
2. **SMTP** — nakonfigurovat v `/labkey/labkey/config/application.properties`
3. **Firewall** — otevřít porty 80 a 443, porty 8080 a 8787 ponechat pouze lokálně dostupné
4. **Zálohování** — nastavit zálohy PostgreSQL a adresáře `/labkey/labkey/`
