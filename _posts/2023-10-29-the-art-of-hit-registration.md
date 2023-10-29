---
layout: post
title: The art of Hit Registration
subtitle: All you need to know about challenges and techniques related to HitReg
tags: [Hit registration, latency concealment, rollback, Shot behind covers]
comments: true
readtime: true
---

One of the core systems of a competitive networked FPS game, using a snapshot interpolation technique to synchronize the different clients with the server, is its Hit Registration algorithm. In this post, I will explain all the details about this process, discuss the different challenges that developers will need to face and explore various approaches used by well-known FPS games that employ this technique.

## Two types of weapons:
In networked shooting games there are two types of weapons [1]:
- **Hitscan weapons**: These weapons are the most straightforward type to implement. Their bullets travel at an infinite speed, allowing for an instant determination of whether they have hit a target as soon as they are fired.
- **Projectile weapons**: These weapons offer a more realistic behaviour since their bullets are physically simulated and, as a consequence, they travel at a limited speed. Based on these facts, they are more difficult to implement.

## What is Hit Registration and how does it work?
In a client-server architecture with the server acting as the authoritative host, when a player decides to open fire, this action is transmitted to the server. Once it arrives, the server will be responsible for executing all the necessary calculations in order to determine whether the bullet, originated by the client's firing action, has collided with any player or not. This process is known as "Hit Registration", and in the case of hitscan weapons, it is achieved by casting a raycast, which is a straight line designed to identify the different points of intersection between that ray and the different colliders within the game world.

![Raycast behaviour](/assets/img/HitReg/raycast.png){: .mx-auto.d-block :}

## The first issue: Lack of client-side responsiveness
This communication between the client and the server has a problem and it is related to latency. Even in the case that hitscan weapons are being used, when the client detects that the player has pressed the fire button, the shot does not occur immediately. Instead, the client must wait for the server to execute the Hit Registration algorithm, in order to confirm whether the shot has collided with a player or not, and wait until the confirmation message arrives to the client. The generated lag would be noticeable for the player and would degrade the responsiveness of shooting actions in the game.

## The solution: Latency Concealment
To address this issue, a technique called "Latency Concealment" could be employed [2]. This technique aims to mitigate the player's perception of latency by adding both visual and sound effects to ensure the game remains responsive. For example, when a player fires a weapon, multiple effects could be added on the client-side:
- Firing sound.
- Recoil or kickback animation for the weapon.
- Muzzle flash at the weapon's barrel.
- Ejection of a bullet casing.

Additionally, the client can also execute a Hit Registration algorithm in order to display other effects at the position the client believes the bullet has impacted [1]. These effects may include:
- Bullet impact decals.
- Impact sounds.
- Blood particle effects.

![Firing effects GIF](/assets/img/HitReg/fire-effects.gif){: .mx-auto.d-block :}

On the other hand, not everything can be addressed through effects. When firing a weapon, the client is certain that a firing action has occurred, even if it is not sure whether the resulting bullet hit a target or not. As a consequence multiple effects, such as those described above, can be displayed in order to provide feedback to the player about their actions. However, effects with permanent consequences such as decreasing a player's health, or those that represent decisions taken by the server, like shot confirmations (often represented as a hitmarker in most FPS games), must wait until the client receives the server's response before being executed. If the client predicts these permanent decisions incorrectly, it could lead to awkward situations that are difficult to undo.

![Latecy Concealment](/assets/img/HitReg/latency-concealment.png){: .mx-auto.d-block :}

## The second issue: Accuracy errors
Another issue that should be addressed is related to the Hit registration accuracy. In networked games based on a client server architecture, if the client is using interpolation techniques to update remote entity states, it will renders the rest of remote players in a state slightly behind in time from the server's one. This is caused by how interpolation works. When those entity states arrive from the server, they are not immediately applied but buffered until the client has at least two different states in order to interpolate between them and create a smooth transition. This technique adds an additional latency which can be calculated with the following formula:

![Interpolation's buffer delay formula](https://latex.codecogs.com/svg.image?interpolationBufferDelay=\frac{numberOfStatesBuffered}{tickrate})

However, this is not the final formula for estimating the time difference between the client and server states. It is also neccesary to account for the time it takes for data packets to travel from the server to the client, which is calculated as half of the client's average Round-Trip Time (RTT). By incorporating this aspect into the interpolation delay formula, it is possible to obtain an aproximation of how far behind in time the client is rendering remote player states.

>**Note:**
>It is important to note that this is an aproximation. Due to network jitter, the RTT may vary over time, resulting in accuracy errors. That is the reason why using an average RTT is preferred over using the most recent RTT recorded on the server. Using the average RTT will reduce these accuracy errors, especially when dealing with connections experiencing medium to high jitter conditions.

Considering all these factors, the final formula for estimating the time difference between the client and server states will look like as follows:

![Time difference formula between the client and server states](https://latex.codecogs.com/svg.image?timeDifference=\frac{numberOfStatesBuffered}{tickrate}&plus;\frac{averageRTT}{2})

As a consequence of this difference, a situation like the one represented in the image below might occur. By the time the client's input arrives to the server, the position of the target player may have changed, and there is a possibility that after executing the Hit registration algorithm, the shot may result in a miss.

![Graphic representation of accuracy errors caused by latency](/assets/img/HitReg/hit-miss.png){: .mx-auto.d-block :}

## The solution: Server-side rewind
To minimize accuracy errors within the Hit Registration process, a technique called "Server-side rewind" could be employed. This technique aims to reduce the desynchronization, explained in the previous paragraph, between the client and server states by replicating the client's world state on the server to match the moment when the client fired the weapon. In order to achieve this, the server needs to store a history of the past world states allowing it to determine, whenever a client's fire petition arrives, which world state from the history corresponds to the moment the petition was created on the client. This process can be divided into a few steps [3]:
1. Determine the exact moment when the client initiated the shooting request.
2. Select the past world state from the history that best corresponds to the calculated moment in step 1.
3. Move all rewindable entities to the state they had during the selected world state.
4. Apply the Hit Registration algorithm to determine whether the shot was successful or missed based on the selected world state.
5. Move all the rewindable entities back to their current states in order to undo the changes made in step 3.

Let's start from the beginning. Before rolling back the entire server's world state, it is essential to determine when the firing action occurred. However, this operation comes with a few challenges [4]:
1. The server cannot always accurately determine the client's RTT due to packet jitter, which in practice can vary significantly.
2. The client-side interpolation buffer delay, calculated in the previous section, may not always be a fixed value, as it can vary due to packet jitter and/or loss (Depending on the implementation).

Due to these issues, determining this time using only local server information can be difficult, leading to accuracy errors that would be unacceptable. A difference of tens of milliseconds can significally impact the transform of an entity, potentially causing a successful shot to become a miss, or vice versa.

In this specific scenario, Netcode developers make an exception to the "Full server authority" rule, which says that the server should never trust any data coming from a client. In this case, and only in this case, the server allows the client to calculate a timestamp that is going to be sent as part of their inputs. This timestamp represents the estimated server-time calculated by the client at the moment the firing action occurred. This time will be equal to the time it took for the client's input to travel from the client to the server, which is equal to half of the average RTT plus the lag the client has in rendering that past state, equivalent to half of the average RTT plus the client-side interpolation buffer delay. Using these parameters, the final formula would look like this:

![Shot instant formula](https://latex.codecogs.com/svg.image?ShotInstant=CurrentServerTime-averageRTT-clientSideBufferInterpolationDelay).

>**Note:**
>You might be concerned that allowing the client to provide the timestamp for the firing action could lead to cheating. In theory, this is true, and as mentioned earlier, the server should minimize such risks as much as possible since any data trusted from a client is susceptible to be manipulated. However, in practice, cheating through the client's inputs timestamp is quite challenging. While the client could modify this timestamp to turn a near-miss into a hit, it is far easier and harder to detect if they simply modify their inputs directly, for example, using injection aimbots [11] and/or silent aimbots [12].
>
>In a later section of this article, I will explain a technique known as 'Conditional Lag Compensation', which helps prevent cheating through the client's inputs timestamp by limiting the maximum amount of time the server can rollback.

This is the basic formula for calculating the server-time at which the action occurred, which is essentially the rewound time. However, if there are any other additional buffers (e.g. a Server-side inputs playout delay buffer or any kind of jitter buffer), it will be necessary to substract them from the current timestamp within the formula above.

Once the server receives the timestamp when the client fired the weapon, the server will rollback the world state of the game to the one that matches best the client's timestamp. However, it is likely that this time does not match exactly any of the past world states within the history buffer. Due to this, two different approaches could be applied:
- Clamping the rewound time to the closest world state. In this first approach, the accuracy will likely be degraded by a maximum error of ![Maximum accuracy error caused by clamping the rewound time to the closest world state](https://latex.codecogs.com/svg.image?&space;1/(tickrate*2)) milliseconds.
- Interpolating between the two closest world states within the history buffer in order to get an intermediate state that matches exactly the rewound time [1].

This step is crucial to replicate the client's world state precisely at the moment the player has fired the weapon every time the client is using interpolation. However, it is important to note that if the client is using extrapolation techniques when the action was executed, any inaccuracies in the extrapolation algorithm of the target player's state may lead to prediction errors and, as a consequence, missed shots.

Once the target world state that the server needs to rollback to has been found, it is neccessary to apply it to all rewindable entities. This means changing its world position, rotation... There are different approaches to accomplish this:

### First approach
One possible approach consists in storing past states of the different colliders from all the rewindable entities within a history buffer. When rolling back the entities, the server will search for the collider states that match the rewound world state. Once these colliders have been rolled back to their target positions and rotations, the server will execute the Hit Registration algorithm, casting a raycast and checking if any of those colliders has been hit by the ray. Once this operation has been done, all the colliders are returned to their current positions and rotations in order to continue with the world state simulation. This approach requires a higher memory consumption since the server needs to store a history of some past states of all these different entity colliders.

### Second approach (Valorant)
Another approach to implement this rollback is the one used by Riot Games developers in Valorant [5]. Unlike the previous technique, they do not store past states from all the rewindable entity colliders within each world state. Instead, they use animation rollback, a technique to resimulate animations in order to determine the exact pose of all the colliders of an entity at the rewound time. One of the main advantages of this technique is related to memory usage. Unlike the previous method, where each world state needs to store all the colliders resulting from the different bone positions and rotations related to the rewindable entity, with animation rollback only the animation inputs need to be stored. This also means that it is important that this process is deterministic so, based on the same inputs, the algorithm always produce the same outputs.

Due to the loss of performance that this approach was providing to the Valorant's team, developers decided to optimize it by rolling back animations only for those entities that could potentially be hit by the bullet.
To determine whether an entity is within the bullet's range, the server performs a rollback only to a bounding box collider that contains each entire entity. After this step, the server executes a Sphere cast, as shown in the image below, starting from the shooter's position in the direction of their shot, but only against the entity bounding box colliders. Only those entities whose bounding box colliders had been hit by the Sphere cast will get an animation rollback in order to perform a second raycast against those entities.

![Sphere Cast in Valorant](/assets/img/HitReg/valorant-sphere-cast.png){: .mx-auto.d-block :}

As the official Valorant's article says, before applying this optimization, updating the animations for one single player entity could take up to 0.1 milliseconds. However, after implementing this selective animation rollback system, the average time for processing all animations from one single frame was reduced to 0.3 milliseconds.

### Third approach (Overwatch)
Another solution similar to the one developed by the Valorant's team is discussed by Ford, one of the developers of Overwatch, in his GDC 2017 presentation [6]. In this approach, each entity (which can be players, doors, turrets...) is equipped with a collider that contains it. So far, the technique is mostly the same as the bounding box optimization applied by Valorant. However, the bounding box collider of Overwatch's solution not only contains the entity at a specific moment but all positions the entity has been within a time interval ranging from X milliseconds in the past to the current moment. In the image below, an example of these colliders can be observed on the enemy located on the left. Based on this approach, during the Hit Registration process, the first step is to check if the raycast collides with any of these custom bounding box colliders. Once the ray cast has been executed, only those entities whose bounding box collider has been hit by the raycast will be rolled back based on the client's ping in order to obtain the final results.

![Custom bounding box collider in Overwatch](/assets/img/HitReg/overwatch-custom-bounding-box-collider.png){: .mx-auto.d-block :}

## The third issue: Shot behind covers (SBC)
Another challenge that must be addressed when talking about Hit registration algorithms is the phenomenon known as "Shot behind covers" (SBC) [7]. This situation arises when a player with high latency shoots at another player with low latency. It is possible that the low-latency player has already taken cover behind an obstacle and is out of the shooter's line of sight. However, due to the attacker's high latency and the fact that clients render remote entities in a past state, the shooter player will still be able to see its victim. When the high-latency player fires his weapon against the low-latency victim, the server's hit registration algorithm will rollback based on the shooter's latency as explained before, so the more latency it has the more back in time the server will rollback. In this case, the victim may experience an unfair SBC situation, as they were killed inmediately after taking cover. This presents a clear advantage for high-latency players.

>**Note:**
>As discussed in the previous pharagraph, the likelihood of encountering a SBC situation depends on the rollback distance of the victim's entity. The larger the rollback distance, the higher the probability of encountering an SBC scenario. According to the physics formulas of motion, the distance traveled by an object depends not only on time but also on velocity. This means that the rollback distance not only depends on the shooter's average RTT (time) but also on the victim's movement speed (velocity). This relation can be expressed using the following formulas:
>
>Physics formula of motion:
>
>![Physics formula of motion](https://latex.codecogs.com/svg.image?&space;S=S_0&plus;v*t).
>
>Physics formula of motion translated to the current context:
>
>![Physics formula of motion translated into this scenario](https://latex.codecogs.com/svg.image?&space;RollbackDistance=currentPosition&plus;victimsVelocity*ShootersAverageRTT)

The issue of SBC can also manifest in the opposite way. When players step out from behind cover, they can see the other players before their clients have rendered the new attacker's state due to interpolation techniques delay. This situation grants a clear advantage to the attackers, as they can start shooting to other players before the victims have even seen them.

## The solution: Conditional Lag Compensation
To minimize the SBC problem, one of the most commonly used solutions is the one known as Conditional Lag Compensation [8]. This technique makes the server to fully compensate for the shooter's latency only when it does not exceeds a certain threshold. If the threshold is exceeded, the server will clamp the shooter's latency to that threshold and compensate for only that amount. As a consequence, the shooter must anticipate the target's trajectory by aiming ahead in time.

![Conditional Lag Compensation example](/assets/img/HitReg/conditional-lag-compensation.png){: .mx-auto.d-block :}

Games like Battlefield 4 and Overwatch had a limit of 250 milliseconds before stopping lag compensation, while others like Call Of Duty: Infinite Warfare had a limit of 500 milliseconds (except for peer-to-peer connections where the attacker was the host, which had no limit) [9].

>**Note:**
>As previously mentioned in the **Server-side rewind** section, Conditional Lag Compensation can also act as a limitation for cheaters trying to manipulate the client's input timestamp significantly as it will be clamped anyway.

## Other solutions investigated:
This final section explores other solutions that have been proposed for improving different aspects related to the Hit Registration process but, in my opinion, contain. It is important to note that while these solution have their merits, they also come with, in the author's opinion, significant drawbacks. For each solution, I will explain its key aspects and additionaly provide my own opinon about why they may not be ideal.

### Advanced Lag Compensation: Victim's approval (+ Author's opinion)
Another approach to mitigate the problem of being hit after taking cover is proposed by Lee and Chang [10], known as Advanced Lag Compensation. In this approach, when a bullet successfully hits a player, the victim has an opportunity to deny the hit if a "Shot behind cover" situation is detected. Every time the server performs the Hit Registration algorithm, it sends to the victim a hit event notification with the results. However, this event is not considered final, as the victim has two options:
- Confirm the shot. In this case, the victim's client detects that the hit was legitimate and no further calculations are made.
- Deny the shot. In this case, the victim's client perceives that the shot occurred in a "Shot Behind Cover" situation. Once the server receives the denied hit event, it must verify it in order to prevent potential cheating.

This technique introduces an additional delay, caused by the victim's confirmation, equivalent to the victim's round-trip-time (RTT). As a consequence, in situations where a player with low latency shoots at a player with high latency, the additional delay for confirming the shot can become significant. To address this issue, the authors have introduced a maximum latency threshold. If the victim's latency exceeds this threshold, the shot is automatically confirmed, regardless of the victim's late response. By applying this restriction, developers ensure that players with low latency are not negatively impacted by those with high latency, preserving a better gaming experience.

>**Author's opinion:**
>This solution, proposed by Lee and Chang to mitigate the SBC situation, has a notable drawback as it introduces an additional, and in my opinion, unnecessary latency to the hit confirmation. This can increase players perception of latency. As mentioned by Jay Mattis in his Lag Compensation article [4], the client needs to render players using only information coming from the server. In this context, when performing the server-side rollback operation, the server should be able to accurately rewind entities back to a previous state for hit detection. Consequently, there would be no need to ask the "opinion" of clients regarding the server's decisions. The server should be capable of recreating the client's opinion and thereby, eliminating the need for this extra latency.

To conclude, if you have questions about any aspect of this article, you want to contribute with new ideas or you just want to chat, please feel free to contact me.

### References:
- [1] [Implementation and Evaluation of HitRegistration in Networked First Person Shooters](https://liu.diva-portal.org/smash/get/diva2:1605200/FULLTEXT01.pdf)
- [2] [A Survey And Taxonomy Of Latency Compensation Techniques For Network Computer Games](https://dl.acm.org/doi/10.1145/3519023)
- [3] [Development of a real-time multiplayer game for the computer tablet](http://www.diva-portal.org/smash/record.jsf?pid=diva2%3A560260&dswid=-4964)
- [4] [Performing Lag Compensation in Unreal Engine 5](https://snapnet.dev/blog/performing-lag-compensation-in-unreal-engine-5/)
- [5] [VALORANT's foundation is Unreal Engine](https://www.unrealengine.com/en-US/tech-blog/valorant-s-foundation-is-unreal-engine)
- [6] [Overwatch Gameplay Architecture and Netcode](https://youtu.be/zrIY0eIyqmI)
- [7] [Latency Compensating Methods in Client/Server In-game Protocol Design and Optimization](https://developer.valvesoftware.com/wiki/Latency_Compensating_Methods_in_Client/Server_In-game_Protocol_Design_and_Optimization)
- [8] [On "Shot Around a Corner" in First-Person Shooter Games](https://ieeexplore.ieee.org/document/7991545)
- [9] [CoD Infinite Warfare & Modern Warfare Netcode Analysis](https://www.youtube.com/watch?v=oKE_eaTb1TU)
- [10] [Enhancing the Experience of Multiplayer Shooter Games via Advanced Lag Compensation](https://dl.acm.org/doi/10.1145/3204949.3204971)
- [11] [Cheat prevention & detection in online games](https://oulurepo.oulu.fi/handle/10024/11147)
- [12] [What Are Aimbots and How Do They Work](https://chatbotsjournal.com/what-are-aimbots-and-how-do-they-work-4936794e5453)