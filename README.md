# Charon — Client-Side Multimedia for SPICE virtual desktops

> **Status: archived Proof of Concept (2020–2021) — outcome: positive.**
> The concept was validated end to end. The code is published for reference and is not maintained.

## The problem

In a virtual desktop (Windows guest on KVM/QEMU, accessed through SPICE), multimedia playback is decoded **inside the VM** and the decoded frames are then re-encoded and pushed to the client as a remote display. The result is high CPU load on the host, heavy bandwidth, and choppy audio/video on the client — while the client machine sitting in front of the user has a perfectly capable GPU and decoder doing nothing.

## The idea

Don't send pixels — send the **original compressed stream**.

Charon ferries the media file, still compressed, from the Windows guest across the SPICE connection to the Linux client, where it is decoded and played locally with the client's own capabilities (hardware decoding, local audio). The guest only sends playback commands and the raw byte stream; the client does the heavy lifting.

## Architecture

```
 Windows guest (VM)                         Linux client
┌────────────────────────────┐            ┌──────────────────────────────────┐
│ bPlayer frontend (Qt GUI)  │            │ bPlayer backend ──► mpv           │
│        │ TCP localhost:60999│            │        ▲  local socket            │
│        ▼                   │            │        │  (port auto-detected)    │
│ spice-conduit-d (service)  │   SPICE    │ remote-viewer (patched,           │
│        │ virtio-serial port├───────────►│   "conduit" branch)               │
│        ▼                   │  port      │                                   │
│ \\.\global\io.bplayer.data.0│ channel   │                                   │
└────────────────────────────┘            └──────────────────────────────────┘
```

1. **bPlayer frontend** (guest) — the user drops a media file; the frontend serves its bytes and sends control commands (play, pause, seek, volume) through a custom stream/control protocol.
2. **spice-conduit-d** (guest) — a Windows service that bridges the local TCP endpoint to the virtio-serial port `io.bplayer.data.0`, declared in the VM's libvirt XML as a SPICE port channel.
3. **remote-viewer, patched** (client) — the SPICE client exposes the port channel as a local socket (`NetworkLocalServer`) in a temporary folder.
4. **bPlayer backend** (client) — finds the socket, feeds the stream to **mpv** through its IPC interface, and forwards playback state (position, state, volume) back to the frontend.

## Repositories

| Repository | Side | Role | License |
|---|---|---|---|
| [bPlayer](https://github.com/unau-thorized/bPlayer) | guest + client | Frontend (Qt GUI), backend (mpv controller), shared protocol library, socket-bridge tool | GPL-2.0 |
| [spice-conduit-d](https://github.com/unau-thorized/spice-conduit-d) | guest | Windows service bridging local TCP ↔ virtio-serial port | MIT |
| [virt-viewer](https://github.com/unau-thorized/virt-viewer/tree/conduit) (`conduit` branch) | client | Fork of upstream virt-viewer 9.0 adding the conduit local socket | GPL-2.0 (upstream) |

## Outcome

The PoC worked: audio and video files selected inside the Windows guest were streamed compressed over SPICE and played back natively on the Linux client, with remote control of playback from the guest.

Known limitations at the time of archiving:

- the backend could crash after the first playback when mpv was quit from its OSD;
- the frontend talks to spice-conduit-d over TCP instead of a named pipe;
- seeking had no visual feedback and could be slow;
- build systems are qmake/autotools; the guest service requires MSVC (MinGW lacks `CancelIoEx`).

## Building

Each repository has its own build notes. In short: Qt 5 + QtCreator for bPlayer and spice-conduit-d (the latter also needs [QtService](https://github.com/Skycoder42/QtService)); autotools for the virt-viewer fork.

## Author

*Unau Thorized* · [spaziocurvo.com](https://www.spaziocurvo.com)
