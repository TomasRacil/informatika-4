# **02. Síťování a Port Forwarding**

Nejčastější frustrace začátečníků: *"Nainstaloval jsem server, ip a mi ukazuje IP adresu, ale z Windows se na ni nemůžu pingnout ani připojit."*

## **Teorie: Síťové režimy ve VirtualBoxu**

VirtualBox se chová jako virtuální router. Nabízí několik režimů zapojení:

### **1. NAT (Network Address Translation) - Výchozí**

* **Jak to funguje:** VM je schovaná za vaším PC. Má přístup ven (internet funguje), ale **zvenku se na ni nikdo nedostane** (ani váš hostitelský PC).  
* **Analogie:** VM je jako zařízení v domácí síti, váš PC je router.  
* **Výhoda:** Bezpečné, funguje vždy (škola, doma, kavárna).  
* **Nevýhoda:** Musíte ručně otevírat porty (Port Forwarding), abyste se p5ipojili.

### **2. Bridged Adapter (Most)**

* **Jak to funguje:** VM se připojí přímo do fyzické sítě (školní Wi-Fi/LAN). Dostane IP adresu od školního routeru, stejně jako váš notebook.  
* **Výhoda:** VM je vidět jako plnohodnotné zařízení v síti.  
* **Nevýhoda:**  
  * Může selhat, pokud je školní síť zabezpečená (MAC filtering).  
  * Při změně sítě (škola -> doma) se změní IP adresa serveru.

## **Praktická úloha: Nastavení Port Forwardingu (NAT)**

Použijeme režim **NAT**, protože je nejstabilnější pro notebooky, které přenášíte. Abychom se ale dostali dovnitř přes SSH, musíme udělat "díru" (přesměrování).

### **Cíl**

Chceme, aby platilo:

* `localhost:2222` (Váš PC) -> `vm-ip:22` (Linux Server SSH)  
* `localhost:8080` (Váš PC) -> `vm-ip:80` (Linux Server Web)

### **Postup**

1. Vypněte virtuální stroj (`poweroff`).  
2. Ve VirtualBoxu klikněte na **Settings** -> **Network**.  
3. Ujistěte se, že **Adapter 1** je **NAT**.  
4. Rozklikněte **Advanced** -> tlačítko **Port Forwarding**.  
5. Přidejte dvě pravidla (tlačítko Plus vpravo):

| Name | Protocol | Host IP | Host Port | Guest IP | Guest Port |
| :---- | :---- | :---- | :---- | :---- | :---- |
| SSH | TCP | 127.0.0.1 | **2222** | (nevyplňovat) | **22** |
| WEB | TCP | 127.0.0.1 | **8080** | (nevyplňovat) | **80** |

6. Potvrďte **OK**. Spusťte virtuální stroj.

### **Ověření**

Otevřete terminál ve svém hostitelském systému (PowerShell ve Windows / Terminál v macOS) a zkuste:

```shell
# Pozor: Připojujeme se na localhost a port 2222!  
ssh -p 2222 student@127.0.0.1
```

Pokud se vás zeptá na heslo a přihlásí vás, máte splněno.

## **Úkoly**

---

### 1. Úkol: Rychlý web 
V materiálech jsme forwardovali port 80. Zkusme si ale vytvořit vlastní službu na nestandardním portu, abyste si ověřili, že chápete princip.

* **Zadání:** Spusťte ve VM dočasný webový server na portu **9000** a zpřístupněte ho na svém PC na portu **9999**.
* **Postup:**
    1.  Ve VM vytvořte soubor: `echo "<h1>Ahoj ze serveru</h1>" > index.html`.
    2.  Spusťte server: `python3 -m http.server 9000` (Python začne servírovat aktuální složku).
    3.  Ve VirtualBoxu (Nastavení -> Sítě -> Port Forwarding) přidejte pravidlo: **Host 9999 -> Guest 9000**.
    4.  Otevřete na svém PC prohlížeč a jděte na `http://localhost:9999`.

---

### 2. Úkol: Princip NAT
Ověřte si v praxi, jak funguje "jednosměrné zrcadlo" NATu.

* **Zadání:** Dokázat, že VM vidí ven, ale hostitel nevidí (přímo) dovnitř.
* **Postup:**
    1.  **VM -> Ven:** V terminálu virtuálky napište `ping -c 3 google.com`. (Mělo by fungovat – server má internet).
    2.  **Ven -> VM:** Zjistěte IP adresu virtuálky příkazem `ip a` (hledáme rozhraní `eth0` nebo `enp0s3`, obvykle **10.0.2.15**).
    3.  Otevřete příkazový řádek na svém **hostitelském PC** (PowerShell/Terminál) a zkuste: `ping 10.0.2.15`.
    4.  **Výsledek:** Ping neprojde (Request timed out). To je důkaz, že bez Port Forwardingu se na tuto IP adresu přímo nedostanete.

---

### 3. Úkol: IP adresa
Když se připojíte přes SSH, server vás musí vidět. Ale jakou IP adresu uvidí, když jste schovaní za NATem?

* **Zadání:** Zjistěte, z jaké IP adresy se serveru *jeví*, že jste připojeni.
* **Postup:**
    1.  Připojte se na server přes SSH (`ssh -p 2222 student@127.0.0.1`).
    2.  Přímo v SSH terminálu spusťte příkaz `w` (who) nebo `ss -tna | grep :22`.
    3.  **Otázka:** Vidíte tam svou skutečnou IP adresu počítače, nebo adresu **10.0.2.2**? (Adresa 10.0.2.2 je ve světě VirtualBoxu "Virtual Router/Gateway" – server si myslí, že komunikuje s routerem).

---

### 4. Úkol: Bind Address
*Tento problém trápí vývojáře denně. Pokud ho pochopíte teď, ušetříte si hodiny debugování v Dockeru.*

Když spouštíte server, můžete mu říct, na kterých síťových kartách má naslouchat.
* `0.0.0.0` = Všechny síťové karty (přijímám spojení odkudkoli).
* `127.0.0.1` = Pouze Loopback (přijímám spojení **jen sám od sebe**).

**Zadání:**
1.  Ukončete Python server z Úkolu 1 (Ctrl+C).
2.  Spusťte ho znovu, ale přinuťte ho naslouchat **pouze** lokálně:
    `python3 -m http.server --bind 127.0.0.1 9000`
3.  Zkuste se na něj připojit z prohlížeče přes port forwarding (`http://localhost:9999`).
4.  **Výsledek:** Nepůjde to. Spojení bude odmítnuto.
5.  **Vysvětlení:** Port forwarding přichází na síťovou kartu `eth0` (10.0.2.15). Váš server ale tuto kartu ignoruje a "poslouchá" jen na vnitřním `lo` (127.0.0.1). Proto ho VirtualBox router nemůže kontaktovat.