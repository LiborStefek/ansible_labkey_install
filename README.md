# ansible_labkey_install

Ansible playbook pro automatizovanou instalaci **LabKey Server v24+** (embedded) na Debian.

## Architektura

```
Klient → nginx:443 (SSL, certifikát od lokální CA)
              ↓
         LabKey:8080 (plain HTTP, interní)
```

Nginx zajišťuje SSL terminaci a přesměrování HTTP→HTTPS. LabKey běží pouze interně na portu 8080.

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

# Plná instalace LabKey
ansible-playbook playbooks/install.yml
```

### HTTPS (po obdržení certifikátu od správce sítě)

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

### `rstudio.yml` (volitelné)

| Krok | Co se provede |
|------|--------------|
| Závislosti | lib32gcc, libclang, libssl a další systémové knihovny |
| Pandoc | Instalace z Debian repozitáře |
| RStudio Server | Instalace z lokálního `.deb`, spuštění služby `rstudio-server` |

RStudio Server bude dostupný na `http://<server>:8787`. Přihlášení přes systémové uživatelské účty.

Před spuštěním uložit `.deb` do `files/` a zkontrolovat název souboru v `group_vars`:
```yaml
rstudio_deb: "rstudio-server-2023.12.1-402-amd64.deb"
```

> **Poznámka k Pandocu:** Pandoc se instaluje z Debian repozitáře (nevyžaduje lokální soubor).
> Verze z repozitáře nemusí odpovídat verzi v `.deb` použité při ručné instalaci — pro LabKey reporty to není podstatné.

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

1. **Hesla** — změnit `labkey_db_pass` a `labkey_encryption_key` v `group_vars/labkey_servers.yml`, poté znovu spustit `install.yml`
2. **SMTP** — nakonfigurovat v `/labkey/labkey/config/application.properties`
3. **Firewall** — otevřít porty 80 a 443, porty 8080 a 8787 ponechat pouze lokálně dostupné
4. **Zálohování** — nastavit zálohy PostgreSQL a adresáře `/labkey/labkey/`
