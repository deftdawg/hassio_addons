# deftdawg's Home Assistant Add-ons

Fork of haberda's add-ons with openclaw support.

# Installation

Either click the button below or add `https://github.com/deftdawg/hassio_addons` to your addon repositories.

[![Install this repository](https://my.home-assistant.io/badges/supervisor_add_addon_repository.svg)](https://my.home-assistant.io/redirect/supervisor_add_addon_repository/?repository_url=https%3A%2F%2Fgithub.com%2Fdeftdawg%2Fhassio_addons%2F)

## Changes from upstream

This fork builds signal-cli-rest-api from [deftdawg/signal-cli-rest-api](https://github.com/deftdawg/signal-cli-rest-api) which adds:
- `/api/v1/rpc` - JSON-RPC passthrough endpoint
- `/api/v1/events` - SSE stream for incoming messages
- `/api/v1/check` - Health check endpoint

These endpoints provide signal-cli daemon HTTP API compatibility required for [openclaw](https://github.com/deftdawg/openclaw) integration.

## Note
Testing is primarily done on x86. If you run a different architecture and have issues, please report them.
