# I2C per G-Code in FluidNC — Analyse und Implementierung

Kurzbeschreibung
- Ziel: I2C-Geräte per G-Code abfragen und steuern (Scan, Probe, Read, Write).
- Ergebnis: M260/M261 als generische I2C-Schnittstelle; auf Hardware verifiziert.

Wichtige Fundstellen im Codebasis
- I2C-Basis / Treiber
  - FluidNC/src/Machine/I2CBus.h / I2CBus.cpp — Abstraktion für einen I2C-Bus (init, read, write, Scan-Option beim init).
  - include/Driver/fluidnc_i2c.h — low-level Schnittstelle (i2c_master_init, i2c_write, i2c_read).
  - X86TestSupport / Wire.* — Mock/Emulation für Tests (Testinfrastruktur bereits vorhanden).

- Konfiguration
  - FluidNC/src/Machine/MachineConfig.h — enthält config->_i2c[] und wie Bus-Instanzen im System verfügbar sind.
  - example_configs/* — Beispielkonfigurationen zeigen, wie i2c/i2c_extender eingebunden werden kann.

- Extender / Peripherie
  - FluidNC/src/Extenders/I2CExtender.* — vorhandene Extender-Implementierung (PCA95xx etc.).

- G-Code Parser / Ausführung
  - FluidNC/src/GCode.h — G-Code Typen, enum NonModal, ModalGroups, parser-datenstrukturen.
  - FluidNC/src/GCode.cpp — Parser (Wort-Parsing, M-Code-Erkennung) und Ausführungslogik. Hier müssen neue M-Codes aufgenommen und ausgeführt werden.

Finale Implementierung (auf Hardware verifiziert)
-------------------------------------------------
Hinweis: Ein zunächst angedachter Marlin-Ansatz mit A (Adresse) / B (Byte) bzw. O (Bus)
wurde **verworfen**. Gründe, die sich erst bei der Umsetzung am Gerät zeigten:
- A/B/C sind in FluidNC Rotationsachsen-Wörter. Auf einer Maschine ohne diese Achsen
  lehnt der Parser sie mit `Error::GcodeUnsupportedCommand` (error:20) ab, *bevor*
  der M-Code ausgeführt wird (siehe GCode.cpp, `case 'A'`: Bedingung `n_axis > A_AXIS`).
- O wird vom Flow-Control-Subsystem abgefangen ("Flow Control Syntax Error") und ist
  daher als Parameter unbrauchbar.

Daher nutzt die finale Version ausschließlich **value words**, die der Parser
unabhängig von der Achszahl akzeptiert.

Parameter
- `P` = I2C-Adresse (7-bit, dezimal)
- `Q` = erstes Datenbyte (Write)
- `R` = zweites Datenbyte (Write, optional)
- `L` = Anzahl zu lesender Bytes (Read)

Befehle
- `M260`                     → Bus-Scan (Adressen 1..126), listet antwortende Geräte
- `M260 P<addr>`             → Probe einer Adresse (ACK/NACK)
- `M260 P<addr> Q<byte>`     → 1 Byte schreiben
- `M260 P<addr> Q<b> R<b2>`  → 2 Bytes schreiben (z.B. Register Q, Wert R)
- `M261 P<addr> L<count>`    → count Bytes lesen

Code-Stellen
- GCode.h: NonModal-Einträge `I2C_Write = 2000` (M260), `I2C_Read = 2001` (M261).
  (Das frühere `I2C_Scan` entfällt; Scan/Probe/Write laufen alle über `I2C_Write`.)
- GCode.cpp Parser: `case 260` → I2C_Write, `case 261` → I2C_Read, ModalGroup MM10.
- GCode.cpp Validierung: P/Q/R/L werden für I2C-Kommandos konsumiert; die optionalen
  Float-Wörter P/Q/R werden bei Abwesenheit auf NAN gesetzt, damit die Ausführung
  "nicht angegeben" von "als 0 angegeben" unterscheiden kann.
- GCode.cpp Ausführung: Zugriff auf `config->_i2c[0]` über `I2CBus::read()/write()`.
  Scan und Probe nutzen einen 1-Byte-Read statt eines 0-Byte-Writes, da
  `i2c_master_write_to_device` unter ESP-IDF 4.x bei 0-Byte-Writes unzuverlässig ist.

I2CRESP-Format (über log_info, maschinenlesbar)
- Scan:  "I2CRESP B0 SCAN: 0x18 0x3C ..."
- Probe: "I2CRESP B0 A0x18 ACK"  bzw.  "... NACK"
- Write: "I2CRESP B0 A0x18 WROTE 2"
- Read:  "I2CRESP B0 A0x18: 33 00 F0 ..."

Konfiguration (config.yaml)
- Der Section-Key ist nummeriert (`sections("i2c", 0, MAX_N_I2C, ...)`), also `i2c0:`
  (nicht `i2c:` — das wird ignoriert).
  ```yaml
  i2c0:
    sda_pin: gpio.21
    scl_pin: gpio.22
    frequency: 100000
  ```

Hardware-Test (LIS3DH, Adresse 0x18)
- `M260`            → I2CRESP B0 SCAN: 0x18
- `M260 P24 Q15`    → WROTE 1 (Registerzeiger auf WHO_AM_I 0x0F)
- `M261 P24 L1`     → 33 (WHO_AM_I bestätigt LIS3DH/LIS2DH12)
- `M260 P24 Q32 R87`→ WROTE 2 (CTRL_REG1 0x20 = 0x57: 100 Hz, alle Achsen)
- `M260 P24 Q168`   → WROTE 1 (Zeiger 0xA8 = OUT_X_L + Auto-Increment)
- `M261 P24 L6`     → C0 01 40 F8 80 3F (Z ≈ 1 g → Schwerkraft, Sensor liegt flach)

Synchronität / Sicherheit
- Die Implementierung führt blockierende I2C-Transaktionen durch. Keine M260/M261
  während zeitkritischer GCode-Operationen senden. Die Firmware warnt, wenn der Bus
  nicht konfiguriert ist oder eine Transaktion fehlschlägt.

Offene Punkte / Grenzen
- Write ist auf 1–2 Bytes (Q + R) begrenzt; längere Payloads erfordern mehrere M260
  (Auto-Increment des Geräts vorausgesetzt). Marlin puffert hier N Bytes via S1.
- Nur Bus 0 (`i2c0`) wird angesprochen; mehrere Busse sind nicht implementiert.
- Unit-Tests (Wire-Mock vorhanden) noch nicht ergänzt.

