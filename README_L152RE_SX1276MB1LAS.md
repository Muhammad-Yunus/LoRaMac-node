# README - Konfigurasi Hardware STM32L152RE + SX1276MB1LAS (AS923)

Dokumen ini menjelaskan konfigurasi yang diperlukan untuk menjalankan project LoRaMac-node pada hardware **Nucleo-L152RE** dengan radio shield **SX1276MB1LAS** di region **AS923**.

---

## 1. Hardware Target

| Komponen | Spesifikasi |
|----------|-------------|
| MCU | STM32L152RE (Cortex-M3, 512KB Flash, 80KB RAM) |
| Board | Nucleo-L152RE (ST NUCLEO-L152RE) |
| Radio Shield | Seeed Studio SX1276MB1LAS |
| Secure Element | SOFT_SE (software-based) |
| Region | AS923 (Channel Plan Group AS923_1) |
| Class | Class A |

---

## 2. File Konfigurasi Utama

### 2.1 `.vscode/settings.json`

File ini mengontrol CMake build configuration untuk VS Code / Cursor.

```json
{
    "cmake.configureSettings": {
        "CMAKE_TOOLCHAIN_FILE": "cmake/toolchain-arm-none-eabi.cmake",
        "APPLICATION": "LoRaMac",
        "SUB_PROJECT": "periodic-uplink-lpp",
        "LORAWAN_DEFAULT_CLASS": "CLASS_A",
        "CLASSB_ENABLED": "ON",

        // Region configuration
        "ACTIVE_REGION": "LORAMAC_REGION_AS923",

        // Board dan radio shield
        "BOARD": "NucleoL152",
        "MBED_RADIO_SHIELD": "SX1276MB1LAS",

        // Secure element
        "SECURE_ELEMENT": "SOFT_SE",
        "SECURE_ELEMENT_PRE_PROVISIONED": "ON",

        // Region support activation
        "REGION_EU868": "OFF",
        "REGION_AS923": "ON",

        // Channel plan untuk AS923
        "REGION_AS923_DEFAULT_CHANNEL_PLAN": "CHANNEL_PLAN_GROUP_AS923_1"
    }
}
```

**Catatan Penting:**
- Default board di repo ini adalah `NucleoL073`, harus diubah ke `NucleoL152`
- Default shield adalah `SX1261MBXBAS`, harus diubah ke `SX1276MB1LAS`
- Region AS923 default-nya `OFF`, harus di-set ke `ON`
- CMake cache perlu dibersihkan saat pertama kali mengubah setting

---

### 2.2 `src/apps/LoRaMac/periodic-uplink-lpp/NucleoL152/main.c`

Menambahkan fallback default region ke AS923:

```c
#ifndef ACTIVE_REGION

#warning "No active region defined, LORAMAC_REGION_AS923 will be used as default."

#define ACTIVE_REGION LORAMAC_REGION_AS923

#endif
```

Ini memastikan jika `ACTIVE_REGION` tidak didefinisikan saat compile, akan default ke AS923 (bukan EU868).

---

### 2.3 `src/peripherals/soft-se/se-identity.h`

Konfigurasi identitas device untuk OTAA join:

```c
// Gunakan DevEUI statis dari konfigurasi ini
#define STATIC_DEVICE_EUI  1

// DevEUI device (big endian)
#define LORAWAN_DEVICE_EUI { 0xFF, 0xFF, 0xFF, 0xFF, 0x00, 0x00, 0x0C, 0x18 }

// JoinEUI/AppEUI (big endian)
#define LORAWAN_JOIN_EUI   { 0x11, 0x11, 0x11, 0x11, 0x11, 0x11, 0x11, 0x11 }

// AppKey (untuk LoRaWAN 1.0.x) / GenAppKey (untuk 1.1.x)
#define LORAWAN_APP_KEY    { 0x27, 0x97, 0xEA, 0xF9, 0x6C, 0x7F, 0x04, 0x53,
                              0x76, 0xCA, 0xFD, 0x05, 0xF1, 0x2C, 0xD3, 0x38 }

// NwkKey (untuk LoRaWAN 1.0.x)
#define LORAWAN_NWK_KEY    { 0x27, 0x97, 0xEA, 0xF9, 0x6C, 0x7F, 0x04, 0x53,
                              0x76, 0xCA, 0xFD, 0x05, 0xF1, 0x2C, 0xD3, 0x38 }
```

**Peta nama key LoRaWAN 1.0.x → 1.1.x:**

| 1.0.x | 1.1.x |
|-------|-------|
| LORAWAN_DEVICE_EUI | LORAWAN_DEVICE_EUI |
| LORAWAN_APP_EUI | LORAWAN_JOIN_EUI |
| LORAWAN_GEN_APP_KEY | LORAWAN_APP_KEY |
| LORAWAN_APP_KEY | LORAWAN_NWK_KEY |

---

### 2.4 `src/mac/LoRaMacCrypto.h`

Mengaktifkan random DevNonce untuk mencegah duplikasi nonce:

```c
// Gunakan random DevNonce (disarankan untuk soft-se)
#define USE_RANDOM_DEV_NONCE  1
```

---

### 2.5 `src/mac/LoRaMacCrypto.c`

Modifikasi untuk menggunakan `Radio.Random()` (hardware RNG dari SX1276) sebagai pengganti `SecureElementRandomNumber()` yang tidak tersedia di soft-se:

```c
#include "radio.h"  // Tambahkan include ini

// Di dalam LoRaMacCryptoPrepareJoinRequest():
#if ( USE_RANDOM_DEV_NONCE == 1 )
    uint32_t devNonce = 0;
    devNonce = Radio.Random( );  // Menggunakan hardware RNG SX1276
    CryptoNvm->DevNonce = devNonce;
#else
    CryptoNvm->DevNonce++;
#endif
```

---

## 3. Langkah Konfigurasi Build

### 3.1 Bersihkan CMake Cache

Karena cache CMake menyimpan setting lama, lakukan:

```bash
rm -rf build/
mkdir build
cd build
```

### 3.2 Konfigurasi dengan CMake

```bash
cmake -S .. -B . -G "Ninja" \
    -DCMAKE_TOOLCHAIN_FILE="../cmake/toolchain-arm-none-eabi.cmake" \
    -DTOOLCHAIN_PREFIX="<path-to-gnu-toolchain>" \
    -DACTIVE_REGION=LORAMAC_REGION_AS923 \
    -DREGION_AS923=ON \
    -DREGION_EU868=OFF \
    -DBOARD=NucleoL152 \
    -DMBED_RADIO_SHIELD=SX1276MB1LAS \
    -DCMAKE_BUILD_TYPE=Release \
    -DCLASSB_ENABLED=ON
```

### 3.3 Build

```bash
cmake --build . --config Release
```

### 3.4 Flash ke Device

```bash
STM32_Programmer_CLI -c port=swd -w build/src/apps/LoRaMac/LoRaMac-periodic-uplink-lpp.bin 0x08000000 -v
```

### 3.5 Reset Device

```bash
STM32_Programmer_CLI -c port=swd -rst
```

---

## 4. Verifikasi Output Serial

Device akan mengirim log melalui ST-Link VCP (COM9) pada baudrate 921600:

```
DevEui      : FF-FF-FF-FF-00-00-0C-18
JoinEui     : 11-11-11-11-11-11-11-11
Pin         : 00-00-00-00

###### =========== MLME-Request ============ ######
######               MLME_JOIN               ######
STATUS      : OK

###### =========== MCPS-Confirm ============ ######
STATUS      : OK

###### =====   UPLINK FRAME       1   ===== ######
CLASS       : A
TX PORT     : 2
TX DATA     : UNCONFIRMED
DATA RATE   : DR_5
U/L FREQ    : 923200000
TX POWER    : 7
```

---

## 5. Daftar File yang Dimodifikasi

| File | Perubahan |
|------|-----------|
| `.vscode/settings.json` | Region AS923, Board NucleoL152, Shield SX1276MB1LAS |
| `src/apps/LoRaMac/periodic-uplink-lpp/NucleoL152/main.c` | Default fallback region ke AS923 |
| `src/mac/LoRaMacCrypto.h` | `USE_RANDOM_DEV_NONCE = 1` |
| `src/mac/LoRaMacCrypto.c` | Gunakan `Radio.Random()` untuk DevNonce |
| `src/peripherals/soft-se/se-identity.h` | DevEUI, JoinEUI, AppKey, NwkKey |

---

## 6. Troubleshooting

### Masalah: Gateway EU868 menerima tapi Gateway AS923 tidak
**Solusi:** Pastikan `.vscode/settings.json` sudah di-set ke `LORAMAC_REGION_AS923` dan `REGION_AS923=ON`, lalu bersihkan CMake cache.

### Masalah: `DevNonce has already been used`
**Solusi:** Aktifkan `USE_RANDOM_DEV_NONCE = 1` di `LoRaMacCrypto.h`.

### Masalah: Build error `SecureElementRandomNumber undefined`
**Solusi:** Ganti `SecureElementRandomNumber()` dengan `Radio.Random()` di `LoRaMacCrypto.c`.

### Masalah: Channel plan salah
**Solusi:** Sesuaikan `REGION_AS923_DEFAULT_CHANNEL_PLAN` di `.vscode/settings.json` (default: `CHANNEL_PLAN_GROUP_AS923_1`).

---

## 7. Referensi

- [LoRaMac-node Documentation](https://github.com/Lora-net/LoRaMac-node)
- [AS923 Region Specification](https://lora-alliance.org/resource_hub/as-923-region-description/)
- [ChirpStack AS923 Configuration](https://www.chirpstack.io/)
