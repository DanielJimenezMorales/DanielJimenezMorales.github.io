---
layout: page
title: Projects
subtitle: Check what I have done!
---

## Online Multiplayer FPS Prototype
![Online Multiplayer FPS Prototype cover image](/assets/img/online-multiplayer-FPS-prototype-cover-image.JPG){: .mx-auto.d-block :}\
This project was the outcome of my bahcelor's final degree project, which received a Honor Award. My final degree project documentation is available [here](https://drive.google.com/drive/folders/16P-oHr5KmGAhIyj1V6s2CAYKn0FFZtFc?usp=drive_link "here") (So far only in Spanish)

In this project I'm tackling all the problems related to network such as latency, jitter or packet loss. Moreover, I need the players to feel a smooth gameplay despite all these issues, thus minimizing the chances of affecting the gameplay. I have also handled most of the high-level netcode aspects, including various latency compensation techniques and achieving a full server authority, as well as implementing snapshot interpolation, among other features.

This prototype is hugely inspired by the Quake III Arena network architecture using techniques such as entity events or temporary entities.

In order to develop the network architecture of this prototype I conducted extensive investigations by reading numerous technical papers related to various aspects of networked games. This process allowed me to gain a deep understanding of the underlying theory, enabling me to effectively implement these concepts in the code.

#### Technologies used:
- Unity3D
- Netcode For GameObjects library
- Photon Relay Services

#### Netcode features implemented:
- Client Side prediction
- Client Side entity interpolation & extrapolation
- Server Side hit registration with latency compensation
- Authoritative Server
- Server Side input prediction
- Jitter buffers/Playout delay buffers

<button name="button" onclick="window.location.href = 'https://danieljimenezmorales.itch.io/online-multiplayer-fps-prototype';" style="border-radius: 8px; background-color: #E79322; border: 2px solid black; font-size: 20px">Click here to download it</button>