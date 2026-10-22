# network-manager-miracast-fix

NetworkManager with a `wifi-p2p.go-intent` property, so Miracast works on MediaTek MT7921 (and any other adapter that falls over when forced to be P2P group owner).

This is real NetworkManager history plus the patch, not a source dump. The branch [`wifi-p2p-go-intent`](../../tree/wifi-p2p-go-intent) is exactly what goes upstream; `main` is that plus this README.

Upstream:
- NetworkManager: https://gitlab.freedesktop.org/NetworkManager/NetworkManager
- gnome-network-displays (companion): https://gitlab.gnome.org/GNOME/gnome-network-displays

## Problem

Miracast to a Wi-Fi Display sink through gnome-network-displays never connects on MT7921. P2P discovery and GO negotiation succeed, then group formation dies:

```
P2P-GO-NEG-SUCCESS role=GO ...
WPS-PBC-ACTIVE
P2P-GROUP-FORMATION-FAILURE
```

The sink never associates back. Works from Windows.

mt7921 P2P GO mode does not drive WFD sinks; client mode does. But NetworkManager always takes the GO role, because it hardcodes go_intent in `nm_supplicant_interface_p2p_connect()`:

```c
g_variant_builder_add(&builder, "{sv}", "go_intent", g_variant_new_int32(7));
```

7 beats what most sinks send, and there was no way to lower it.

Checked by hand against wpa_supplicant:

```
go_intent=7  ->  role=GO      ->  FORMATION_FAILURE
go_intent=0  ->  role=client  ->  group up, DHCP, video
```

## Fix

`NMSettingWifiP2P` gets a `go-intent` property, range 0..15. The default `-1` keeps the old 7, so existing profiles behave exactly as before. The value is passed down to wpa_supplicant's P2P `Connect` call.

It is wired through everywhere a new property has to be: libnm API (`nm_setting_wifi_p2p_get_go_intent()`, `NM_AVAILABLE_IN_1_58`), D-Bus and keyfile, nmcli metadata and docs, and NEWS.

```
$ nmcli c add type wifi-p2p con-name tv wifi-p2p.peer 02:11:22:33:44:55 wifi-p2p.go-intent 0
$ nmcli c mod tv wifi-p2p.go-intent default      # back to built-in (7)
$ nmcli c mod tv wifi-p2p.go-intent 16
Error: failed to modify wifi-p2p.go-intent: '16' is out of range [-1, 15].
```

gnome-network-displays needs a one-line companion change to set `go-intent` to 0 on the profiles it creates; that goes to GNOME separately.

## Build (Arch)

```
sudo pacman -S --needed base-devel meson ninja gobject-introspection gtk-doc python-gobject python-dbus \
    libndp jansson libnl libnewt curl
meson setup build -Dtests=yes -Ddocs=true -Dintrospection=true \
    -Dvapi=false -Dqt=false -Dnbft=false -Dman=false -Dppp=false -Dmodem_manager=false \
    -Dovs=false -Dteamdctl=false -Dlibaudit=no -Dselinux=false -Debpf=false -Dclat=false
ninja -C build
meson test -C build
```

`-Ddocs=true` is only needed to check the generated docs files (`settings-docs.h.in`, the `gen-metadata-*.xml.in`); drop it and `-Dintrospection=true` for a plain build. CLAT is off only because it needs `bpftool`.

## Status

- Full test suite: 75/75 pass on the patched tree.
- Live-tested with an H.264 stream from an MT7921 laptop to a 4K TV.
- NetworkManager MR: pending fd.o account verification.
- gnome-network-displays companion change: to do.

## Driver

This is a userspace workaround. The real bug is in `mt76`: it cannot be a reliable P2P GO for WFD. That is not touched here; a report goes to the mt76 maintainers and linux-wireless once the userspace side is settled.

Original NetworkManager README: [README.upstream.md](README.upstream.md)
