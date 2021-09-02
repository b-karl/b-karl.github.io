---
layout: single
author_profile: false
title:  "Automatically generate gerbers from KiCad using GitHub actions and KiBot"
date:   2021-05-20 14:00:00 +0100
categories: guides kicad
excerpt: ""
---

To generate gerbers, the production files used to order PCBs, can be a bit tedious in KiCad. For my mechanical keyboard project I have three different PCBs and it doesn't feel worth generating and zipping the files every time you do some change. Thankfully, there is an automation tool for KiCad, called [KiBot](https://github.com/INTI-CMNB/kibot), and a [GitHub Action wrapper called kicad-export](https://github.com/marketplace/actions/kicad-exports) for it so that you can easily create automated workflows that will do all that work for you, straight on GitHub with every push of code!

The kicad-export documentation has a good example, but if you are unfamiliar with GitHub Actions (like myself), there are a few tweaks that might be worth doing. You can [find my current workflow file here.](https://github.com/b-karl/KBIC65/blob/main/.github/workflows/generate_gerbers.yml)

## Create a KiBot configuration

To run kicad-export we are going to need a KiBot configuration file that tells KiBot what to do. I use JLCPCB to manufacture my PCBs and the KiBot repository has [an example configuration for JLCPCB](https://github.com/INTI-CMNB/KiBot/blob/master/docs/samples/JLCPCB.kibot.yaml). However, for a mechanical keyboard and running it as a GitHub Action workflow we can simplify things a bit.

```yaml
kibot:
  version: 1

outputs:
  - name: JLCPCB_gerbers
    comment: Gerbers compatible with JLCPCB
    type: gerber
    dir: gerbers  
    options: &gerber_options
      exclude_edge_layer: true
      exclude_pads_from_silkscreen: true
      plot_sheet_reference: false
      plot_footprint_refs: true
      plot_footprint_values: false
      force_plot_invisible_refs_vals: false
      tent_vias: true
      use_protel_extensions: false
      create_gerber_job_file: false
      disable_aperture_macros: true
      gerber_precision: 4.6
      use_gerber_x2_attributes: false
      use_gerber_net_attributes: false
      line_width: 0.1
      subtract_mask_from_silk: true
    layers:
      # Usually there are no inner layers required for a keyboard PCB,
      # if you do have inner layers, you need to add additional lines here
      - F.Cu
      - B.Cu
      - F.SilkS
      - B.SilkS
      - F.Mask
      - B.Mask
      - Edge.Cuts

  - name: JLCPCB_drill
    comment: Drill files compatible with JLCPCB
    type: excellon
    dir: gerbers
    options:
      pth_and_npth_single_file: false
      pth_id: '-PTH'
      npth_id: '-NPTH'
      metric_units: false
      output: "%f%i.%x"
```

This config generates gerbers for the layers specified above and the drill file. Normally KiBot could zip these together as well, but since the workflow artifact is naturally zipped there is no need to do this in KiBot. Store the file somewhere in your repository.

## Create a GitHub Action workflow

Once the KiBot config is ready, we can integrate it into a workflow. First step in the workflow file is to set a name and some triggers, the example for kicad-export triggers on changes to .sch and .kicad_pcb file but I think it's good to expand it a bit. 

```yaml
name: Generate Gerber files

on:
  push:
    paths:
    - '**.sch'
    - '**.kicad_pcb'
    - '.github/**.yml' # Trigger on changes to the workflow file
    - '**.kibot.yml' # Trigger on changes to the KiBot config file used in the workflow

  # Repeat for PRs
  pull_request: 
    paths:
    - '**.sch'
    - '**.kicad_pcb'
    - '.github/**.yml'
    - '**.kibot.yml'

  # To allow for manual triggering from the Action console, add workflow_dispatch
  workflow_dispatch:
    inputs:
      name:
        description: 'Workflow run name'
        required: false
        default: 'Manual test run'

```

Now we have made sure the workflow gets triggered, let's add a job!

```yaml
jobs:      
  pcb:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - uses: nerdyscout/kicad-exports@v2.3.1
      with:
        config: kicad/jlcpcb_gerbers.kibot.yml
        dir: pcb
        # The following lines find the schema and PCB files, here it assumes your KiCad project is located in kicad/pcb
        schema: 'kicad/pcb/*.sch' 
        board: 'kicad/pcb/*.kicad_pcb'
    - name: upload results
      uses: actions/upload-artifact@v2
      with:
        name: pcb
        path: pcb
```

In my project, I have three separate KiCad project and I add them as separate jobs. This way they are run independently and in parallell, saving some time making sure things are nice and compartmentalized.

## Conclusion

Maybe generating an automated workflow for generating gerbers feels unnecessary for hobby hardware projects since you are not likely to produce many actual prototypes. For my mechanical keyboard project I am done with v1.0 and have had it manufactured, but feel like I can start tackling some known areas of improvmement even if I will not have them prototyped any time soon. This workflow helps make these improvements accessible to other who might want to try to have it produced, and of course myself if I have more manufactured in the future.

