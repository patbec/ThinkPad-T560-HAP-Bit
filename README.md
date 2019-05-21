# ThinkPad T560 HAP-Bit setzen

![Banner](/Images/banner.jpg)
<p align="right">
<a href="https://github.com/patbec/ThinkPad-T560-HAP-Bit/blob/master/README_EN.md">Read this page in English</a>
</p>

## Beschreibung

Hier ein kleines Tutorial wie auf einem ThinkPad mit Skylake CPU die Intel Management Engine (ME) vollständig deaktiviert werden kann.
Das Ganze wird durch setzen des HAP-Bits ermöglicht und erfordert das flashen des EEPROMs mit einer modifizierten Firmware.

Dieses Tutorials wie anhand eines ThinkPad T560 (`20FH0023GE`) mit der **ME 11.6** Firmware erklärt.

### Vorbereitung

Wir brauchen ein externes Gerät um den EEPROM auszulesen und mit einer veränderten Firmware neu zu beschreiben. Die Laptop CPU selbst hat auf die erforderten Flash-Bereiche weder Lese- noch Schreibzugriff.

Hardware
- WINGONEER EEPROM Routing USB Programmer **CH341A Writer** LCD Flash für 25 SPI Serie 24 I2C
- WINGONEER **SOIC8** SOP8 Test Clip Für EEPROM 93CXX / 25CXX / 24Cxx

Um die Software zu verwenden wird ein Linux basiertes Betriebssystem wie *z.B Ubuntu* benötigt.
Es wird empfohlen das Ganze auf einem 2ten Laptop durchzuführen, falls auf eurem Gerät Windows installiert ist gibt es folgende alternativen:
- Eine Ubuntu-VM mit VirtualBox erstellen und den CH341A Writer durchreichen
- Ein Live Linux verwenden
- Einen RaspberryPi kaufen

## Die Ausführung

Die Ausführung ist in 3 Schritte unterteilt:
1. Die Laptop Firmware aktualisieren
2. Die benötigte Software installieren
3. Die modifizierte Firmware erstellen und den EEPROM beschreiben

### Schritt 1

Zuerst muss der Laptop auf einen aktuellen Stand gebracht werden:

ME Firmware Dateien herunterladen, zu finden auf der offiziellen Lenovo Seite:
https://pcsupport.lenovo.com/de/de/downloads/ds112240


In diesem Paket befindet sich auch das `MEInfoWin Tool` um den aktuellen Status der Management Engine anzuzeigen.

Nach dem herunterladen die Anwendung auf dem Laptop starten und **das Update durchführen**.

Mit diesem Befehl kann der aktuelle Status der Management Engine angezeigt werden:
```
cmd /k "C:\DRIVERS\WIN\ME\MEInfoWin.exe -verbose"
```
Beispiel Ausgabe:
```
Intel(R) MEInfo Version: 11.6.29.3287
Copyright(C) 2005 - 2017, Intel Corporation. All rights reserved.




Windows OS Version : 7.0

...

  CurrentState:                               Normal
  ManufacturingMode:                          Disabled
  FlashPartition:                             Valid
  OperationalState:                           CM0 with UMA
  InitComplete:                               Complete
  BUPLoadState:                               Success
  ErrorCode:                                  No Error
  ModeOfOperation:                            Normal
  SPI Flash Log:                              Not Present
  Phase:                                      ROM/Preboot
  ICC:                                        Valid OEM data, ICC programmed
  ME File System Corrupted:                   No
  PhaseStatus:                                AFTER_SRAM_INIT
  FPF and ME Config Status:                   Match
```

Wie zu erwarten wird unter `CurrentState` der Zustand `Normal` ausgegeben.

Nun in das BIOS booten und den **Service Modus aktivieren**, dieser deaktiviert temporär die Interne Batterie. Beim nächsten hochfahren wird dieser wieder deaktiviert.
```
F1 -> Config -> Power -> Disable Built-in Battery
```

Den Laptop aufschrauben und den EEPROM suchen.

![ThinkPad BIOS](/Images/thinkpad-01.jpg)
Unter https://www.lenovoservicetraining.com gibt es Anleitungen und Videos zu dem jeweiligen Model.

### Schritt 2
Ist der EEPROM gefunden kann es losgehen:
Den CH341A Writer mit dem Laptop / Desktop PC verbinden und flashrom installieren:
```
sudo apt-get install flashrom
```

Ein Ordner erstellen mit dem wir arbeiten werden
```
# Arbeitsverzeichnis erstellen und in dieses wechseln
mkdir ThinkPad-T560 && cd ThinkPad-T560
```

Die me_cleaner Repository herunterladen
```
git clone https://github.com/corna/me_cleaner
```

Python ist in einer Ubuntu-Standardinstallation **bereits enthalten**, falls nicht kann es mit folgendem Befehl installiert werden:
```
sudo apt-get install python
sudo apt-get install python-pip
```

### Schritt 3

Den SOIC8-Clip an den EEPROM klemmen.

![SOIC8 Clip](/Images/thinkpad-02.jpg)
![ThinkPad Overview](/Images/thinkpad-03.jpg)

Mit dem Befehl `flashrom -p ch341a_spi` kann geprüft werden ob der Clip richtig sitzt und der EEPROM erkannt wird.
```
sudo flashrom -p ch341a_spi
flashrom v0.9.9-rc1-r1942 on Linux 4.10.0-37-generic (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Winbond flash chip "W25Q128.V" (16384 kB, SPI) on ch341a_spi.
No operations were specified.
```

Den EEPROM auslesen und die Original Firmeware in das Arbeitsverzeichnis exportieren:
```
sudo flashrom -p ch341a_spi -r factory_ME_11.6_Consumer_T560.bin
flashrom v0.9.9-rc1-r1942 on Linux 4.10.0-37-generic (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Winbond flash chip "W25Q128.V" (16384 kB, SPI) on ch341a_spi.
Reading flash... done.
```

Das HAP-Bit setzen, hier gibt es 2 verschiedene Möglichkeiten:
> me_cleaner sets this HAP/AltMeDisable bit when the -s (enable only the kill-switch, but don't remove the extra code from the firmware) or the -S (enable the kill-switch and remove the extra code from the firmware) are passed.

Wir werden **nur den kill-switch aktivieren** und die bearbeitete Firmware nach `patched_ME_11.6_Consumer_T560.bin` speichern.
```
sudo me_cleaner/me_cleaner.py -s factory_ME_11.6_Consumer_T560.bin -O patched_ME_11.6_Consumer_T560.bin
Full image detected
The ME/TXE region goes from 0x3000 to 0x700000
Found FPT header at 0x3010
Found 13 partition(s)
Found FTPR header: FTPR partition spans from 0x4000 to 0x133000
Found FTPR manifest at 0x4448
ME/TXE firmware version 11.6.29.3287
Setting the HAP bit in PCHSTRP0 to disable Intel ME...
Checking the FTPR RSA signature... VALID
Done! Good luck!
```

Die veränderte Firmware aufspielen:
```
sudo flashrom -p ch341a_spi -w patched_ME_11.6_Consumer_T560.bin
flashrom v0.9.9-rc1-r1942 on Linux 4.10.0-37-generic (x86_64)
flashrom is free software, get the source code at https://flashrom.org

Calibrating delay loop... OK.
Found Winbond flash chip "W25Q128.V" (16384 kB, SPI) on ch341a_spi.
Reading old flash chip contents... done.
Erasing and writing flash chip... Erase/write done.
Verifying flash... VERIFIED.
```

Zum Schluss die Rechte der beiden Firmewaredateien anpassen, hierzu wird `chown` verwendet:
```
# Beispiel
chown -R USERNAME:GROUPNAME /PATH/TO/FILE

# Die Benutzergruppe users sollte auf den meisten Linux Distributionen vorhanden sein
chown -R $USER:users factory_ME_11.6_Consumer_T560.bin
chown -R $USER:users patched_ME_11.6_Consumer_T560.bin
```

Beim ersten starten kann eine Fehlermeldung auftreten das die BIOS Einstellungen nicht geladen werden können.

Um zu prüfen ob ME deaktiviert ist ins BIOS booten:
```
F1 -> Config -> Intel(R) AMT
```
Der Menüpunkt `Intel (R) AMT Control` sollte jetzt mit `[Permanently Disabled]` angezeigt werden und **nicht** mehr auswählbar sein.


Mit `MEInfoWin` kann zusätzlich überprüft werden ob die Management Engine deaktiviert ist:
```
cmd /k "C:\DRIVERS\WIN\ME\MEInfoWin.exe -verbose"
```

Die Ausgabe sollte nun wie folgt aussehen:
```
  CurrentState:                               Disabled
  ManufacturingMode:                          Disabled
  FlashPartition:                             Valid
  OperationalState:                           Transitioning
  InitComplete:                               Initializing
  BUPLoadState:                               Success
  ErrorCode:                                  Disabled
  ModeOfOperation:                            Alt Disable Mode
  SPI Flash Log:                              Not Present
  Phase:                                      BringUp
  ICC:                                        Valid OEM data, ICC programmed
  ME File System Corrupted:                   No
  PhaseStatus:                                UNKNOWN
  FPF and ME Config Status:                   Match
```

**Glückwunsch!**


## Quellen
- [HAP-Bit](http://blog.ptsecurity.com/2017/08/disabling-intel-me.html)
- [me_cleaner](https://github.com/corna/me_cleaner)
- [Intel ME 11.x Firmware Images Unpacker](https://github.com/ptresearch/unME11)
- [Intel Banner](https://www.bleepingcomputer.com/news/hardware/intel-fixes-9-year-old-cpu-flaw-that-allows-remote-code-execution)

## Autor

* **Patrick Becker** - [GitHub](https://github.com/patbec)

E-Mail: [github@bec-wolke.de](mailto:github@bec-wolke.de)

## tl;dr
```
# flashrom installieren
sudo apt-get install flashrom

# Arbeitsverzeichnis erstellen
mkdir ThinkPad-T560 && cd ThinkPad-T560

# me_cleaner herunterladen
git clone https://github.com/corna/me_cleaner

# SOIC8-Clip auf dem EEPROM anklemmen und prüfen ob flashrom den EEPROM erkennt
flashrom -p ch341a_spi

# EEPROM auslesen und die Firmware sichern
sudo flashrom -p ch341a_spi -r factory_ME_11.6_Consumer_T560.bin

# HAP-Bit setzten
sudo me_cleaner/me_cleaner.py -s factory_ME_11.6_Consumer_T560.bin -O patched_ME_11.6_Consumer_T560.bin

# Die veränderte Firmware aufspielen
sudo flashrom -p ch341a_spi -w patched_ME_11.6_Consumer_T560.bin

# Rechte anpassen
chown -R $USER:users factory_ME_11.6_Consumer_T560.bin
chown -R $USER:users patched_ME_11.6_Consumer_T560.bin
```

Prüfen ob ME deaktiviert ist, ins BIOS booten:
```
F1 -> Config -> Intel(R) AMT
```
`AMT Control` sollte jetzt mit `[Permanently Disabled]` angezeigt werden und **nicht** mehr auswählbar sein.

**Glückwunsch!**
