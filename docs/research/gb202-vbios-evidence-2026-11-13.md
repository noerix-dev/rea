# BL-HW-5090-FW-01 — Evidenzbericht 1: GB202-VBIOS und Speicherinitialisierung

**Datum:** 2026-11-13 · **Zone:** 1 · **Befundstatus:** Source-Code-Audit, **kein GPU-ROM- oder Hardwaretest**.
**Fragestellung:** Können Firmware und Treiber der GeForce RTX 5090 die physische VRAM-Konfiguration auf eine anders bestückte GB202-Karte abstimmen — und was erlaubt der dokumentierte Override wirklich?

## 1. Kurzergebnis

**[V]** Die öffentlich verfügbare VBIOS-Parserimplementierung unterscheidet eine **Board-Strap-Auswahl**, eine **Speicher-Konfigurationstabelle**, Speicher-Taktbänder/Timings und ein strukturell erkennbares, aber nicht entschlüsseltes GB20x-Objekt `0x89`. Die Speicher-Konfigurationstabelle nennt Type-Code, Hersteller-Code, Strap und Column, **keine verifizierte Dichte-/Kapazitätsdefinition**.

**[V]** Windows und Linux haben dokumentierte VBIOS-Override-Pfade, mit denen ein *vorhandenes VBIOS-Image* ohne Flash auf der GPU verwendet werden kann. Das ist weder eine nachgewiesene Speicherumschaltung noch ein Nachweis, dass das `0x89`-Profil über diesen Weg wirksam wird.

**[A]** Ein firmwareseitiger Support für eine physisch anders bestückte GB202-Karte erscheint denkbar. **[V]** Mehr VRAM ohne zusätzliche physische GDDR7-Speicherkapazität kann Firmware nicht bereitstellen. **[U]** Ob eine technisch umbestückte 5090 über einen reinen VBIOS-/GSP-Pfad auf 48/64/96 GB initialisiert werden könnte, bleibt ohne Original-Images, Laufzeitevidenz und unabhängige Validierung ungeklärt.

## 2. Echte Quellcode-Fundstellen

| ID | Erkenntnis | Nachweis | Bewertung |
|---|---|---|---|
| E01 | `BIT M+0x03` (v0x10) führt pro Entry Type (low nibble), Strap (high nibble), Column und Hersteller-Code; Board-Strap wählt Entry. | [memory.py L1–9](https://github.com/Panchovix/nvidia-vbios-vix-editor/blob/3ca26ae74b76ed62a9de8a9a5a33409e514599f8/src/nvidia_vbios/rom/memory.py#L1-L9), [L94–114](https://github.com/Panchovix/nvidia-vbios-vix-editor/blob/3ca26ae74b76ed62a9de8a9a5a33409e514599f8/src/nvidia_vbios/rom/memory.py#L94-L114) | [V] Parsercode, Semantik des Board-Straps durch Autor |
| E02 | `BIT P+0x04` (v0x11) enthält Speicher-Taktbänder und Indizes auf Timings; `BIT P+0x08` (v0x20) enthält Timing-Register-Abbilder. tRC/tRAS/tRP abgeleitet, tRFC wahrscheinlich, CL/tWR tentativ. | [memory.py L9–17](https://github.com/Panchovix/nvidia-vbios-vix-editor/blob/3ca26ae74b76ed62a9de8a9a5a33409e514599f8/src/nvidia_vbios/rom/memory.py#L9-L17) | [V] Dekoder, einzelne Felddeutungen nicht offiziell bestätigt |
| E03 | GB20x-BIT-M referenziert zusätzliche Objekte `0x05`, `0x09`, `0x19`, `0x1D`, `0x21`, `0x25`. Die Implementierung **hasht** diese und entschlüsselt ihre Inhalte nicht. | [memory.py L40–43](https://github.com/Panchovix/nvidia-vbios-vix-editor/blob/3ca26ae74b76ed62a9de8a9a5a33409e514599f8/src/nvidia_vbios/rom/memory.py#L40-L43), [L232–245](https://github.com/Panchovix/nvidia-vbios-vix-editor/blob/3ca26ae74b76ed62a9de8a9a5a33409e514599f8/src/nvidia_vbios/rom/memory.py#L232-L245) | [V] |
| E04 | GB20x-Objekt `0x89`: zwei Magic-Header mit Abstand `0x8B0`, 0x3A-Byte-Header, zwei 0x716-Byte-Records mit gepaarten Spalten `B1/0`, `B1/1`, `B2/0`, `B2/1`; IDs/Revisionen und Feldbedeutung nicht nachgewiesen. | [memory.py L43–62](https://github.com/Panchovix/nvidia-vbios-vix-editor/blob/3ca26ae74b76ed62a9de8a9a5a33409e514599f8/src/nvidia_vbios/rom/memory.py#L43-L62), [L178–219](https://github.com/Panchovix/nvidia-vbios-vix-editor/blob/3ca26ae74b76ed62a9de8a9a5a33409e514599f8/src/nvidia_vbios/rom/memory.py#L178-L219) | [V] Struktur, [A] Funktion als Memory-Training |
| E05 | Parser liefert `board_profile`, `tables` und Vergleichs-Hashes, schreibt jedoch nichts; Änderungskontrolle und kryptografische Signaturen sind **nicht** Teil des Memory-Parsers. Der Autor sagt ausdrücklich, `0x89` wirke nur bei vollständigem Flash, nicht über seinen Override. | [memory.py L247–284](https://github.com/Panchovix/nvidia-vbios-vix-editor/blob/3ca26ae74b76ed62a9de8a9a5a33409e514599f8/src/nvidia_vbios/rom/memory.py#L247-L284), [editor-api.md Memory](https://github.com/Panchovix/nvidia-vbios-vix-editor/blob/3ca26ae74b76ed62a9de8a9a5a33409e514599f8/docs/editor-api.md#L297-L327) | [V] dokumentierte Einschränkung, Boot-Wirkung ohne eigene Reproduktion |
| E06 | Windows-Loader schreibt VBIOS als `vbios` `REG_BINARY` in GPU-Treiber-Key; Versionen ab 0.4 ändern `EnableGpuFirmware` normalerweise nicht. Alte v0.3.x setzten `0x11` für GSP-Modus, Default `0x12`. | [operations.rs L1–30](https://github.com/Panchovix/nvidia-windows-vbios-vix-loader/blob/6f944f561bdc585593667eb1146e1393ca579580/src/operations.rs#L1-L30), [README L29–46](https://github.com/Panchovix/nvidia-windows-vbios-vix-loader/blob/6f944f561bdc585593667eb1146e1393ca579580/README.md#L29-L46) | [V] |
| E07 | `NvAPI_GPU_GetVbiosImage` liest das **laufende** BIOS-Image zur Überprüfung des Overrides zurück, mit `0xFA000` Bytes Bufferlimit; bestätigt nur geladene Bytes, **keine** VRAM-Topologie. | [nvapi.rs L1–34](https://github.com/Panchovix/nvidia-windows-vbios-vix-loader/blob/6f944f561bdc585593667eb1146e1393ca579580/src/nvapi.rs#L1-L34), [README Verify](https://github.com/Panchovix/nvidia-windows-vbios-vix-loader/blob/6f944f561bdc585593667eb1146e1393ca579580/README.md#L212-L244) | [V] |
| E08 | Linux-Loader patcht offene NVIDIA-Kernelmodule, legt einen VBIOS-Descriptor für die GSP-Initialisierung an und prüft PCI-Device-ID/Legacy-Checksum. Der eigene Logtext sagt: Host-Staging ist **kein Nachweis**, dass Firmware das Image akzeptiert. | [_payload.py L160–235](https://github.com/Panchovix/nvidia-linux-vbios-vix-loader/blob/8a6b9e3b21a96f86684dd0131d5bec73cd9adfd4/src/nvidia_vbios/driver/_payload.py#L160-L235) | [V] Source-Code-Befund |
| E09 | Vix Editor verifiziert bzw. erneuert keine kryptografischen Signaturen; bei geänderten Images bleibt die ursprüngliche Signatur erhalten und passt nicht mehr. Der Loaderpfad ist nicht mit signierter Flash-Firmware gleichzusetzen. | [editor-api.md L37–46](https://github.com/Panchovix/nvidia-vbios-vix-editor/blob/3ca26ae74b76ed62a9de8a9a5a33409e514599f8/docs/editor-api.md#L37-L46), [README](https://github.com/Panchovix/nvidia-vbios-vix-editor/blob/3ca26ae74b76ed62a9de8a9a5a33409e514599f8/README.md#read-before-using-an-edited-image) | [V] |

Quellstände bei Recherche: **Editor** `3ca26ae74b76ed62a9de8a9a5a33409e514599f8`; **Windows-Loader** `6f944f561bdc585593667eb1146e1393ca579580` (v0.5.0-Linie); **Linux-Loader** `8a6b9e3b21a96f86684dd0131d5bec73cd9adfd4`. Die verlinkten Commits sichern das untersuchte Quellmaterial.

## 3. Firmware-Verantwortungsgrenzen (Analyse)

| Ebene | Bekannter Beitrag | Noch unbelegt |
|---|---|---|
| Physische Speicherchips/Leiterplatte | Existenz, Dichte und Anzahl der GDDR7-ICs, elektrisches Clamshell-Design | Ob eine normale ASUS-5090-Platine andere Konfigurationen elektrisch abbildet |
| Board-Straps | Auswahl eines Memory-Config-Eintrags laut Parser | Ob andere Quellen (Fuse/GSP) die effektive Auswahl überschreiben |
| VBIOS BIT-M/P-Tabellen | Memory-Type/Hersteller/Column, Taktbänder, Controller-Timings | Speicher-Dichte-/Kapazitätsfelder oder dynamisch wirksame Editierbarkeit |
| GB20x Objekt 0x89 und andere BIT-M-Objekte | Rohstruktur/Hashes, wahrscheinlich Training | Präzise Feldsemantik, trainierter Kanal-/Rank-Modus, Authentifizierung |
| NVIDIA Treiber/GSP | Erhält via dokumentiertem Overridepfad ein anderes VBIOS-Abbild (Vix), Host-Staging vs. Firmware-Annahme getrennt | Welche Daten er bei Boot/Memory-Discovery aus ROM, Strap oder Fuse verwendet |
| Laufzeit-API | Rücklesen des tatsächlich verwendeten VBIOS-Abbilds | Wird damit mehr *physisch vorhandener* Speicher initialisiert? Nein, der Rücklesecheck belegt das nicht |

## 4. 48 / 64 / 96 GB differenziert

- **48 GB:** denkbar mit 16×3-GB- statt 16×2-GB-Chips, falls elektrische, thermische und initialisierungsseitige Anforderungen erfüllt wären; Firmware-Kompatibilität nicht belegt.
- **64 GB:** benötigt gegenüber 32 GB andere physische Kapazität/Bestückung; bei 32×2-GB-Clamshell-Annahme muss das Board zusätzliche Chips unterstützen. Firmware-spezifischer Modus und Training unbekannt.
- **96 GB:** benötigt andere physische Kapazität/Bestückung, beispielsweise 32×3 GB; Vorhandensein einer professionellen 96-GB-GB202-GPU beweist nicht, dass ein GeForce-Board diesen Modus übernehmen kann.
- **Nur Firmware auf unveränderter RTX 5090:** keine echte Kapazitätserhöhung von 32 GB; ein geänderter Treiberwert oder vorgetäuschte Anzeige wäre kein nachweisbarer zusätzlicher Speicher.

## 5. Was REA derzeit leisten kann und was nicht

Das geprüfte `noerix-dev/rea/AGENTS.md` beschreibt REA als generisches lokales Reverse-Engineering-Tool (Hopper/Ghidra/IDA-Adapter; Schwerpunkt native Binäranalyse). **Nicht belegt** ist ein natives GB202-VBIOS-/GSP-Objektformat-Backend. Daher:

- [E] **Vix Editor** als spezialisierter Offline-Dekoder für legitime Original-ROM-Dateien.
- [E] **REA** für quellgebundene Investigation, Evidenzgraph und gegebenenfalls über unterstützte externe Werkzeuge analysierbare Binärabschnitte; keine Behauptung, dass REA jetzt schon GSP-Firmware dekompiliert.
- [E] **Noelab** als Persistenz des Forschungsstandes, Vermutungen und negativer Ergebnisse; kein Schreiben nach amarux ohne Judge/Writer.
- [R] Nicht geprüfte ROM-Downloads/Loader nicht auf dem produktiven Rechner ausführen.

## 6. Folgeuntersuchung (nur lesend)

1. Mit legitimer Originaldatei der **genauen ASUS-5090-Variante** (GPU-Z `Save to file`) und berechtigt verfügbarem Profi-GB202-ROM die `nvbios memory --raw --compare`-Ausgaben gewinnen; beide Originale bleiben unverändert und werden **nicht** ins Repo committet.
2. Firmware-Identitäten: GPU-/Board-/PCI-ID, Version, SHA-256, Parser-Version; Differenzen der Speichertabellen getrennt von `0x89`-Differenzen ausweisen. **Ein Unterschied beweist keine Kausalität.**
3. Als nächstes Memory-Objekte `0x05`/`0x09`/`0x19`/`0x1D`/`0x21`/`0x25` quellenbasiert untersuchen; gegen von NVIDIA öffentlich bereitgestellten Open-Kernel-Code auf Memory-Discovery-Bezüge prüfen.
4. Entscheidend: Nachweis, dass ein bestimmtes Feld/Strap/GSP-Verhalten tatsächlich die **physisch vorhandene** Speichergröße beeinflusst, fehlt. Ohne solchen Nachweis bleibt `firmware_only_unlock=UNPROVEN`.

## 7. Teststatus und Einschränkungen

**Getan:** Upstream-Sourcecode und README/Editor-API direkt über GitHub gelesen, Semantik mit Implementierung abgeglichen, Commit-Permalinks festgehalten.  
**Nicht getan:** keine Original-ROMs gelesen; keine GPU ausgelesen; keine Firmware-Binäranalyse; keine Tests lokal ausgeführt. Container hatte keinen Netzwerk-DNS-Zugriff auf github.com; lokaler REA-MCP-Provider dieser Sitzung nicht verbunden. Keine Behauptung eines 96-GB-Hacks.

**Kurzurteil:** `Firmware-only: Nein` für eine unverändert bestückte 32-GB-5090 (kein zusätzlicher physischer RAM). `Firmware-Unterstützung für anders bestückte Karten: Unbekannt`; die quellseitigen ROM- und GSP-Schnittstellen sind reale Untersuchungsansätze.
