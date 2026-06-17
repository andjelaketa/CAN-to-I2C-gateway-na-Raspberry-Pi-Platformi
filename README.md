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

### 2.1 I2C Transfer funkcija (`i2c_transfer`)
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
Navedena funckija obavlja čitanje i upis u registar. Funkcija prima parametre:
* **int fd**: File Descriptor. Opisuje fajl za otvoren I2C interfejs. Preko ovog parametra funkcija zna na kom hardverskom I2C magistralnom vodu treba da izvrši komunikaciju.
* **int addr**: I2C adresa ciljnog uređaja na magistrali sa kojm želimo da komuniciramo
* **unsigned char reg**: adresa specifičnog registra unutar ciljnog I2C uređaja iz kojeg želimo da čitammo podatke ili u koji želimo podatke da upišemo.
* **unsigned chat *data **: Pokazivač na niz bajtova (bafer). Ukoliko se vrši operacija čitanja, pročitani podaci že biti smješteni u taj bafer. Ukoliko se vrđi upis, podaci koji treba da se upišu se nalaze u tom baferu.
* **int len**: broj bajtova koji se čita ili upisuje.
* **int is_read**: kontrolna promjenljiva koja ima vrijednost 1 ako želimo da izvršimo čitanje i 0 ako se vrši upis.
Ova funkcija koristi standardni Linux _ioctl_ (I/O control) meganizam za komunikaciju sa I2C uređajem. Rad funkcije se oslanja na strukture i2c_msg (definiše jednu I2C poruku) i _i2c_rdwr_ioctl_data_ (koja grupiše više poruka u jedan transfer).

### 2.1 Is register R/W funkcija (`is_register_rw`)
```c
int is_register_rw(unsigned char reg) {

    if (reg == 0x31 || reg == 0x38) {
        return 1;
    }
    if (reg >= 0x1D && reg <= 0x2A) {
        return 1;
    }
    if (reg >= 0x2C && reg <= 0x2F) {
        return 1;
    }

    return 0;
}
```
Navedena funckija obavlja provjeru da li je registar kojem korisnik želi pristupiti R/W. 
Na linku https://www.alldatasheet.com/html-pdf/254714/AD/ADXL345/384/13/ADXL345.html se nalazi datasheet I2C senzora koji je korišten u okviru ovog projekta. U tabeli registara senzora, navedeno je koji su registri R/W (read/write) a koji samo R (read) te je pomoću ove funkcije izvršena provjera da li korisnik pristupa registru koji dozvoljava upis. 
Funkcija prima samo jedan parametar:
* **unsigned char reg**: heksadecimalna adresa I2C registra u koju korisnik želi da upiše podatke
Funkcija vraća vrijednost 1 ako je registar konfigurisan kao Read/Write, a vraća 0 ako je registar konfigurisan kao Read-only.


### 2.3 Glavna funkcija aplikacije (`main`)
```c
int main(int argc, char *argv[]) {
    if (argc < 3) return 1;
   
    int can_sock = socket(PF_CAN, SOCK_RAW, CAN_RAW);
    struct ifreq ifr;
    strcpy(ifr.ifr_name, argv[1]);
    ioctl(can_sock, SIOCGIFINDEX, &ifr);
   
    struct sockaddr_can addr = { .can_family = AF_CAN, .can_ifindex = ifr.ifr_ifindex };
    bind(can_sock, (struct sockaddr *)&addr, sizeof(addr));
   
    int i2c_fd = open("/dev/i2c-1", O_RDWR);
    unsigned char my_dev_id = (unsigned char)atoi(argv[2]) & 0x0F;
   
    static unsigned char last_addr = 0, last_reg = 0;

    while (1) {
        struct can_frame frame;
        if (read(can_sock, &frame, sizeof(frame)) < 0) continue;

        unsigned char rx_dev_id = (frame.can_id >> 7) & 0x0F;
        if (rx_dev_id != my_dev_id) continue;

        printf("\n[PRIMLJENO] ID: 0x%03X | RTR: %s | Len: %d",
               frame.can_id & 0x7FF,
               (frame.can_id & CAN_RTR_FLAG) ? "DA" : "NE",
               frame.can_dlc);

        unsigned char i2c_addr = frame.can_id & 0x7F;
        unsigned char reg = (frame.can_id & CAN_RTR_FLAG) ? last_reg : frame.data[0];
       
        struct can_frame tx_frame = { .can_id = frame.can_id & CAN_SFF_MASK, .can_dlc = 2 };
        int status = 0;
        int valid_rw = 1;

        if (!(frame.can_id & CAN_RTR_FLAG)) {
            last_addr = i2c_addr;
            last_reg = reg;
           
            if ((frame.data[1] >> 7) & 0x01) {
                printf(" | Tip: Citanje (Reg: 0x%02X)", reg);
                status = i2c_transfer(i2c_fd, i2c_addr, reg, &tx_frame.data[2], 1, 1);
                tx_frame.can_dlc = 3;
            } else {
                printf(" | Tip: Upis (Reg: 0x%02X, Data: 0x%02X)", reg, frame.data[2]);
               

                if (is_register_rw(reg)) {
                    status = i2c_transfer(i2c_fd, i2c_addr, reg, &frame.data[2], frame.can_dlc - 2, 0);
                } else {
                    printf(" -> БЛОКИРАНО: Регистар је Read-Only!");
                    valid_rw = 0;
                }
            }
        } else {
            printf(" | Tip: RTR Citanje (Reg: 0x%02X)", last_reg);
            status = i2c_transfer(i2c_fd, last_addr, last_reg, &tx_frame.data[2], 1, 1);
            tx_frame.can_dlc = 3;
        }


        if (status < 0 || !valid_rw) {
            if (status < 0) printf(" -> I2C GRESKA!");
            tx_frame.data[1] = 0x04;
        } else {
            printf(" -> I2C USPEH!");
            tx_frame.data[1] = (frame.can_id & CAN_RTR_FLAG) ? 0x80 : 0x00;
        }

        tx_frame.data[0] = reg;
       
        usleep(1000);
        write(can_sock, &tx_frame, sizeof(tx_frame));
        printf("\n[POSLATO] Statusni okvir poslat.\n");
    }
    return 0;
}
```
Rad main funkcije dijelimo na dva dijela:
* **Inicijalizacija:**: dio main funkcije do while petlje
* **Filtriranje uređaja:**: while petlja u main funkciji
Inicijalizacija se itvršava samo jednom prilikom pokretanja programa. Funckija osigurava se da je program pokrenut sa odgovarajućim brojem parametara iz komandne linije te vrši podešavanje CAN socketa i I2C interfejsa. Takođe, u segmentu inicijalizacije, program pamti svoj jedinstveni identifikator koji je unešen preko komandne linije, i inicijalizuje na nulu statičke promjenljive last_addr i last_reg koje će biti korištene prilikom RTR okvira.

Nakon inicijalizacije pokreće se beskonačna petlja koja neprestano osluškuje i obrađuje dolazne CAN poruke. Petlja izvlači ID ciljnog uređaja iz 11-bitnog CAN ID-ja te frejm ignoriše ako se ne poklapa sa my_dev_id (postavljen u okviru inicijalizacije). 
Program se grana u zavisnosti od toga da li je primljen standardni CAN frejm ili RTR frejm.
Ako je primljen standardni frejm, prvo se ažuriraju statičke promljenljive last_addr i last_reg, zatim se provjerava da li je u piranju čitanje ili upis.
*Ako je u pitanju čitanje, poziva se funkcija `i2c_transfer` da pročita 1 bajt iz I2C registra. 
*Ako je u pitanju upis, prvo se poziva funkcija `is_register_rw`, pa tek ako je upis dozvoljen, funkcijia `i2c_transfer`.
*Ako je u pitanju RTR okvir, program automatski inicira čitanje koristeći promjenljive last_addr i last_reg.

Nakon pokušaja I2C komunikacije, program provjerava da li je bilo grešaka i u skladu sa tim ispisuje odgovarajuću poruku korisniku, formira odgovor i na kraju pauzira 1ms prije slanja odgovora i povratka na početak petlje.


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
> [!NOTE]
> Prilikom testiranja aplikacije, (navođenja registara u koje će se vršiti upis ili iz kojih će se vršiti čitanje) mora se voditi računa o statusu registra. Neki registri I2C senzora koji je kokrišten u okviru projekta, podržavaju samo čitanje (read only). Korisnik će biti obaviješten, porukom, da pokušava izvršiti upis u takav registar, međutim čitanje je dozvoljeno i sprovodi se bez dodatnih komentara korisniku.

## 5. Značajni segmenti koda
### 5.1. Funkcija  `int i2c_transfer(int fd, int addr, unsigned char reg, unsigned char *data, int len, int is_read)`
Navedena funkcija izvršava čitanje i upis u registar. Zadnji parametar koji funkcija prima je 'int is_read' te pomoću njega definišemo da li se radi o čitanju iz registra ( ako jeste ima vrijednost 1 ) ili o upisu u registar (vrijednost 0).




