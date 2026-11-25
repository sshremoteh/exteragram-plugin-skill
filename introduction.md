---
title: "Introduction"
source: "https://plugins.exteragram.app/docs"
---

# Introduction

Plugin development in the language most Telegram modders already use.

# exteraGram Plugins

exteraGram plugins are written in Python and run inside the app through the built-in plugin engine.

The SDK in this repository is powered by [Chaquopy v16](<https://chaquo.com/chaquopy/doc/16.0/>) for Python interop and Xposed-style method hooking.

## Current baseline

  * Current SDK version in this repository: `1.4.4.3`
  * Python runtime: `3.11`
  * Recommended minimum app version for this SDK: `exteraGram 12.5.1+`

> > [!NOTE] **App version matters** > If the app build is too old, the SDK enters safe mode and plugin modules refuse to initialize. In practice, treat12.5.1+as the baseline when writing or testing new plugins against this tree.

## What you can do with the SDK

  * register high-level plugin lifecycle hooks through [`BasePlugin`](</docs/plugin-class>)
  * intercept requests, responses, updates, and outgoing messages
  * create plugin settings pages and custom settings rows
  * show Telegram-style dialogs and bulletins
  * send or edit messages and media
  * reflect into Java classes and fields
  * hook Java methods and constructors
  * generate Java proxy classes from Python with [`class proxy`](</docs/class-proxy>)

## Recommended reading order

  1. [Setup](</docs/setup>)
  2. [First Plugin](</docs/first-plugin>)
  3. [Plugin Class](</docs/plugin-class>)
  4. [Plugin Settings](</docs/plugin-settings>)
  5. [Client Utilities](</docs/client-utils>)

## Main modules at a glance

  * `base_plugin`: metadata, lifecycle, event hooks, menu items, Xposed helpers
  * `android_utils`: UI thread helpers, listener wrappers, logging, clipboard
  * `client_utils`: requests, controllers, queues, sending and editing messages
  * `file_utils`: directories plus simple file I/O helpers
  * `hook_utils`: reflection helpers for classes and private fields
  * `ui.*`: dialogs, bulletins, settings
  * `extera_utils.*`: class proxy DSL, text formatting helpers, metadata utilities

[NextSetup](</docs/setup>)
