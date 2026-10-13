# CA-L01  —  CrossAir LTE/4G/3G/2G SMD chip antenna

## What it is
Small surface-mount ceramic patch antenna, 23.0 x 3.5 x 1.6 mm.
LCSC C2838841 / JLCPCB Assembly ANT_CA_L01.
Bands: GSM/CDMA/WCDMA/LTE/NB-IoT.

Datasheet (Global Antenna): https://www.global-antenna.com/download/download-33-696.html
Product page:               https://www.global-antenna.com/product/product-89-635.html

## Footprint
No public KiCad library found. Footprint is a simple 2-pad SMD outline
~ 23 x 3.5 mm. You can:
1. Build the footprint from datasheet dimensions (easy — single feed pad
   + ground pad), OR
2. Convert from EasyEDA: this part is in LCSC's catalogue (C2838841), so
   `easyeda2kicad` will pull it directly:
       pip install easyeda2kicad
       easyeda2kicad --full --lcsc_id=C2838841 --output=./CA-L01

Option 2 is the fastest reliable path for any LCSC-listed part.
