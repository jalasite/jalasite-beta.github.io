---
title: "Setting Usb D-link DWA-125"
layout: post
categories: posting update
post_desc: Berikut ini langkah - langkah yang saya lakukan untuk dapat menggunakan Usb Wifi D-link DWA-125 pada linux fedora 34.
---

Berikut ini langkah - langkah yang saya lakukan untuk dapat menggunakan Usb Wifi D-link DWA-125 pada linux fedora 34.
1. Buka terminal linux 
2. \# wpa_passphrase "SSID" password >> /etc/wpa_supplicant/wpa_supplicant.conf
3. $ sudo wpa_supplicant -B -Dwext -iusbwifi_kita -c /etc/wpa_supplicant/wpa_supplicant.conf
4. $ sudo dhclient usbwifi_kita
5. $ sudo resolvectl dns usbwifi_kita ip-gateway

