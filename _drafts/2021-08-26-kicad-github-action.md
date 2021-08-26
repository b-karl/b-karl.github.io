---
layout: single
author_profile: false
title:  "Automatically generate gerbers from KiCad using GitHub actions"
date:   2021-05-20 14:00:00 +0100
categories: projects mechanical-keyboard pcb kicad github
excerpt: ""
---

To generate gerbers, the production files used to order PCBs, can be a bit tedious in KiCad. For my mechanical keyboard project I have three different PCBs and it doesn't feel worth generating and zipping the files every time you do some change. Thankfully, there is an automation tool for KiCad, called [KiBot](https://github.com/INTI-CMNB/kibot), and a [GitHub Action wrapper called kicad-export](https://github.com/marketplace/actions/kicad-exports) for it so that you can easily create automated workflows that will do all that work for you, straight on GitHub with every push of code!

The kicad-export documentation has a good example, but if you are unfamiliar with GitHub Actions (like myself), there are a few tweaks that might be worth doing. You can [find my current workflow file here.](https://github.com/b-karl/KBIC65/blob/main/.github/workflows/generate_gerbers.yml)

```yaml
name: Generate Gerber files

on:
  push:
    paths:
    - '**.sch'
    - '**.kicad_pcb'
    - '.github/**.yml'
    - '**.kibot.yml'
  pull_request:
    paths:
    - '**.sch'
    - '**.kicad_pcb'
    - '.github/**.yml'
    - '**.kibot.yml'
  workflow_dispatch:
    inputs:
      name:
        description: 'Workflow run name'
        required: false
        default: 'Manual test run'
  
jobs:
  bottom_plate:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: nerdyscout/kicad-exports@v2.3.1
      with:
        config: kicad/jlcpcb_gerbers.kibot.yml
        dir: bottom_plate
        schema: 'kicad/bottom/*.sch'
        board: 'kicad/bottom/*.kicad_pcb'
    - name: upload results
      uses: actions/upload-artifact@v2
      with:
        name: bottom_plate
        path: bottom_plate

  top_plate:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: nerdyscout/kicad-exports@v2.3.1
      with:
        config: kicad/jlcpcb_gerbers.kibot.yml
        dir: top_plate
        schema: 'kicad/plate/*.sch'
        board: 'kicad/plate/*.kicad_pcb'
    - name: upload results
      uses: actions/upload-artifact@v2
      with:
        name: top_plate
        path: top_plate
        
  pcb:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: nerdyscout/kicad-exports@v2.3.1
      with:
        config: kicad/jlcpcb_gerbers.kibot.yml
        dir: pcb
        schema: 'kicad/pcb/*.sch'
        board: 'kicad/pcb/*.kicad_pcb'
    - name: upload results
      uses: actions/upload-artifact@v2
      with:
        name: pcb
        path: pcb

```

