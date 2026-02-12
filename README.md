# Raspberry Pi Zero 2 W + 3.5" ST7796 (SPI) + CardKB (I2C)

Działający setup konsoli tekstowej na ekranie ST7796 + klawiatura CardKB jako wirtualna klawiatura Linux (`uinput`).

## 1. Sprzęt

- Raspberry Pi Zero 2 W
- Wyświetlacz 3.5" TFT ST7796 (moduł SPI z IM0/IM1/IM2)
- M5Stack CardKB (I2C, adres `0x5F`)
- Raspberry Pi OS Lite (Bookworm)

## 2. Połączenia

| Moduł | Pin modułu | Raspberry Pi (pin fizyczny) | GPIO | Opis |
|---|---|---:|---:|---|
| LCD ST7796 | VCC | 4 | 5V | zasilanie LCD |
| LCD ST7796 | GND | 6 | GND | masa |
| LCD ST7796 | SCL | 23 | GPIO11 | SPI SCLK |
| LCD ST7796 | SDA | 19 | GPIO10 | SPI MOSI |
| LCD ST7796 | CS | 24 | GPIO8 | SPI CE0 |
| LCD ST7796 | DC | 15 | GPIO22 | data/command |
| LCD ST7796 | RST | 13 | GPIO27 | reset |
| LCD ST7796 | BL | 12 | GPIO18 | podświetlenie |
| CardKB | VCC | 4 | 5V | zasilanie klawiatury |
| CardKB | GND | 6 | GND | masa |
| CardKB | SDA | 3 | GPIO2 | I2C SDA1 |
| CardKB | SCL | 5 | GPIO3 | I2C SCL1 |

## 3. Zworki IM na płytce LCD

Ustawienie działające dla tego projektu:

- `IM0 = 1`
- `IM1 = 1`
- `IM2 = 1`

## 4. Włączenie interfejsów i pakietów

```bash
sudo apt update
sudo apt full-upgrade -y

sudo raspi-config nonint do_spi 0
sudo raspi-config nonint do_i2c 0

sudo apt install -y i2c-tools python3-smbus2 python3-evdev
sudo modprobe uinput
echo uinput | sudo tee /etc/modules-load.d/uinput.conf >/dev/null
