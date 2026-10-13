# TP4056 KiCad assets

## chip/  — the TP4056 IC
- tp4056.kicad_sym             KiCad 6+ schematic symbol
- tp4056.lib / .dcm            legacy KiCad 5 symbol (same part)
- TP4056_SOP-8-PP.kicad_mod    SOP-8 footprint w/ exposed pad
Source: https://github.com/alltheworld/tp4056

## module/  — the pre-built TP4056 charger PCB (drop-in)
- TP4056-18650.kicad_mod       footprint of the whole 18650 charger module
Source: https://github.com/ccadic/TP4056-18650

If you want a generic SOP-8 instead, KiCad's built-in library has it:
  Package_SO:SOP-8_3.9x4.9mm_P1.27mm
  Package_SO:SOP-8-1EP_4.57x4.57mm_P1.27mm_EP4.57x4.45mm
