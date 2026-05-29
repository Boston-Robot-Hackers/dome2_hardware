────────────────────────────────────────────
        5-IN-1 USB A & C HUB
────────────────────────────────────────────

               TO COMPUTER
            ┌───────────────┐
            │ USB-C plug    │
            │ or USB-A plug │
            └──────┬────────┘
                   │
                   ▼

        ┌───────────────────────┐
        │        HUB            │
        │                       │
        │ [USB-C POWER IN]      │◄── 5V/3A power
        │   power only          │
        │                       │
        │ [USB-C DATA]          │◄── SSD / phone
        │                       │
        │ [USB 2.0]             │◄── keyboard
        │                       │    mouse
        │ [USB 2.0]             │◄── ESP32
        │                       │    serial
        │ [USB 3.0 BLUE]        │◄── SSD / camera
        └───────────────────────┘


────────────────────────────────────────────
PORT TYPES
────────────────────────────────────────────

USB-C POWER
  • feeds power INTO hub
  • usually no data
  • usually no laptop charging

USB-C DATA
  • regular USB data
  • usually no video

USB 3.0 (blue)
  • fast
  • 5 Gbps
  • SSDs / cameras

USB 2.0
  • slower
  • 480 Mbps
  • keyboard / mouse / serial


────────────────────────────────────────────
IMPORTANT
────────────────────────────────────────────

ALL PORTS SHARE:
  • power
  • bandwidth

GOOD:
  • serial adapters
  • ESP32
  • keyboard/mouse

RISKY:
  • OAK-D Lite
  • powering Raspberry Pi
  • multiple high-current devices
────────────────────────────────────────────