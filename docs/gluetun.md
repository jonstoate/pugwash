# Gluetun

This has been written using NordVPN and OpenVPN protocol for our VPN connection.

For a different provider follow the setup guide in the [Gluetun Wiki](https://github.com/qdm12/gluetun-wiki)

Firstly edit `/opt/docker/gluetun/gluetun.yaml` 

```bash
$EDITOR /opt/docker/gluetun/gluetun.yaml
```



The file looks like this:

```yaml
---
services:
  gluetun:
    image: qmcgaw/gluetun
    container_name: gluetun
    # line above must be uncommented to allow external containers to connect.
    # See https://github.com/qdm12/gluetun-wiki/blob/main/setup/connect-a-container-to-gluetun.md#external-container-to-gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    ports:
      - 8888:8888/tcp # HTTP proxy
      - 8388:8388/tcp # Shadowsocks
      - 8388:8388/udp # Shadowsocks
    volumes:
      - /opt/docker/gluetun/config:/gluetun
    environment:
      # See https://github.com/qdm12/gluetun-wiki/tree/main/setup#setup
      - VPN_SERVICE_PROVIDER=ivpn
      - VPN_TYPE=openvpn
      # OpenVPN:
      - OPENVPN_USER=
      - OPENVPN_PASSWORD=
      # Wireguard:
      # - WIREGUARD_PRIVATE_KEY=wOEI9rqqbDwnN8/Bpp22sVz48T71vJ4fYmFWujulwUU=
      # - WIREGUARD_ADDRESSES=10.64.222.21/32
      # Timezone for accurate log times
      - TZ=Australia/West
      # Server countries - a comma seperated list of counties
      - SERVER_COUNTRIES=Australia 
      # Server list updater
      # See https://github.com/qdm12/gluetun-wiki/blob/main/setup/servers.md#update-the-vpn-servers-list
      - UPDATER_PERIOD=0

```

We will need to update:

```
      - VPN_SERVICE_PROVIDER=ivpn
      - VPN_TYPE=openvpn
      # OpenVPN:
      - OPENVPN_USER=
      - OPENVPN_PASSWORD=
  ...
        - TZ=Australia/West
        - SERVER_COUNTRIES=Australia
        - UPDATER_PERIOD=0
```

