# CAN-to-I2C Gateway na Raspberry Pi Platformi

Ovaj projekat predstavlja softversko rješenje implementirano u programskom jeziku C koje omogućava transport I2C transakcija preko CAN (Controller Area Network) magistrale. Uređaj funkcioniše kao gateway (mrežni prolaz) koji prihvata specifično kodovane CAN okvire, dekoduje ih, izvršava ekvivalentne operacije na lokalnoj I2C magistrali Raspberry Pi-ja, te vraća statusni odgovor nazad na CAN mrežu.

---

## 1. Teorijska podloga i arhitektura sistema

Ugrađeni sistemi često zahtijevaju premošćavanje između različitih komunikacionih protokola u zavisnosti od topologije mreže i zahtjeva za robusnošću.

* **CAN (Controller Area Network):** Diferencijalni, serijski komunikacioni protokol dizajniran za robusne industrijske i automobilske aplikacije. Koristi 11-bitne (Standard) ili 29-bitne (Extended) identifikatore za arbitražu i adresiranje poruka, a ne samih čvorova. Podržava **RTR (Remote Transmission Request)** okvire za eksplicitan zahtjev za podacima.
* **I2C (Inter-Integrated Circuit):** Sinhroni, master-slave protokol zasnovan na dvije linije (SDA i SCL) sa 7-bitnim adresiranjem čvorova (slaves) i internom strukturom registara unutar samih čvorova.

### Protokol mapiranja (Enkapsulacija)
Da bi se I2C transakcija prenijela preko CAN magistrale, mapiranje je izvršeno unutar 11-bitnog CAN ID-a i Data polja (DLC) na sljedeći način:

#### 11-bitni CAN Identifikator (CAN ID)
| Biti | Opis |
| :--- | :--- |
| `ID[10:7]` (4 bita) | **ID Uređaja (Device ID):** Selektuje konkretan Raspberry Pi gateway na mreži (konfigurabilni parametar). |
| `ID[6:0]` (7 bita) | **I2C Adresa:** Direktno mapira 7-bitnu adresu ciljnog I2C slave uređaja. |

#### CAN Data Polje (Payload)
Struktura bajtova u CAN okviru zavisi od operacije, ali generalno prati pravilo:
* **Byte 0:** Adresa internog registra I2C čvora (`reg`).
* **Byte 1 (Kontrolni/Statusni bajt):**
  * Najviši bit (`MSB` / `Bit 7`): Određuje smjer transakcije. `0` za **Upis**, `1` za **Čitanje**.
  * Ostali biti: Služe za prenos statusnih informacija i detekciju grešaka.
* **Byte 2 do Byte N:** Podaci koji se upisuju (kod upisa) ili pročitani podaci (u povratnom okviru).

---

## 2. Analiza softverske implementacije

Aplikacija je realizovana u C-u koristeći standardni Linux **SocketCAN** interfejs i **i2c-dev** drajverski podsistem preko `ioctl` poziva.

### 2.1 I2C Transfer Funkcija (`i2c_transfer`)
Za razliku od standardnih `read()` i `write()` sistemskih poziva koji često automatski šalju *STOP* sekvencu, ova funkcija koristi `I2C_RDWR` ioctl zahtjev. Ovo omogućava generisanje **Repeated START** kondicije, što je kritično za ispravno čitanje registara kod većine I2C senzora (prvo se upisuje adresa registra, pa se bez otpuštanja magistrale vrši čitanje).

```c
int i2c_transfer(int fd, int addr, unsigned char reg, unsigned char *data, int len, int is_read) {
    struct i2c_msg msgs[2];
    struct i2c_rdwr_ioctl_data msgset;
    unsigned char reg_buf = reg;

    msgs[0].addr = addr;
    msgs[0].flags = 0;
    msgs[0].buf = &reg_buf;
    msgs[0].len = 1;

    msgset.msgs = msgs;
    msgset.nmsgs = 1;

    if (is_read) {
        msgs[1].addr = addr;
        msgs[1].flags = I2C_M_RD;
        msgs[1].buf = data;
        msgs[1].len = len;
        msgset.nmsgs = 2;
    } else {
        msgs[0].len = 1 + len;
        unsigned char buf[9];
        buf[0] = reg;
        memcpy(buf + 1, data, len);
        msgs[0].buf = buf;
    }
    return ioctl(fd, I2C_RDWR, &msgset);
}
```
---

### 2.2 Glavna petlja aplikacije (main)
* **Inicijalizacija:** Kreira se sirovi mrežni socket za CAN (SOCK_RAW), vezuje se za proslijeđeni mrežni interfejs (can0), i otvara se fajl deskriptor za lokalni I2C kontroler (/dev/i2c-1).
* **Filtriranje uređaja:** Aplikacija prihvata samo okvire čiji se gornji 4 bita CAN ID-a poklapaju sa lokalno zadatim my_dev_id.
* **Traženje stanja (RTR memorija):** Koriste se statičke promjenljive last_addr i last_reg kako bi gateway zapamtio koja je bila posljednja adresirana lokacija, što omogućava korektno izvršavanje nadolazećeg RTR okvira.
> [!NOTE]
> Kod aplikacije objašnjen je na kraju dokumenta, u sekciji 5. 
---

## 3. Matrica testnih scenarija i komande
Za demonstraciju rada korišćen je Raspberry Pi kao gateway, dok se simulacija generisanja CAN okvira vrši pomoću Linux `cansend` i `candump` alata iz can-utils paketa.

U primjerima ispod pretpostavićemo da je aplikacija pokrenuta sa parametrom za Device ID = 2 (0x2), a I2C slave uređaj se nalazi na adresi 0x3A.
Iz toga slijedi bazni **CAN ID = (0x2 << 7) | 0x3A = 0x13A.**

| Scenario | Opis operacije | Komanda za slanje (cansend) | Očekivani odgovor (candump) |
| :--- | :--- | :--- | :--- |
| **1. Uspješan upis** | Upis vrijednosti 0xFF u registar 0x10 | `cansend can0 13A#1000FF` | `can0 13A [2] 10 00` |
| **2. Uspješno čitanje** | Čitanje 1 bajta iz registra 0x15 | `cansend can0 13A#1580` | `can0 13A [3] 15 80 DA` *(gdje je DA podatak)* |
| **3. RTR Čitanje** | Ponovno čitanje iz zadnjeg registra | `cansend can0 13A#R` | `can0 13A [3] 15 80 DA` |
| **4. I2C Greška** | Pokušaj pristupa nepostojećoj I2C adresi | `cansend can0 17F#0000` *(adresa 0x7F)* | `can0 17F [2] 00 04` |

---
### 3.1 Detaljna analiza testnih slučajeva

#### Scenario 1: Uspješan upis podataka
Želimo upisati vrijednost `0xFF` u registar `0x10` I2C uređaja na adresi `0x3A`.

* **CAN ID:** `0x13A`
* **Data polje poruke:** `10 00 FF`
    * `Data[0] = 0x10` (Registar)
    * `Data[1] = 0x00` (Bit 7 je 0 $\rightarrow$ upis)
    * `Data[2] = 0xFF` (Podatak)

**Pokretanje:**
```bash
cansend can0 13A#1000FF
```
Očekivani povratni okvir (candump):
```bash
can0  13A   [2]  10 00
```
**Objašnjenje:** Gateway vraća *DLC=2*. *Data[0]=0x10* (potvrda registra), *Data[1]=0x00* (status uspješnog upisa, nema postavljenih bita greške).


#### Scenario 2: Uspješno standardno čitanje podataka
Želimo upročitti podatak iz registra `0x15`.

* **CAN ID:** `0x13A`
* **Data polje poruke:** `15 80`
    * `Data[0] = 0x15` (Registar)
    * `Data[1] = 0x80` (Bit 7 je 0 $\rightarrow$ čitanje, binarno 1000 0000)

**Pokretanje:**
```bash
cansend can0 13A#1580
```
Očekivani povratni okvir (candump):
```bash
can0  13A   [3]  15 80 DA
```
**Objašnjenje:** Gateway vraća *DLC=3*. *Data[0]=0x15* (potvrda registra), *Data[1]=0x80* (status uspješnog čitanja, nema postavljenih bita greške), , *Data[2]=DA* (pročitani podatak sa I2C magistrale).

> [!NOTE]
> Ukoliko želimo poslati RTR okvir, koristi se komanda sa sufiksom `#R` (npr. `cansend can0 13A#R`). U tom slučaju očekujemo povratni ispis koji je potpuno analogan standardnom čitanju podataka (kao u Scenariju 2), jer gateway automatski vraća posljednju poznatu vrijednost iz memorije.

#### Scenario 3: Pokušaj čitanja sa nepostojeće adrese / nepostojećeg registra
Želimo izvršiti čitanje iz registra `0x99` koji ne postoji na I2C uređaju, ili sa I2C adrese koja je ispravna ali ne podržava taj specifični registar.

* **CAN ID:** `0x13A`
* **Data polje poruke:** `99 80`
    * `Data[0] = 0x99` (Nepostojeći/nevalidan registar)
    * `Data[1] = 0x80` (Bit 7 je 1 $\rightarrow$ Čitanje)

**Pokretanje:**
```bash
cansend can0 13A#9980
```
Očekivani povratni okvir (candump):
```bash
can0  13A   [2]  99 04
```
**Objašnjenje:** Gateway pokušava izvršiti I2C transakciju čitanja za registar `0x99`. Budući da uređaj šalje NACK na pokušaj pristupa ovoj adresi registra, funkcija `i2c_transfer` javlja grešku. Gateway vraća DLC=2, gdje `Data[0]=0x99` potvrđuje ciljani registar, a statusni bajt `Data[1]=0x04` signalizira grešku na magistrali (analogno ponašanju u Scenariju 4).
## 4. Uputstvo za pokretanje aplikacije

### Kompajliranje
Kompajliranje se vrši pomoću `arm-linux-gnueabihf-gcc` prevodioca namijenjenog za ARM ciljne platforme (cross-compilation):

```bash
arm-linux-gnueabihf-gcc -o can_i2c_gateway main.c
```
*Pokretanje*
Aplikacija zahtijeva dva argumenta komandne linije: ime CAN interfejsa i željeni Device ID (u opsegu 0-15).
```bash
sudo ./can_i2c_gateway can0 2
```
### Prilikom testiranja aplikacije, (navođenja registara u koje će se vršiti upis ili iz kojih će se vršiti čitanje) mora se voditi računa o statusu registra. Neki registri I2C senzora koji je kokrišten u okviru projekta, podržavaju samo čitanje (read only). Korisnik će biti obaviješten, porukom, da pokušava izvršiti upis u takav registar, međutim čitanje je dozvoljeno i sprovodi se bez dodatnih komentara korisniku.

## 5. Značajni segmenti koda
###5.1. Funkcija int i2c_transfer(int fd, int addr, unsigned char reg, unsigned char *data, int len, int is_read)
Navedena funkcija izvršava čitanje i upis u registar. Zadnji parametar koji funkcija prima je 'int is_read' te pomoću njega definišemo da li se radi o čitanju iz registra ( ako jeste ima vrijednost 1 ) ili o upisu u registar (vrijednost 0).



