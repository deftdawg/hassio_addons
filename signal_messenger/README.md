[![](logo.png)](https://www.signal.org/)

# Signal Messenger

Rest-API to Signal-CLI Home Assistant add-on (deftdawg fork with openclaw support)

# How to use this add-on

Install the add-on, choose your desired port, start.

After the add-on is started follow the directions at the link below starting from "Register phone number"

https://github.com/deftdawg/signal-cli-rest-api/blob/master/doc/HOMEASSISTANT.md

Then proceed here:

https://www.home-assistant.io/integrations/signal_messenger/

# API details

If you want to use REST to receive messages in HA, you can find more details [here](https://github.com/deftdawg/signal-cli-rest-api)

This fork adds `/api/v1/rpc`, `/api/v1/events`, and `/api/v1/check` endpoints for [openclaw](https://github.com/deftdawg/openclaw) integration.

It is highly suggested that you use the machine IP address, rather than the loopback address mentioned in the upstream container documentation to register numbers. In HAOS everything is containerized and loopback addresses stay inside the respective containers.

Based on work by [@bbernhard](https://github.com/bbernhard) ([signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api)) and [@haberda](https://github.com/haberda) ([hassio_addons](https://github.com/haberda/hassio_addons)).
