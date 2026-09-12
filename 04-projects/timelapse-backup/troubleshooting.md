# Home Assistant + RPi3 Timelapse
## Diagnostika problému, recovery fotografií a návrh Timelapse v2

**Dátum riešenia:** 12. 9. 2026

---

## 1. Pôvodný systém

Home Assistant vytvára každých 15 minút fotografiu z balkónovej kamery a ukladá ju cez sieť na SSD pripojené k Raspberry Pi 3.

Network storage v Home Assistante:

- názov: `linux_server_ssd`
- server: `192.168.1.149`
- Samba share: `storage`

RPi3 používa externé SSD:

```text
/dev/sda1
```

mountnuté na:

```text
/media/storage
```

Fotografie sú uložené v:

```text
timelapse/balcony/
```

Na RPi3 sa pripájam cez SSH alias:

```bash
ssh server-home
```

Server je možné bezpečne vypnúť cez Home Assistant. Po shutdown RPi3 HA približne po 3 minútach vypne smart zásuvku, z ktorej je server napájaný.

---

# 2. Problém

RPi3 bol raz vypnutý a neskôr znovu zapnutý.

Po zapnutí nebolo skontrolované, či sa Home Assistant opäť pripojil k sieťovému úložisku.

Po čase sa zistilo, že na RPi3 nepribúdajú nové fotografie.

Home Assistant zobrazoval:

```text
Network storage device failed
```

a:

```text
Could not set up linux_server_ssd
```

---

# 3. Diagnostický postup

Cieľom nebolo náhodne reštartovať zariadenia, ale postupne kontrolovať jednotlivé vrstvy:

```text
SSD
 ↓
Linux mount
 ↓
Samba služba
 ↓
Samba share
 ↓
sieť
 ↓
Home Assistant network mount
 ↓
HA automatizácia
 ↓
kamera
```

Týmto spôsobom sa dá presne určiť, na ktorej vrstve problém vznikol.

---

# 4. SSH pripojenie na RPi3

Použitý SSH alias:

```bash
ssh server-home
```

Pripojenie funguje pomocou existujúcej SSH konfigurácie/kľúča.

---

# 5. Kontrola SSD

Príkaz:

```bash
lsblk
```

Výsledok:

```text
sda
└─sda1  931.5G  /media/storage
```

Záver:

- RPi3 vidí SSD
- partícia `sda1` existuje
- filesystem je mountnutý
- mount point je `/media/storage`

SSD teda nebolo príčinou problému.

---

# 6. Kontrola Samba služby

Príkaz:

```bash
systemctl status smbd
```

Výsledok:

```text
active (running)
```

Samba teda bežala.

Dôležitá poznámka:

Bežiaca Samba ešte automaticky neznamená, že konkrétny share je správne nakonfigurovaný.

---

# 7. Kontrola Samba share

Príkaz:

```bash
testparm -s
```

Relevantná konfigurácia:

```text
[storage]
    path = /media/storage
    read only = No
    valid users = jan
```

Teda Samba share:

```text
storage
```

správne smeruje na:

```text
/media/storage
```

a používateľ `jan` má povolený zápis.

---

# 8. Kontrola Samby z Windows

V Prieskumníkovi Windows:

```text
\\192.168.1.149\storage
```

Share bol dostupný a bolo možné prezerať existujúce fotografie.

V tomto bode bolo potvrdené:

```text
RPi3              OK
SSD               OK
Linux mount       OK
Samba             OK
Samba share       OK
LAN               OK
```

Problém teda nebol na strane RPi3.

---

# 9. Kontrola Home Assistant network mountu

V:

```text
Settings → System → Storage
```

bolo viditeľné:

```text
linux_server_ssd
192.168.1.149:storage
```

Network storage však bolo v chybovom stave.

Po použití Reload Home Assistant oznámil:

```text
Cannot mount linux_server_ssd because there is existing data at
/data/media/linux_server_ssd.
Move it away first, then retry.
```

Toto bola kľúčová informácia.

---

# 10. Skutočná príčina problému

Keď bolo RPi3 vypnuté, Samba share nebol dostupný.

Home Assistant však ďalej každých 15 minút spúšťal automatizáciu kamery.

Pravdepodobný priebeh:

```text
RPi3 OFF
   ↓
network storage zmizne
   ↓
HA automatizácia ďalej fotografuje
   ↓
fotografie sa začnú zapisovať lokálne
   ↓
lokálne súbory vzniknú v mount pointe
   ↓
RPi3 sa neskôr zapne
   ↓
HA chce znovu mountnúť Samba share
   ↓
mount point už obsahuje lokálne dáta
   ↓
HA odmietne mount
```

Home Assistant tým chránil lokálne súbory pred prekrytím network mountom.

---

# 11. Home Assistant Terminal a kontajnery

Používaný Terminal & SSH add-on:

```text
[core-ssh ~]$
```

Pokus:

```bash
ls -lah /data/media/linux_server_ssd
```

vrátil:

```text
No such file or directory
```

To neznamenalo, že cesta Supervisora neexistuje.

`core-ssh` beží vo vlastnom kontajneri/prostredí a nevidí hostiteľský filesystem rovnakými cestami.

V Terminal add-one však fungovalo:

```bash
ls -lah /media
```

kde bolo:

```text
linux_server_ssd
```

Poučenie:

Rôzne komponenty Home Assistant OS môžu mať rozdielny pohľad na filesystem.

---

# 12. Recovery lokálnych dát

Home Assistant Repair ponúkol:

```text
Move blocking local data away
```

Pred potvrdením HA oznámil:

```text
No files are deleted.
```

Lokálne dáta mali byť presunuté do:

```text
linux_server_ssd_local_recovery
```

Po potvrdení vzniklo:

```text
/media/linux_server_ssd_local_recovery/
└── timelapse/
    └── balcony/
        └── fotografie
```

Zároveň sa:

```text
/media/linux_server_ssd
```

opäť stal skutočným network mountom smerujúcim na SSD RPi3.

---

# 13. Overenie časovej osi fotografií

Posledné fotografie na RPi3 pred výpadkom:

```text
2026-07-29 16:15
2026-07-29 16:30
2026-07-29 16:45
```

Najstaršia fotografia v recovery:

```text
2026-07-29 17:00
```

Recovery pokračovalo až po:

```text
2026-09-12 12:45
```

Po oprave network mountu sa na RPi3 objavila:

```text
2026-09-12 13:00
```

Časová os:

```text
RPi3                         RECOVERY                         RPi3

16:15 → 16:30 → 16:45 → 17:00 → ... → 12:30 → 12:45 → 13:00
```

Interval presne 15 minút.

Záver:

Automatizácia ani kamera neprestali fungovať.

Fotografie sa celý čas vytvárali, iba sa počas nedostupnosti RPi3 ukladali lokálne do Home Assistanta.

---

# 14. Počítanie súborov

Všeobecný príkaz:

```bash
find /cesta/k/adresaru -type f | wc -l
```

Recovery:

```bash
find /media/linux_server_ssd_local_recovery/timelapse/balcony -type f | wc -l
```

Výsledok:

```text
4303
```

Na RPi3 bolo pred prenesením recovery približne:

```text
1029
```

---

# 15. Zobrazenie najnovších súborov

```bash
ls -lt /cesta | head -5
```

Význam:

```text
-l = detailný výpis
-t = zoradiť podľa času
head -5 = ukázať prvých 5 riadkov
```

---

# 16. Zobrazenie najstarších súborov

```bash
ls -ltr /cesta | head -5
```

`-r` obráti poradie.

---

# 17. Veľkosť adresára

```bash
du -sh /cesta
```

Recovery malo približne:

```text
360M
```

Celý timelapse na RPi3 po prenose:

```text
462M
```

---

# 18. Kopírovanie recovery na RPi3

Fotografie boli najprv kopírované, nie presúvané.

Dôvod:

Ak by prenos zlyhal, recovery stále zostane zachované.

Príkaz:

```bash
cp -av /media/linux_server_ssd_local_recovery/timelapse/balcony/. /media/linux_server_ssd/timelapse/balcony/
```

Význam:

```text
cp = copy
-a  = archive, zachovanie metadát/časov
-v  = verbose, vypisovanie kopírovaných súborov
```

`/.` na konci zdrojovej cesty znamená obsah daného adresára.

---

# 19. Znak \ v shell príkazoch

Ak je dlhý príkaz rozdelený na viac riadkov:

```bash
prikaz \
dalsia_cast
```

znak:

```text
\
```

na konci riadku znamená:

```text
príkaz pokračuje na ďalšom riadku
```

Nie je súčasťou cesty.

---

# 20. Kontrola prenesených súborov

Po kopírovaní:

```text
Recovery: 4303
RPi3:     5343
```

RPi3 obsahovalo:

- pôvodné fotografie
- 4303 recovery fotografií
- nové fotografie vzniknuté po oprave

Preto bol počet vyšší.

---

# 21. BusyBox find

Pôvodne bol skúšaný príkaz s:

```text
find -printf
```

HA Terminal však používa BusyBox `find`, ktorý parameter:

```text
-printf
```

nepodporuje.

Chyba:

```text
find: unrecognized: -printf
```

Poučenie:

Nie všetky Linux utility majú na každom systéme rovnaké možnosti.

---

# 22. Porovnanie názvov recovery a serverových fotografií

Najprv sa vytvoril zoznam recovery fotografií:

```bash
find /media/linux_server_ssd_local_recovery/timelapse/balcony -type f | sed 's#.*/##' | sort > /tmp/recovery.txt
```

Potom zoznam fotografií na serveri:

```bash
find /media/linux_server_ssd/timelapse/balcony -type f | sed 's#.*/##' | sort > /tmp/server.txt
```

Porovnanie:

```bash
comm -23 /tmp/recovery.txt /tmp/server.txt
```

Príkaz nevypísal nič.

To znamená:

```text
Každý názov súboru z recovery existuje aj na RPi3.
```

Prenos 4303 recovery fotografií bol teda úspešný.

---

# 23. Odstránenie recovery

Pred odstránením bola overená cesta:

```bash
ls -ld /media/linux_server_ssd_local_recovery
```

Až potom:

```bash
rm -rf /media/linux_server_ssd_local_recovery
```

Význam:

```text
rm = remove
-r = recursive
-f = force
```

Teda:

```text
odstráň adresár a celý jeho obsah bez ďalšieho potvrdenia
```

DÔLEŽITÉ:

Pri použití:

```bash
rm -rf
```

vždy pred stlačením Enter skontrolovať celú cestu.

Nie je to Windows Kôš a príkaz sa nepýta na potvrdenie.

---

# 24. Kontrola po odstránení recovery

```bash
ls -lah /media
```

Výsledok obsahoval už iba:

```text
linux_server_ssd
```

Recovery bolo odstránené.

---

# 25. Konečný stav po oprave

```text
RPi3                 OK
SSD                  OK
Samba                OK
Network storage HA   OK
Automatizácia        OK
Nové fotografie      OK
4303 recovery fotiek prenesených
Recovery             odstránené
```

---

# 26. Hlavná lekcia z diagnostiky

Pri podobnom probléme neísť hneď cestou:

```text
reštartuj všetko
```

ale rozdeliť systém na vrstvy:

```text
hardware
 ↓
filesystem
 ↓
služba
 ↓
sieť
 ↓
mount
 ↓
aplikácia
 ↓
automatizácia
```

a postupne dokazovať:

```text
toto funguje
toto funguje
toto funguje
TOTO NEFUNGUJE
```

Tak sa problém výrazne zúži.

---

# TIMELAPSE V2

Počas diagnostiky vznikol nápad prerobiť celý systém.

Cieľ:

RPi3 nemusí bežať 24/7 iba kvôli archivácii fotografií.

---

# 27. Súčasná architektúra

```text
kamera
  ↓
Home Assistant
  ↓
network mount
  ↓
RPi3
  ↓
SSD
```

Nevýhoda:

Ak RPi3 nie je dostupné, vznikajú problémy s network mountom.

---

# 28. Nová architektúra

```text
KAMERA
   ↓
HOME ASSISTANT
   ↓
LOKÁLNY BUFFER
   ↓
fotografie každých 15 minút
   ↓
RPi3 môže byť OFF
   ↓
periodická synchronizácia
   ↓
RPi3 SSD
   ↓
dlhodobý archív
```

Home Assistant teda bude vždy fotografovať lokálne.

RPi3 bude iba archívny server.

---

# 29. Množstvo dát

Fotografia každých 15 minút:

```text
4 fotografie / hodina
96 fotografií / deň
672 fotografií / týždeň
```

Pri približne 100 kB na fotografiu:

```text
cca 67 MB / týždeň
```

Týždenný lokálny buffer je teda úplne rozumný.

---

# 30. Automatická týždenná synchronizácia

Napríklad sobota v noci:

```text
HA zapne smart zásuvku
        ↓
RPi3 začne bootovať
        ↓
HA čaká, kým je server SKUTOČNE online
        ↓
overí dostupnosť servera
        ↓
spustí synchronizáciu
        ↓
overí úspešnosť synchronizácie
        ↓
bezpečný Linux shutdown
        ↓
počká približne 3 minúty
        ↓
vypne smart zásuvku
```

Nechceme iba:

```text
zapni
počkaj 60 sekúnd
kopíruj
```

Lepšie je:

```text
zapni
 ↓
overuj dostupnosť
 ↓
server naozaj online
 ↓
pokračuj
```

---

# 31. Synchronizácia pri manuálnom zapnutí

Ak RPi3 zapnem sám:

```text
MANUAL power ON
      ↓
RPi3 boot
      ↓
HA zistí server online
      ↓
existujú nové fotografie?
      ↓
ÁNO
      ↓
synchronizácia
      ↓
server zostáva zapnutý
```

HA server po synchronizácii nesmie vypnúť, pretože na ňom môžem práve pracovať.

---

# 32. AUTO vs MANUAL session

Systém musí vedieť, prečo bol server zapnutý.

```text
AUTO
```

znamená:

```text
server zapla automatizácia kvôli synchronizácii
```

Po úspešnom prenose ho môže vypnúť.

```text
MANUAL
```

znamená:

```text
server zapol používateľ
```

Synchronizácia môže prebehnúť, ale server zostane zapnutý.

---

# 33. Lock

Lock znamená:

```text
zámok
```

V našom prípade:

```text
práve prebieha synchronizácia
```

Napríklad HA helper:

```text
input_boolean.timelapse_sync_running
```

Stavy:

```text
OFF = synchronizácia neprebieha
ON  = synchronizácia práve prebieha
```

---

# 34. Ochrana shutdownu počas synchronizácie

Situácia:

```text
15:30:00 začne synchronizácia
15:30:10 používateľ stlačí Vypnúť server
15:30:20 synchronizácia stále beží
```

Server sa nesmie okamžite vypnúť.

Logika:

```text
Používateľ chce shutdown
          ↓
sync_running?
     ↙         ↘
   NIE         ÁNO
    ↓           ↓
shutdown    zapamätaj požiadavku
                ↓
         počkaj na koniec sync
                ↓
           sync úspešný?
                ↓
             shutdown
```

---

# 35. Ochrana používateľa pred automatickým shutdownom

Ak server zapol používateľ:

```text
MANUAL session
```

automatizácia môže fotografie synchronizovať, ale nesmie server vypnúť.

Tým sa zabráni situácii:

```text
používateľ pracuje cez SSH
        ↓
sync skončí
        ↓
automatizácia vypne server
        ↓
SSH connection closed
```

---

# 36. rsync

Pre Timelapse v2 bude vhodnejší:

```bash
rsync
```

než jednoduché:

```bash
cp
```

alebo:

```bash
mv
```

`rsync` je určený na synchronizáciu súborov a umožní vytvoriť bezpečnejší proces.

Zásada:

```text
zdrojové fotografie neodstraňovať,
kým nie je potvrdený úspešný prenos
```

---

# 37. Mobilné notifikácie

Po automatickom cykle:

```text
📷 Timelapse backup dokončený

672 fotografií prenesených.
RPi3 bezpečne vypnuté.
Smart zásuvka vypnutá.
```

Pri manuálnom zapnutí:

```text
📷 Timelapse synchronizovaný

192 fotografií prenesených.
Server zostáva zapnutý – manuálne spustenie.
```

Pri chybe:

```text
⚠️ Timelapse backup zlyhal

Synchronizácia nebola úspešná.
Zdrojové fotografie zostali zachované.
Server nebol automaticky vypnutý.
```

---

# 38. Logging

Systém bude viesť log.

Príklad:

```text
2026-09-12 03:00 | AUTO   | Server power ON
2026-09-12 03:01 | AUTO   | Server online
2026-09-12 03:01 | SYNC   | 672 files found
2026-09-12 03:03 | SYNC   | 672 files transferred successfully
2026-09-12 03:03 | AUTO   | Shutdown successful
2026-09-12 03:06 | AUTO   | Power OFF

2026-09-14 15:30 | MANUAL | Server power ON
2026-09-14 15:31 | SYNC   | 192 files transferred successfully
2026-09-14 15:32 | MANUAL | Server remains ON
```

Log umožní spätne zistiť:

- kedy bol server zapnutý
- či bol AUTO alebo MANUAL
- koľko fotografií bolo prenesených
- či synchronizácia uspela
- či prebehol shutdown
- či bola vypnutá zásuvka
- prípadné chyby

---

# 39. Plán implementácie Timelapse v2

Projekt nebudeme vytvárať celý naraz.

Postup:

```text
1. lokálny buffer fotografií
        ↓
2. overenie lokálneho fotografovania
        ↓
3. ručná synchronizácia na RPi3
        ↓
4. rsync
        ↓
5. automatický sync pri zapnutí RPi3
        ↓
6. týždenné automatické zapnutie servera
        ↓
7. rozlíšenie AUTO / MANUAL
        ↓
8. bezpečný shutdown
        ↓
9. lock / ochrana synchronizácie
        ↓
10. mobilné notifikácie
        ↓
11. logging
```

Každú vrstvu najprv samostatne pochopiť a otestovať.

Až potom ju spojiť s ďalšou.

---

# Cieľ projektu

Výsledkom nemá byť iba automatizácia, ktorá nejako funguje.

Cieľom je vytvoriť systém, pri ktorom rozumiem:

```text
čo robí
prečo to robí
kde sú uložené dáta
ako prebieha synchronizácia
ako sa zisťuje stav servera
ako sa server bezpečne vypína
ako sa systém chráni pred chybami
ako ho diagnostikovať
```

Projekt tak slúži zároveň ako praktické učenie:

- Home Assistant
- Linux
- SSH
- filesystem
- mount
- Samba
- sieťové úložisko
- rsync
- automatizácie
- stavová logika
- lock
- bezpečný shutdown
- notifikácie
- logging
- troubleshooting
