RAIDEN S0-1 — ESP32-S3FN8 DEVKIT — Rev A — 2026-07-14
Raiden Technology
========================================================

ICERIK / CONTENTS
  Gerber\                        Gerber (11 katman) + Excellon delik (.drl) + delik haritasi
  Raiden S0-1 BOM.csv            Malzeme listesi (MPN'li, Value+MPN gruplu)
  Raiden S0-1 PickAndPlace.csv   Dizgi koordinat dosyasi (mm, ust+alt)
  Raiden S0-1 Schematic.pdf      Sema

URETIM NOTLARI / FABRICATION NOTES
  1. 4 katman / 4-layer, 1.6 mm FR-4.
     Stackup: F.Cu (sinyal) / In1.Cu = GND duzlemi / In2.Cu = +3.3V duzlemi / B.Cu (sinyal)
     JLC7628 veya esdegeri standart stackup uygundur.
  2. Soldermask: SIYAH / BLACK. Silkscreen: BEYAZ / WHITE (her iki yuz).
  3. KONTROLLU EMPEDANS / CONTROLLED IMPEDANCE:
     RF hatti LNA_IN -> ANT1 (F.Cu, 0.2 mm, kart ust bolgesi x~112) 50 ohm single-ended.
     Fab stackup'ina gore iz genisligi fab tarafindan ayarlanabilir / please adjust
     trace width to 50 ohm per your stackup, or advise required width.
  4. VIA-IN-PAD: U2 (SOT-223 regulator) tab pad'i altinda via (111.15, 138.75).
     Lutfen kapatin / plugged (IPC-4761 Type VI) veya en az tented.
  5. H1-H4: 4x M2 montaj deligi, NPTH (kaplamasiz) / non-plated.
  6. Min iz/bosluk / min track/clearance: 0.2 mm / 0.2 mm. Min delik / min drill: 0.3 mm.

DIZGI NOTLARI / ASSEMBLY NOTES
  - U1 = QFN-56 0.4 mm pitch, EP 4x4 mm — stencil ve refill profili onemli.
  - Y2 40 MHz kristal MUTLAKA CL = 8 pF olmali (C9/C10 = 10 pF buna gore secildi).
    Onerilen / suggested: ECS-400-8-30B-CKM.
  - Y1 32.768 kHz kristal CL = 12.5 pF (Epson FC-135; C11/C12 = 22 pF).
  - BOM'daki MPN'ler oneridir; stok/fiyat siparis oncesi dogrulanmali.

DOGRULAMA / VERIFICATION (KiCad 9.0.7, kicad-cli)
  ERC: 0 hata. DRC: 0 hata, 0 kopuk baglanti.
  (5 kalici uyari = kasitli yerel footprint kopyalari: ANT1 RF keepout + H1-H4.)
