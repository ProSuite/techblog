---
layout: post
title: Build Add-Ins without the SDK
author: ujr
---

How to build an Add-In for ArcMap without the SDK.

**Why?** After updates of Visual Studio the SDK regularly
cannot find its dependencies, usually one or the other
of a Visual Studio DLL. Could be fixed with some effort
(keeping old VS around, assembly binding redirection),
but the problem recurs.
The SDK is one of those dependencies you do not want.

**How?** ... (see my BoVIS solution) ...
