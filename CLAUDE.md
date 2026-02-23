# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

This is a **documentation-only repository** — a setup guide for configuring voice typing on Ubuntu using [VoxType](https://voxtype.io) (Whisper-based speech recognition) and [ydotool](https://github.com/ReimuNotMoe/ydotool) (virtual keyboard input). There is no application source code, build system, or test suite.

The entire content is in `README.md` and `LICENSE` (MIT).

## Repository Purpose

The README walks through a complete setup: installing ydotool, configuring uinput permissions, installing VoxType, downloading a Whisper model, creating systemd user services, and enabling GPU acceleration (Vulkan). It also covers troubleshooting, model trade-offs, updating, and uninstalling.

## When Editing

- All user-facing content is in `README.md`. Keep the existing structure: Prerequisites, Setup (numbered steps), Usage, Performance Optimization, Troubleshooting, Updating, Uninstall.
- Shell commands in the guide target Ubuntu 24.04 with systemd user sessions.
- The uninstall section is marked as untested — preserve that note unless the steps have been verified.
- VoxType config lives at `~/.config/voxtype/config.toml` (TOML format). The example config in the README should stay consistent with VoxType's actual options.
