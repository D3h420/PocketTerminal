# Raspberry Pi Zero 2 W + 3.5" ST7796 (SPI) + CardKB (I2C)

Działający setup konsoli tekstowej na ekranie ST7796 + klawiatura CardKB jako wirtualna klawiatura Linux (`uinput`).
![IMG_8349](https://github.com/user-attachments/assets/1f3419e5-5778-4272-ac83-d4440f506f9a)

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

## 3. Zworki IM na płytce LCD (zostały domyślnie)

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
```

## 5. Konfiguracja LCD (config.txt)

W pliku config.txt zostaw dla LCD ten blok:
```
[all]
dtparam=spi=on
dtoverlay=mipi-dbi-spi,spi0-0,write-only,speed=10000000
dtparam=width=320,height=480
dtparam=reset-gpio=27,dc-gpio=22,backlight-gpio=18
```

## panel.bin (działający init + poprawne kolory)

6.1 Generowanie panel.bin
```
python3 - <<'PY'
out = bytearray(b"MIPI DBI" + b"\x00"*7 + b"\x01")

def cmd(c, *args):
    out.extend([c, len(args), *args])

def delay(ms):
    while ms > 255:
        out.extend([0x00, 0x01, 255]); ms -= 255
    out.extend([0x00, 0x01, ms])

cmd(0x01); delay(150)
cmd(0x11); delay(120)
cmd(0x36, 0x48)
cmd(0x3A, 0x55)
cmd(0xF0, 0xC3); cmd(0xF0, 0x96)
cmd(0xB4, 0x02)
cmd(0xB7, 0xC6)
cmd(0xC0, 0xC0, 0x00)
cmd(0xC1, 0x13)
cmd(0xC2, 0xA7)
cmd(0xC5, 0x21)
cmd(0xE8, 0x40,0x8A,0x1B,0x1B,0x23,0x0A,0xAC,0x33)
cmd(0xE0, 0xD2,0x05,0x08,0x06,0x05,0x02,0x2A,0x44,0x46,0x39,0x15,0x15,0x2D,0x32)
cmd(0xE1, 0x96,0x08,0x0C,0x09,0x09,0x25,0x2E,0x43,0x42,0x35,0x11,0x11,0x28,0x2E)
cmd(0xF0, 0x3C); cmd(0xF0, 0x69)
cmd(0x20)          # INVOFF (fix kolorów)
cmd(0x29); delay(20)

open('/tmp/panel.bin','wb').write(out)
print('panel.bin bytes:', len(out))
PY

sudo install -m 644 /tmp/panel.bin /lib/firmware/panel.bin
sudo reboot
```

6.2 Checksum działającej wersji

Rozmiar: 112 bajtów
SHA256: 7b00dcc47ba563843ead08f595edae6c994fdf44cdab23afd52c11be9d43a6bb

## CardKB -> klawiatura Linux (uinput)

7.1 Skrypt
```
sudo tee /usr/local/bin/cardkb-uinput.py >/dev/null <<'PY'
#!/usr/bin/env python3
import time
from smbus2 import SMBus
from evdev import UInput, ecodes as e

I2C_BUS = 1
ADDR = 0x5F

KEYMAP = {}
letters = "abcdefghijklmnopqrstuvwxyz"
for ch in letters:
    KEYMAP[ch] = (getattr(e, f"KEY_{ch.upper()}"), False)
    KEYMAP[ch.upper()] = (getattr(e, f"KEY_{ch.upper()}"), True)

for d in "0123456789":
    KEYMAP[d] = (getattr(e, f"KEY_{d}"), False)

KEYMAP.update({
    ' ': (e.KEY_SPACE, False),
    '-': (e.KEY_MINUS, False), '_': (e.KEY_MINUS, True),
    '=': (e.KEY_EQUAL, False), '+': (e.KEY_EQUAL, True),
    '[': (e.KEY_LEFTBRACE, False), '{': (e.KEY_LEFTBRACE, True),
    ']': (e.KEY_RIGHTBRACE, False), '}': (e.KEY_RIGHTBRACE, True),
    '\\': (e.KEY_BACKSLASH, False), '|': (e.KEY_BACKSLASH, True),
    ';': (e.KEY_SEMICOLON, False), ':': (e.KEY_SEMICOLON, True),
    "'": (e.KEY_APOSTROPHE, False), '"': (e.KEY_APOSTROPHE, True),
    ',': (e.KEY_COMMA, False), '<': (e.KEY_COMMA, True),
    '.': (e.KEY_DOT, False), '>': (e.KEY_DOT, True),
    '/': (e.KEY_SLASH, False), '?': (e.KEY_SLASH, True),
    '`': (e.KEY_GRAVE, False), '~': (e.KEY_GRAVE, True),
})

SPECIAL = {
    8:(e.KEY_BACKSPACE, False), 9:(e.KEY_TAB, False),
    10:(e.KEY_ENTER, False), 13:(e.KEY_ENTER, False),
    27:(e.KEY_ESC, False), 127:(e.KEY_BACKSPACE, False),
}

def emit(ui, keycode, shift=False, ctrl=False):
    if shift: ui.write(e.EV_KEY, e.KEY_LEFTSHIFT, 1)
    if ctrl: ui.write(e.EV_KEY, e.KEY_LEFTCTRL, 1)
    ui.write(e.EV_KEY, keycode, 1); ui.write(e.EV_KEY, keycode, 0)
    if ctrl: ui.write(e.EV_KEY, e.KEY_LEFTCTRL, 0)
    if shift: ui.write(e.EV_KEY, e.KEY_LEFTSHIFT, 0)
    ui.syn()

def map_code(code):
    if code in SPECIAL:
        return (*SPECIAL[code], False)
    if 1 <= code <= 26:
        ch = letters[code-1]
        return (getattr(e, f"KEY_{ch.upper()}"), False, True)
    try:
        ch = chr(code)
    except ValueError:
        return None
    if ch in KEYMAP:
        k, sh = KEYMAP[ch]
        return (k, sh, False)
    return None

def main():
    keys = {v[0] for v in KEYMAP.values()} | {v[0] for v in SPECIAL.values()} | {e.KEY_LEFTSHIFT, e.KEY_LEFTCTRL}
    ui = UInput({e.EV_KEY: list(keys)}, name="cardkb-uinput")
    bus = SMBus(I2C_BUS)

    last = None
    next_repeat = 0
    repeat_delay = 0.4
    repeat_rate = 0.05

    while True:
        try:
            code = bus.read_byte(ADDR)
        except OSError:
            time.sleep(0.05); continue

        if code == 0:
            last = None; time.sleep(0.01); continue

        m = map_code(code)
        if not m:
            time.sleep(0.01); continue

        keycode, shift, ctrl = m
        now = time.time()

        if code != last:
            emit(ui, keycode, shift, ctrl)
            last = code
            next_repeat = now + repeat_delay
        elif now >= next_repeat:
            emit(ui, keycode, shift, ctrl)
            next_repeat = now + repeat_rate

        time.sleep(0.005)

if __name__ == "__main__":
    main()
PY

sudo chmod +x /usr/local/bin/cardkb-uinput.py
```

7.2 Usługa systemd (autostart)
```
sudo tee /etc/systemd/system/cardkb.service >/dev/null <<'SERVICE'
[Unit]
Description=CardKB I2C keyboard
After=multi-user.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/bin/cardkb-uinput.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
SERVICE

sudo systemctl daemon-reload
sudo systemctl enable --now cardkb.service
```

## quick diag

```
i2cdetect -y 1
# CardKB powinno pokazać 0x5f

systemctl status cardkb.service --no-pager -l
journalctl -u cardkb.service -b --no-pager | tail -n 80

dmesg | grep -Ei 'panel-mipi-dbi|spi0|firmware'
```
