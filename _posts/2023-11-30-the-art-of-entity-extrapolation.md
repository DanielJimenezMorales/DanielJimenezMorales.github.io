---
layout: post
title: The art of Entity Extrapolation
subtitle: All you need to know about challenges and techniques related to HitReg
tags: [Hit registration, latency concealment, rollback, Shot behind covers]
comments: true
readtime: true
author: Daniel Jiménez Morales
---

## What is interpolation?
Entity interpolation is a latency compensation technique responsible for determining the state of objects controlled by other players or artificial intelligence based on a slightly earlier world state [44]. Its goal is to provide a smoother movement of remote entities on clients. Without this technique, the client would experience small teleportations each time an update arrives from the server, potentially impacting user experience quality [48]. These small teleportation depend on three main factors:
1. The server's tickrate
2. Network conditions (jitter, packet loss)
3. Entity's movement

### The server's tickrate
The lower the server's tick rate, the more significant these client-side teleportations become since the distance between two consecutive server positions will be bigger. Think of this like the FPS (Frames per second) of a video. Fewer FPS results in less smoothness. This becomes noticeable when the game's update rate is higher than the server's tick rate, making the client to render during multiple consecutive frames the entity at the same position as the last frame. Server's tckrate vary across different games, taking values like 25Hz, 30Hz, 60Hz, etc.

### The network conditions
Conversely, increasing the server's tick rate doesn't guarantee smooth entity movements since, as explained in a [previous article](https://danieljimenezmorales.github.io/2023-07-06-challenges-and-issues-of-the-network/), the network presents a few issues that need to be handled. It is possible that network packets arrive unordered or late due to jitter, making them be obsolte by the time they arrive. Also, these packets can simply not arrive to the destination host. In these cases, we would be loosing information, and even with a high server tickrate, these issues will be perceptible for the player since they will directly affect to the smoothness of the entity's movement.

## How Interpolation works?
Interpolating an entity only requires two states. The initial state (i) and the end state (ii). However, due to network issues such as jitter or packet loss, the end state might not always arrive by the time the client needs to interpolate.

To tackle this issue, rather than inmediately applying incoming states from the server, the interpolation system stores them in a history of various states within a buffer. This buffer, commonly known as 'Jitter Buffer' or 'Playout Delay Buffer', it is a well known technique in various aspects of Network programming that in this case, provides a continuous interpolation guaranteeing a visual smoothness [37].

When employing this buffer, it is neccesary to decide its size. The size of the Jitter buffer represents the number of states the client should receive from the server before initiating the interpolation. By doing this, in case an incoming state get lost, we still have a more states buffered to keep our interpolation working until the next valid state arrives. If the size of the buffer is too small (e.g just one state), it is very likely that due to network issues the buffer becomes empty, leading to glitchy movement. On the other hand, if the buffer size is too large, the client introduces a great amount of constant additional latency, causing remote entities to be rendered in a state too far in the past, potentially impacting other systems like the [Hit Registration](https://danieljimenezmorales.github.io/2023-10-29-the-art-of-hit-registration/) process [22]. It is possible to calculate the amount of additional latency that the interpolation's jitter buffer is adding using the following formula [37]:

[PONER FORMULA AQUÍ]

The following image displays an example taken from the Source Engine, showcasing the interpolation technique and its jitter buffer [49]. In this scenario, the client receives 20 snapshots per second, resulting in a time difference of 50 milliseconds between two consecutive snapshots. Additionally, the client stores these snapshots in a jitter buffer for 100 milliseconds, which is equivalent to holding two snapshots before initiating the interpolation. This setup allows the client to keep stored one extra snapshot for interpolation if one incoming snapshot is lost. In this particular instance, the last snapshot was received by the client at timestamp 10.30. However, the current client time is 10.32. Since the jitter buffer adds 100 milliseconds of delay, the interpolated time will be 10.22. Consequently, the client will interpolate between snapshots 340 and 342. If due to packet loss, snapshot 342 hadn't arrived, the client would still have interpolated between snapshots 340 and 344.

![Interpolation system example from Source Engine](/assets/img/InterpolationAndExtrapolation/source-engine-interpolation-example.JPG){: .mx-auto.d-block :}

Jitter buffers promote consistency by increasing data availability during the interpolation process. As a consequence, they add additional latency before processing incoming data from the server, reducing responsiveness.

## Interpolation algorithms
There are different ways to employ calculate an interpolated state based on the initial and end states.

### Linear interpolation
The easiest interpolation algorithm is known as 'Linear interpolation' [52]. This technique generates smooth paths between two points by creating a straight line that joins them. However, the final path contains discontinuities when joining the path segments generated. This issue is represented in the following image as 'peeks'.

![Linear interpolation](/assets/img/InterpolationAndExtrapolation/linear-interpolation.JPG){: .mx-auto.d-block :}

## What is entity extrapolation?
Extrapolation is another latency compensation technique aimed at predicting future states of objects controlled by other players, assuming their current behaviours will remain unchanged [44]. This method is also known as Dead Reckoning. Extrapolation occurs when the jitter buffer used in interpolation techniques becomes empty due to network issues, lacking additional states to interpolate between [22]. In such situations, uncertainty arises in the entity's state, which needs to be managed. In order to manage it, extrapolation assumes that the entity continues moving in the same direction as it did in the last state received from the server. Based on that assumption, the next state is predicted.

The success of extrapolation techniques is closely tied to the type of videogame in which they are applied. In racing games, for instance, the player's maneuverability regarding the car's movement is relatively limited [48]. The position of a vehicle at high speeds depends significantly on its velocity, direction and previous position. For example, if a vehicle is moving a 100 meters per second, it is almost certain that after 1 second, it will have advanced 100 meters. The player could have accelerated, decelerated or event turned, but the difference in the new position compared to the predicted one would be small. Unless the vehicle collides with an object, its speed won't be drastically affected.

The same principle could be applied to games with slow movements, such as World Of Warships. Since these ships move really slow and take a lot of time to accelerate or deccelerate them the plauer's maneuverability is really limited making it really simple to predict future states based on the ship's current state.

On the other hand, games like first-person shooters involve much more dynamic movements where players can rapidly change direction, making the movement far less predictable than in the previous examples [22]. This will cause that the difference between the server's true position and the estimated uncertainity in the extrapolation process increase over time. By doing so, if the time that extrapolation algorithms have been active in a row exceeds a threshold, these predictions are stopped. This helps reduce the maginitude of entity teleportations genetared during the processs of correction when the server's true position is received.

If the maximum extrapolation time is reached and a server's state is not received within a reasonable time, the following actions may be necessary [37]:
- Force the affected client disconnection.
- Freeze the affected client's game and wait for it to receive messages from the server to resume the game. If unsuccessful, Force disconnection.
- Disconnect the affected client and inmediately after attempt to reconnect it.

Upon receiving messages again from the server while extrapolations are being applied, the transition back to interpolation with the new information should be as seamless as possible in order to maintain the player's experience quality. Ofter, extrapolation may have caused errors between the current entity position and its intended position, requiring corrections. In order to apply this correction we could have three different scenarios based on the error's magnitude:
1. If the error is significant, the best approach is to make an abrupt shift teleporting the current entity to the correct position received from the server.
2. If the entity position only differs slightly from the true server's position, a better approach would be to apply a smoother correction over time by applying a percentage of the correction in each frame. This technique is known as exponential smoothing. Staken and AlRegib conducted a study describing and evaluating different exponential smoothing algorithms in collaborative virtual environments to assess their effectiveness [50].
3. If the error is minimal, the correction can be ignored.

## Different approaches to implement interpolation and extrapolation algorithms

### Using physics!
The simplest approach to apply Dead Reckoning is through the physics equations of motion. The simplest equation is the one known as Uniform Rectilinear Motion (URM). This equation calculates a future position based on the current position, velocity and a time increment. These are the inputs:
- r[k]: Position at time instant k
- v[k]: Velocity at time instant k
- i: Time increment
- s: Elapsed time between two consecutive updates.

Using these parameters, we could calculate the position at time instant k + i emplying the following formula:

https://latex.codecogs.com/svg.image?&space;r[k&plus;i]=r[k]&plus;t\ast&space;v[k],t=i\ast&space;s

This is the simplest formula of motion. However, we 

### Using splines

### Projective Velocity Blending

### DOOM III approach
