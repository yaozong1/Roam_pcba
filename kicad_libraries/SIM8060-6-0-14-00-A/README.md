# SIM8060-6-0-14-00-A  —  GCT hinged Nano-SIM connector, 6 contacts

## What it is
Global Connector Technology hinged Nano-SIM card holder, 6 positions,
SMT, gold-plated, right angle.
Datasheet/product page: https://gct.co/connector/sim8060

## Footprint
Custom — no stock KiCad equivalent. GCT-style SIM holders are not
in `Connector_Card.pretty`. Use a vendor download:

- SnapEDA: https://www.snapeda.com/parts/SIM8060-6-0-14-00-A/Global%20Connector%20Technology/view-part/
- Component Search Engine (Samacsys, embedded on the GCT product page):
  https://componentsearchengine.com/part-view/SIM8060-6-1-14-00-A/GCT
  (the -6-0- and -6-1- variants share the same footprint — only the
  trim/hinge orientation differs)

Both offer free KiCad export. Drop the unpacked .pretty/.kicad_sym
into this folder and add them to your library tables.
