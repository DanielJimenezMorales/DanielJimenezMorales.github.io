---
layout: page
title: Projects
subtitle: Check what I have done!
---

## Online Multiplayer FPS Prototype
![Online Multiplayer FPS Prototype cover image](/assets/img/online-multiplayer-FPS-prototype-cover-image.JPG){: .mx-auto.d-block :}\
This project was the outcome of my bahcelor's final degree project, which received a Honor Award. My final degree project documentation is available [here](https://drive.google.com/drive/folders/16P-oHr5KmGAhIyj1V6s2CAYKn0FFZtFc?usp=drive_link "here") (So far only in Spanish)

This project has been fully developed by myself, including all the different gameplay and multiplayer systems where I'm tackling all the problems related to network such as latency, jitter, and packet loss. The main goal is to provide a seamless multiplayer experience to players apart from a smooth gameplay despite all network issues.

I have also handled all the high-level netcode aspects, including the implementation of various techniques to compensate latency and achieving full server authority, among other features.

This prototype is hugely inspired by the Quake III Arena network architecture using techniques such as entity events or temporary entities, which are explained in detail in one of the linked posts below.

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

## Network Library for realtime multiplayer games (C++)
![Network Library for realtime multiplayer games cover image](https://github.com/DanielJimenezMorales/Network-Library/social.png){: .mx-auto.d-block :}\
In this project, I am developing a C++ Network Library using UDP and BSD sockets to support realtime multiplayer games. [Link here](https://github.com/DanielJimenezMorales/Network-Library "Link to repository")

#### Technologies used:
- BSD Sockets
- Premake