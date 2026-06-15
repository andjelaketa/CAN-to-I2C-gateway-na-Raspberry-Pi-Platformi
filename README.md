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
