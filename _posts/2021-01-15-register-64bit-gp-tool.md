---
layout: post
title: Register Custom GP Tool for 64 Bit Background Geoprocessing
author: ujr
date: 2021-01-15
---

By default, Custom Function GP Tools are registered for 32bit use only.
On ArcGIS Server and when using the 64bit Background Geoprocessing
option for ArcGIS Desktop, GP tools can be run in a 64bit process.
This requires special registration.

## Build your Custom GP Tool as usual

- set platform target to **Any CPU**
- build to generate the DLL

## Register for 32 bit Apps (ArcCatalog, ArcMap)

- call **ESRIRegAsm.exe** to create necessary Windows Registry entries  
  (find this tool in *C:\Program Files (x86)\Common Files\ArcGIS\bin*)
- these entries are in the locations for 32 bit applications (WOW64)

## Register for 64 bit Apps (Server, Background Geoprocessing)

See the Esri technical article at
<https://support.esri.com/en/technical-article/000011695>
(Article ID 000011695, updated for 10.8.1 on 2020-07-29).
Essentially, follow these steps:

- Use the 32bit **ESRIRegAsm.exe** to create a registry (.reg) file:

```cmd
ESRIRegAsm.exe C:\Path\To\MyGP.dll /regfile:MyGP.reg
```

- From a 64bit command prompt (*C:\Windows\System32\cmd.exe*),
  run **regedit.exe** with the file just created:

```cmd
regedit C:\Path\To\MyGP.reg
```

- The registry entries should now also have been added to the 64bit locations

- Restart ArcMap/ArcCatalog and enable background processing
  from the Geoprocessing Options, then run your custom GP tool

The typical symptom of running in 64-bit background a custom function
GP tool that has been registered for 32bit only is the error
"000816: The tool is invalid". When running the tool from standalone
Python, you get a different error, probably a TypeError from Python.

## References

- <https://support.esri.com/en/technical-article/000011695>
- <https://desktop.arcgis.com/en/arcobjects/latest/net/webframe.htm#AboutESRIRegAsmUtility.htm>
