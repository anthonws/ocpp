# ⚡ Fork note — Huawei FusionCharge OCPP workarounds & firmware analysis

This is a community fork of [`lbbrhzn/ocpp`](https://github.com/lbbrhzn/ocpp), maintained to **work around — and document — serious OCPP defects in Huawei FusionCharge AC wallboxes** (e.g. `SCharger-7KS-S0`, firmware `FusionCharge V100R023C10SPC220`).

**The problem:** the charger's OCPP client has **no dead-peer detection**. After any transient network interruption (e.g. a nightly Wi-Fi AP reboot) it loses the OCPP session and **never reconnects** — it stays reachable on the IP network but OCPP-dead, shows the fault LED ("red ring"), and only recovers on a **physical power-cycle**. Charging is unaffected; remote control/monitoring via OCPP is lost.

**What this fork adds:** central-system-side detection of the dead peer (fast WebSocket-ping + force-close), honest `unavailable` state so automations don't act on a phantom charger, a lowered heartbeat interval, and a best-effort wake. It cannot wake a charger that has fully *parked* — the only reliable cure is to stop the network from blipping (wired link / no AP reboot).

- 📄 **Full write-up — failure analysis, the HA-side fix, live test results:** [`docs/huawei-fusioncharge/README.md`](docs/huawei-fusioncharge/README.md)
- 🔬 **Firmware reverse-engineering details:** [`docs/huawei-fusioncharge/firmware-reverse-engineering.md`](docs/huawei-fusioncharge/firmware-reverse-engineering.md)
- 🛠 **Code changes:** [`custom_components/ocpp/ocppv16.py`](custom_components/ocpp/ocppv16.py) (see releases tagged `…-fork.N`).

> Documentation is shared for interoperability and right-to-repair purposes; not affiliated with or endorsed by Huawei, and no proprietary firmware binaries are redistributed.

---

*Upstream README below.*

[![hacs_badge](https://img.shields.io/badge/HACS-Default-orange.svg)](https://github.com/custom-components/hacs)
[![codecov](https://codecov.io/gh/lbbrhzn/ocpp/branch/main/graph/badge.svg?token=3FRJIF5KRW)](https://codecov.io/gh/lbbrhzn/ocpp)
[![Documentation Status](https://readthedocs.org/projects/home-assistant-ocpp/badge/?version=latest)](https://home-assistant-ocpp.readthedocs.io/en/latest/?badge=latest)
[![hacs_downloads](https://img.shields.io/github/downloads/lbbrhzn/ocpp/latest/total)](https://github.com/lbbrhzn/ocpp/releases/latest)

![OCPP](https://github.com/home-assistant/brands/raw/master/custom_integrations/ocpp/icon.png)

This is a Home Assistant integration for Electric Vehicle chargers that support the following Open Charge Point Protocols 1.6j, 2.0.1 and 2.1 (experimental).

* based on the [Python OCPP Package](https://github.com/mobilityhouse/ocpp).
* HACS compliant repository 

Documentation can be found here [home-assistant-ocpp.readthedocs.io](https://home-assistant-ocpp.readthedocs.io)

**💡 Tip:** If you like this project consider buying me a coffee or a cocktail:

<a href="https://www.buymeacoffee.com/lbbrhzn" target="_blank">
  <img src="https://cdn.buymeacoffee.com/buttons/default-black.png" alt="Buy Me A Coffee" width="150px">
</a>
