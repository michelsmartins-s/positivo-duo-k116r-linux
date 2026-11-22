# Ubuntu 24.04 no Positivo Duo K116R

Guia para deixar o Ubuntu 24.04 funcionando no **Positivo Duo K116R** (2 em 1, Intel Celeron Gemini Lake, 11,6"), com as soluções para os problemas que quase ninguém documenta: Wi-Fi sem interface, toque desalinhado, tela piscando e modo tablet.

> English: see [README.md](README.md)

## Status

| Componente | Status | Observação |
|---|---|---|
| Vídeo (Intel UHD, i915) | Funciona | Precisa de parâmetros do kernel |
| Wi-Fi (RTL8723BU) | Funciona, intermitente | A interface pode sumir em algum boot (erro de EFuse). Reset de energia resolve |
| Bluetooth | Funciona | Mesmo chip do Wi-Fi |
| Touchscreen (Silead MSSL1680) | Funciona | Precisa de firmware e correção de escala |
| Teclado / touchpad | Funciona | Desativar no modo tablet exige um script |
| Câmera | Detectada | Não testada a fundo |
| Áudio | Detectado | Não testado a fundo |
| Rotação de tela | Não testado | |

## Sistema testado

- Ubuntu 24.04, kernel `6.8.0-40-generic`
- BIOS AMI Aptio 2.20.1273 (2021), somente UEFI (sem CSM)
- Menu de boot: `F7`. Setup: `Del` ou `F2`
- Armazenamento: eMMC (`mmcblk1`)

## 1. Tela piscando / sessão gráfica reiniciando

Em `/etc/default/grub`:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash i915.enable_psr=0 i915.enable_dc=0 usbcore.autosuspend=-1"
```

Depois rode `sudo update-grub`. **Não use `acpi=off`**: ele desativa o toque, o teclado e o eMMC.

## 2. Wi-Fi sem interface (sem `wlan0`)

Sintoma: o `lsusb` mostra o RTL8723BU, o módulo `rtl8xxxu` está carregado, o firmware existe, mas o `ip link` mostra só `lo`. No `dmesg`:

```
usb 1-4: Fatal - failed to read EFuse
rtl8xxxu: probe of 1-4:1.2 failed with error -5
```

O que funcionou:

1. Desligue o aparelho.
2. Tire o carregador.
3. Segure o botão de energia por 15 a 20 segundos (nada aparece na tela).
4. Espere 1 minuto, conecte o carregador e ligue.

A interface passou a aparecer como `wlx<mac>`. O problema pode voltar em outro boot, então isso é um contorno, não uma correção garantida. Não rode `modprobe rtl8xxxu` várias vezes com o chip em mau estado: pode travar.

## 3. Touchscreen não funciona

```
silead_ts i2c-MSSL1680:00: Direct firmware load for silead/mssl1680.fw failed with error -2
```

O firmware **não** está no pacote `linux-firmware`. Pegue no repositório da comunidade:

```
sudo apt install -y git
git clone https://github.com/onitake/gsl-firmware.git
sudo mkdir -p /lib/firmware/silead
sudo cp $(find gsl-firmware -name "gsl1680-positivo-c4128b.fw" | head -1) /lib/firmware/silead/mssl1680.fw
sudo update-initramfs -u
```

O arquivo `gsl1680-positivo-c4128b.fw` (de outro modelo Positivo) funcionou no K116R. O firmware não está incluído neste repositório.

## 4. Toque funciona, mas o ponteiro está deslocado

Sintoma: o canto superior esquerdo está certo, o centro cai perto do canto e o canto inferior direito cai perto do meio da tela.

Causa: o driver anuncia `0-4095` nos dois eixos, mas o painel reporta cerca de `0-1890` (X) e `0-1258` (Y). Para medir:

```
sudo evtest /dev/input/eventN | grep --line-buffered ABS_MT_POSITION
```

Crie `/etc/udev/hwdb.d/99-silead-touch.hwdb` (veja [files/99-silead-touch.hwdb](files/99-silead-touch.hwdb)):

```
evdev:name:silead_ts:*
 EVDEV_ABS_35=0:1900:1
 EVDEV_ABS_36=0:1270:1
```

Cada linha `EVDEV_ABS` começa com um espaço. Aplique:

```
sudo systemd-hwdb update
sudo udevadm trigger
sudo reboot
```

Regras udev com `LIBINPUT_CALIBRATION_MATRIX` não tiveram efeito no GNOME neste caso.

## 5. Desativar teclado e touchpad no modo tablet

Não testamos se o aparelho avisa o Linux quando é dobrado (rode `sudo libinput debug-events` dobrando a tela para checar). Enquanto isso, o jeito é manual: `evtest --grab` no teclado interno e `gsettings` para o touchpad. Arquivos: [files/toggle-teclado](files/toggle-teclado) e [files/toggle-teclado.desktop](files/toggle-teclado.desktop).

Instalação:

```
sudo cp files/toggle-teclado /usr/local/bin/toggle-teclado
sudo chmod +x /usr/local/bin/toggle-teclado
mkdir -p ~/.local/share/applications
cp files/toggle-teclado.desktop ~/.local/share/applications/
sudo visudo -f /etc/sudoers.d/toggle-teclado
```

No `visudo`, adicione (troque `seuusuario`):

```
seuusuario ALL=(root) NOPASSWD: /usr/bin/evtest, /usr/bin/pkill
```

Dica: ao testar, deixe um teclado externo ou o teclado na tela à mão. Se travar, segure o botão de energia por 5 segundos: o `--grab` não sobrevive ao reinício.

## 6. Caiu no Shell EFI

Se o aparelho abrir em `Shell>`, inicie manualmente:

```
map -r
FS0:
FS0:\EFI\ubuntu\shimx64.efi
```

Depois conserte de vez:

```
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
sudo update-grub
```

## 7. Nota sobre o Ubuntu 26.04

Um teste inicial com o Ubuntu 26.04 abriu com vídeo funcionando, mas sem Wi-Fi. Este guia é baseado no 24.04.

## Contribuindo

Testado em uma única unidade. Issues e pull requests com resultados de outras unidades ou distribuições são bem-vindos.

## Licença

MIT. Os firmwares pertencem aos seus respectivos autores.
