# Automatická konfigurácia Firefoxu pre Burpsuite certifikát na Kali Linux
Tento Bash skript automaticky nastaví proxy server v prehliadači Firefox a importuje požadovaný certifikát nástroja Burpsuite. Skript je kompatibilný s Linuxovou disribúciou Debian a testovaný bol na Kali Linux.
## Funkcionalita skriptu

- `Overenie a inštalácia potrebných balíkov` - Skript skontroluje, či je prítomný nástroj certutil potrebný na import certifikátov do Firefoxu. Ak nie je nainštalovaný, automaticky sa nainštaluje balík `libnss3-tool`s (platné len pre Debian).
- `Stiahnutie certifikátu` - Certifikát je možné stiahnuť z vopred definovanej URL, v tomto prípade ale v skripte nie je definované. 
- `Import certifikátu do Firefoxu` - Skript automaticky detekuje predvolený Firefox profil a importuje certifikát, ak ešte nie je pridaný.
- `Nastavenie proxy servera vo Firefoxe`  - Proxy je nastavené pre HTTP a HTTPS na testovanú IP `10.0.2.15` a port `8080`. Konfigurácia je uložená do súboru `prefs.js` v profile Firefoxu.

## Požiadavky
- `Operačný systém` - Debian (napr. Kali Linux).
- `Root oprávnenia` - Skript potrebuje vedieť root heslo.
- `Firefox` - Musí byť nainštalovaný v systéme.
- `Certifikát` - Súbor `cacert.der` musí byť umiestnený v priečinku `/Downloads/`.

## Použitie skriptu
Vytvorenie súboru `firefox_ca_script.sh` prostredníctvom príkazu `touch`. Priradiť spustitelné práva súboru príkazom `chmod +x firefox_ca_script.sh` a spustiť ho príkazom `bash firefox_ca_script.sh`.

## 1 Popis skriptu
Toto heslo sa použije na spustenie príkazov vyžadujúcich administrátorské oprávnenia bez manuálneho zadania.

```bash
ROOT_PASSWORD="kali"
```
## 2 Overenie a inštalácia potrebných balíkov
Skript kontroluje, či je nástroj `certutil` dostupný v systéme. Ak nie je nainštalovaný, inštaluje ho pomocou apt balíčkovača pre Debian.

```bash
if ! command -v certutil &> /dev/null; then
    echo "certutil nebol najdeny. Instalujem potrebne baliky..."
    if [ -f /etc/debian_version ]; then
        echo $ROOT_PASSWORD | sudo -S apt update && echo $ROOT_PASSWORD | sudo -S apt install -y libnss3-tools
    else
        echo "Tento skript podporuje iba Debian"
        exit 1
    fi
else
    echo "certutil je uz nainstalovany."
fi
```

## 3 Definovanie ciest certifikátu a profilu
Definuje cestu k certifikátu, alias certifikátu a umiestnenie profilov Firefoxu.

```bash
CERT_FILE="$HOME/Downloads/cacert.der"
CERT_NICKNAME="BurpCA"
FIREFOX_PROFILE_DIR="$HOME/.mozilla/firefox"
```

## 4 Nájsť Firefox profil
Skript vyhľadá predvolený Firefox profil, ktorý obsahuje `default-esr` označenie.

```bash
FIREFOX_PROFILE_DIR="$HOME/.mozilla/firefox"
PROFILE=$(find "$FIREFOX_PROFILE_DIR" -maxdepth 1 -type d -name "*.default-esr" -o -name "*.default" | head -n 1)

if [ -z "$PROFILE" ]; then
    echo "Firefox profil nebol najdeny!"
    exit 1
fi
```
## 5 Overenie a import certifikátu
Skript kontroluje, či certifikát už existuje v certifikačnej databáze Firefoxu. Ak nie, certifikát sa importuje s označením ako dôveryhodný pre TLS spojenia a certifikačnú autoritu.

```bash
if certutil -L -d "$PROFILE" | grep -q "$CERT_NICKNAME"; then
    echo "Certifikat uz existuje vo Firefox profile. Preskakujem import."
else
    certutil -A -n "$CERT_NICKNAME" -t "TCu,Cu,Tu" -i "$CERT_FILE" -d "$PROFILE"
    if [ $? -ne 0 ]; then
        echo "Nepodarilo sa importovat certifikat!"
        exit 1
    fi
    echo "Certifikat bol uspesne importovany."
fi
```

## 6 Nastavenie proxy servera vo Firefoxe
Skript pridáva do súboru `prefs.js` nastavenia proxy servera. Nastavenia sú použité pre HTTP aj HTTPS pripojenia. Proxy je nastavená na IP 10.0.2.15 a port 8080 a zmeny sa prejavia po reštartovaní Firefoxu ideálne príkazom `pkill firefox`. Po úspešnom dokončení, skript informuje používateľa.

```bash
PREFS_FILE="$PROFILE/prefs.js"

echo "user_pref(\"network.proxy.type\", 1);" >> "$PREFS_FILE"
echo "user_pref(\"network.proxy.http\", \"10.0.2.15\");" >> "$PREFS_FILE"
echo "user_pref(\"network.proxy.http_port\", 8080);" >> "$PREFS_FILE"
echo "user_pref(\"network.proxy.ssl\", \"10.0.2.15\");" >> "$PREFS_FILE"
echo "user_pref(\"network.proxy.ssl_port\", 8080);" >> "$PREFS_FILE"
echo "user_pref(\"network.proxy.share_proxy_settings\", true);" >> "$PREFS_FILE"
echo "user_pref(\"network.proxy.no_proxies_on\", \"\");" >> "$PREFS_FILE"

echo "Proxy nastavenia boli uspesne aplikovane. Restartujte Firefox pre uplatnenie zmien."
```
#### Autor: M
