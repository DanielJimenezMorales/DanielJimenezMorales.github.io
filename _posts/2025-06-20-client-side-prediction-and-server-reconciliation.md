---
layout: post
title: Client-Side Prediction and Server Reconciliation
subtitle: All you need to know about challenges and techniques related to Client-Side prediction
tags: [client-side prediction, latency concealment, rollback, server reconciliation]
comments: true
readtime: true
author: Daniel Jiménez Morales
---

In fast-paced multiplayer games, specially shooters, in-game responsiveness is crucial. But it can be hard to achieve due to [networking issues](https://danieljimenezmorales.github.io/2023-07-06-challenges-and-issues-of-the-network/) (e.g. latency, jitter...), as they can introduce noticeable delays that break inmersion and hurt the overall experience. In this post, we'll explore client-side prediction, a key technique used in Client-Server architectures (where the server is authoritative) to keep gameplay smooth and input feel immediate.

## Types of client-side prediction algorithms
Client-side prediction techniques can vary depending on the type of entity being predicted and who controls it. For remote entities, those controlled by other players or server-side AI systems, developers often use a technique called extrapolation. This technique attempts to estimate a remote entity's future position based on its past movement data. However, extrapolation comes with its own set of challenges and deserves an entire separate discussion. In this case, we'll focus on the technique used for predicting the local player entity, that is the one controlled by the player's own inputs.

## The problem: Lack of responsiveness
Most fast-paced online games use a client-server architecture where the server is responsible for running all the game logic. Clients connect to the server to send inputs of the player and receive updates about the world state to render. The server receives these client inputs, runs the game logic with those inputs, updates its world state and sends the results back to each client.

Due to security concerns, the server acts as the authoritative host, which means that the information that clients receive from the server will be considered correct.

A lack of game responsiveness can be seen in this process. For example, if the Round Trip Time (*Round Trip Time (RTT) is the total time it takes for a message to go to the server and back*) between a client and the server is 60 ms, it means that since the moment the client sends its inputs to the server until it receives the updated state for its player and renders it, at least a delay equal to the RTT has passed. And this is assuming the best scenario possible, where there's no packet loss, no jitter or no processing delays in the server. In those cases the wait becomes even longer!

But let's go back to the best case scenario. A delay of 60ms is noticeable to the human eye. This becomes unacceptable for a fast-paced game as its gameplay experience can be heavily degraded. Players expect their actions, like moving or shooting, to be reflected immediately, otherwise the game will be perceived as unresponsive and frustrating.

To address this issue, developers implement the client-side prediction technique, allowing the game to simulate the expected result of the player's inputs immediately, without waiting for the server confirmation. Let's explore how this works and why it's so effective.

## What is Client-Side Prediction and how does it work?
Client-side prediction (also known as Input prediction) is a latency compensation technique that allows the client to simulate the outcome of the local player inputs inmediately, before receiving the official outcome from the server. The key idea here is to anticipate or "predict" the results of an input locally, so the player doesn't need to wait for a round-trip time to see their effects rendered on screen. As a consequence, a much smoother experience is created and responsiveness increases!

{: .box-note}
**Note:** As I mentioned at the beginning of this post, this latency compensation technique applies exclusively to the local player, which is the entity being controlled directly by the player inputs. If we want to compensate the latency for remote entities, we will instead use other techniques like interpolation or extrapolation.

Here's a simplified code snippet of how a client might handle input and server updates without prediction:
```cpp
// Runs at a fixed rate.
void OnClientTick(float32 elapsed_time)
{
	const InputState input = GetInputState();
	SendInputStateToServer(input);
}

// Called on demand.
OnPlayerStateReceivedFromServer(const PlayerState& state)
{
	ApplyPlayerState(state);
}
```
In this code, the client sends the player inputs generated for that tick to the server and waits until receiving the resulted player state. This creates a noticeable lag.

However, with client-side prediction the previous code snippet changes into this:
```cpp
// Runs at a fixed rate.
void OnClientTick(float32 elapsed_time)
{
	const InputState input = GetInputState();
	SendInputStateToServer(input);
	
	const PlayerState currentState = GetCurrentPlayerState();

	PlayerState resultedState;
	SimulatePlayerState(input, currentState, resultedState, elapsed_time);
	ApplyPlayerState(resultedState);
}

// Called on demand.
OnPlayerStateReceivedFromServer(const PlayerState& state)
{
	// We don't need to do anything here now
}
```
Now the client no longer waits for the server's state for updating the player. Instead, it simulates the result of its inputs locally, making a best guess of what the server will calculate, and updates the player immediately.

## Great! Prediction works, but it introduces a new problem...
This technique, as with any other type of prediction, comes with its trade-offs. While responsiveness improves significantly, it reduces the data consistency between the client and the server. This can lead to desynchronizations that, unless they're fixed, will become bigger and bigger making the game unplayable. Let's explore a few common scenarios where these discrepancies can arise:

### Desyncs due to non-deterministic code
Although the client can now predict its own state, this doesn't guarantee the result will always match the server's one. If the code used for prediction is different from the code used in the server for its simulation we could have problems.

To prevent this, the code used for simulation must be shared between client and server and be deterministic (*Deterministic code always produces the same output given the same input*). To achieve that, a simple solution is to make the code responsible for the simulation logic shared between client and server.

### Desyncs due to collisions
Imagine two players try to move to the same position at the same time. On the server, one player will be blocked by a collision with the other. However on the client, that player has no knowledge of the remote input and, as a consequence, simulates movement as if the space were empty, resulting in an incorrect prediction.

### Desyncs due to jitter or packet loss
Another cause of desyncs is packet loss. If an input sent from the client arrives late or simply doesn't arrive to the server, the server won't wait for it. Instead, it will reuse the last known input received from that client to continue simulating. If the lost input is different from the previous one reused by the server, the server will generate a different state that will diverge from the one predicted by the client.

## Reconciliation with the server
Once a desynchronization is detected the client must reconcile to get back in sync with the server.

### Checking for desynchronizations
To detect a desynchronization the client needs to compare the authoritative state received from the server with the predicted state for that same tick. If they differ, we apply a correction. However, since a few ticks may have passed betweeen the prediction and the arrival of the server state, we need to buffer these prediction states to be able to get back to them later for performing this comparison.

Let's structure this by using an ECS approach. We are going to create a component called `ClientSidePredictionComponent`, which stores past inputs and player prediction states.
```cpp
struct ClientSidePredictionComponent
{
	const uint32 PREDICTION_BUFFER_SIZE = 1024;

	// For saving the input state used for each tick
	InputState[PREDICTION_BUFFER_SIZE] inputStateBuffer;
	// For saving the final predicted state calculated for each tick
	PlayerState[PREDICTION_BUFFER_SIZE] playerStateBuffer;

	// To avoid comparing outdated states
	uint32 lastPlayerStateTickReceivedFromServer;
}
```

These buffers have a fixed size based on `PREDICTION_BUFFER_SIZE`. Also, since they are circular we won't need to clean old states as they will be overwritten by the new ones. However, we have to be careful and set a `PREDICTION_BUFFER_SIZE` big enough to avoid overwritting these states too early, specially with higher RTTs, otherwise we won't be able to fix any desynchronization.

Now that we have our prediction buffers created, let's modify our prediction code to store the data we will need later for the server's state comparison and reconciliation.
```cpp
// Runs at a fixed rate.
void OnClientTick(float32 elapsed_time)
{
	// Get and send input to server
	const uint32 currentTick = GetCurrentTick();
	const InputState input = GetInputState(currentTick);
	SendInputStateToServer(input);
	
	// Simulate input locally
	const PlayerState currentState = GetCurrentPlayerState(currentTick);
	PlayerState resultedState;
	SimulatePlayerState(input, currentState, resultedState, elapsed_time);

	// Stores input and resulted state in prediction buffer
	ClientSidePredictionComponent& predictionCmp = GetPredictionComponent();
	const uint32 predictionBufferIndex = currentTick % predictionCmp.PREDICTION_BUFFER_SIZE;
	predictionCmp.inputStateBuffer[predictionBufferIndex] = input;
	predictionCmp.playerStateBuffer[predictionBufferIndex] = resultedState;

	ApplyPlayerState(resultedState);
}
```

Now that we're storing prediction history, we can compare it with the authoritative server states once they arrive and detect desyncs.
```cpp
bool IsReconciliationNeeded(const ClientSidePredictionComponent& prediction_cmp, const PlayerState& state_from_server)
{
	bool result = false;

	// Check if the received state is newer than the last received
	if(state_from_server.tick > prediction_cmp.lastPlayerStateTickReceivedFromServer)
	{
		// Get the predicted state associated with the state from server tick
		const uint32 predictionBufferIndex = state_from_server.tick % prediction_cmp.PREDICTION_BUFFER_SIZE;
		const PlayerState& predictedState = prediction_cmp.playerStateBuffer[predictionBufferIndex];

		// Check that it has not already been overwritten. Otherwise, we won't be able to perform any comparison.
		if(predictedState.tick != state_from_server.tick)
		{
			LOG_WARN("Trying to compare against a predicted state that has been overwritten. Increase PREDICTION_BUFFER_SIZE or decrease RTT");
			return;
		}

		if(!AreSimulationStatesEqual(predictedState, state_from_server))
		{
			result = true;
		}
	}

	return result;
}

// Called on demand
OnPlayerStateReceivedFromServer(const PlayerState& state)
{
	ClientSidePredictionComponent& predictionCmp = GetPredictionComponent();	
	if(IsReconciliationNeeded(predictionCmp, state))
	{
		// Add reconciliation here... (We'll do this in the next step)

		// To avoid comparing again with a duplicated or out of order state
		predictionCmp.lastPlayerStateTickReceivedFromServer = state.tick;
	}
}
```

### The reconciliation algorithm
These is how the reconciliation process works:
1. Overwrite the predicted state with the authoritative state from the server.
2. Re-simulate all subsequent ticks using the inputs buffered in `ClientSidePredictionComponent` until reaching the current tick.
3. Apply the re-simulated state to the current tick to render the correction.

```cpp
void ReconciliateWithServer(ClientSidePredictionComponent& prediction_cmp, const PlayerState& state_from_server)
{
	// Overwrite the predicted state with the one received from server
	const uint32 slotIndexToReconciliate = state_from_server.tick % prediction_cmp.PREDICTION_BUFFER_SIZE;
	prediction_cmp.playerStateBuffer[ slotIndexToReconciliate ] = state_from_server;

	PlayerState currentPlayerState = state_from_server;
	uint32 currentTick = state_from_server.tick + 1;
	const uint32 lastTick = GetCurrentTick();

	// Re-simulate the entity logic with the corrected simulation state from server until reaching the current tick.
	while ( currentTick != lastTick )
	{
		const uint32 currentSlotIndex = currentTick % prediction_cmp.PREDICTION_BUFFER_SIZE;

		const InputState& input = prediction_cmp.inputStatesBuffer[ currentSlotIndex ];
		const float32 elapsedTime = prediction_cmp.elapsedTimeBuffer[ currentSlotIndex ];
		const PlayerState resultedPlayerState = SimulatePlayerState(input, currentState, resultedState, elapsed_time);

		prediction_cmp.playerStatesBuffer[ currentSlotIndex ] = resultedPlayerState;
		currentPlayerState = resultedPlayerState;

		++currentTick;
	}

	// Apply the current simulation state to the entity
	ApplyPlayerState( currentPlayerState );
}

// Called on demand
OnPlayerStateReceivedFromServer(const PlayerState& state)
{
	ClientSidePredictionComponent& predictionCmp = GetPredictionComponent();	
	if(IsReconciliationNeeded(predictionCmp, state))
	{
		ReconciliateWithServer(predictionCmp, state);
		predictionCmp.lastPlayerStateTickReceivedFromServer = state.tick;
	}
}
```

{: .box-note}
**Note:** During reconciliation with the server, the re-simulation process should only simulate those mandatory systems. This means that during the re-simulations, no VFX, sounds or any optional behaviour is triggered, as we won't be applying any of these states until we reach the current tick.

## The problem with Snap Server Reconciliation
In the reconciliation algorithm shown in the previous section, in case of discrepancies, the authoritative state received from the server is inmediately applied, overwriting the client's predicted state in a single simulation tick. While this guarantees accuracy and consistency with the server, it introduces certain drawbacks on the client's side, specially from a visual perspective.

### Teleportation effects
The most obvious issue of instant reconciliation is the sudden shift in the player’s position or orientation. If the server's state differs from what the client predicted, the player will be abruptly "teleported" from one location to another. Even if the correction is small this snapping effect is particularly visible and can feel unnatural, creating a visual artifact that players perceive as jittery or glitchy.

### Perceived loss of control
When these abrupt corrections happen frequently due to high latency, dropped packets or prediction mismatches, players may begin to feel like their inputs are not being properly respected. Although these corrections are only applied to the local client and remain invisible for the rest of players the impact on the user experience can be significant. The outcome is a loss of perceived control, where player inputs feel inconsistent or unreliable.

## Smoothing Server Reconciliation
To address the issues introduced by snap reconciliation, we can improve it by implementing a smooth reconciliation approach. Instead of applying the server correction instantly in a single simulation tick, we apply it gradually over time. This allows us to keep the client synchronized with the authoritative server while ensuring a smooth correction process without any visual artifacts.

### Dual object model: Ghost vs Interpolated
To implement the smooth correction, the client-side representation of the player is divided in two different objects:
1. **Ghost object:** This object contains all the gameplay and simulation data (position, velocity, ammo left, etc). It is updated on every fixed simulation tick by the client-side prediction code. When the client receives a player simulation state, in case it differs from the predicted state for that tick and needs to be reconciled, we apply a snap reconciliation directly to this object, using the algorithm discussed in the previous section.

2. **Interpolated object:** This object only contains rendering data (sprites, muzzle flashes, particle effects, etc) and it is responsible for rendering the player visuals on screen. This object is not part of the simulation logic. Instead, it follows the ghost object to mimic its position and orientation. It is updated on every variable frame update by smoothly interpolating toward the ghost object. This will create a seamless transition even when the ghost object experiences corrections.

| Aspect | Ghost Object | Interpolated object |
| -------- | ------- | ------- |
| Purpose | Simulation | Rendering |
| Contains | Simulation & gameplay data | Rendering & VFX data |
| Update rate | Fixed Simulation tick | Variable Frame Update |
| Applies server corrections | Yes (snap correction) | No (follows ghost via interpolation) |

{: .box-note}
**Note:** All the code snippets from the previous sections are still valid for this dual-object approach. However, now they will only be applied in the ghost object and not in the interpolated object. Since we already have the ghost object code, in this section we are going to focus on the interpolated object side.

Let's implement this step by step. First, we will add a component to the interpolated object to define how it follows the ghost.
```cpp
struct ClientPlayerInterpolatedObjectComponent
{
	// How smooth the interpolation is.
	// The bigger the smoothing factor the faster the interpolated object
	// will reach the ghost position and orientation.
	float32 smoothingFactor;

	// The maximum allowed distance between the ghost and the interpolated
	// object positions before we decide to snap it.
	float32 snapToGhostPositionDistanceThreshold;

	// The maximum allowed difference between the ghost and the interpolated
	// object orientation orientations before we decide to snap it.
	float32 snapToGhostOrientationThreshold;
}
```

{: .box-note}
**Note:** The position and orientation thresholds defined in the code snippet above are used to determine when to snap the interpolated object's transform to the ghost object. Snapping is preferred over smooth interpolation when the discrepancy between the predicted and actual states is too large. In such cases, continuing to interpolate could result in noticeable accuracy errors. Snapping helps maintain better synchronization between the client and server, even if it momentarily reduces visual smoothness.

This next function determines whether we should interpolate or snap to the ghost object's transform. We will use it in the next code snippets.
```cpp
bool IsSnapRequired(const Transform& ghost, const Transform& interpolated, const ClientPlayerInterpolatedObjectComponent& config)
{
	return (Distance(ghost.position, interpolated.position) >= config.snapToGhostPositionDistanceThreshold
		|| Abs(ghost.orientation - interpolated.orientation) >= config.snapToGhostOrientationThreshold)
}
```

Now here's the logic for the interpolated object to perform either a snap or a smooth interpolation based on `IsSnapRequired`:
```cpp
// This is executed on the variable frame update
void OnClientUpdate(float32 elapsed_time)
{
	// Grab necessary information first
	const ClientPlayerInterpolatedObjectComponent& interpolatedObjectCmp = GetClientPlayerInterpolatedObjectComponent();
	const Transform& ghostTransform = GetGhostTransform();	
	TransformComponent& interpolatedObjectTransform = GetInterpolatedObjectTransform();

	Vec2f finalPosition;
	float32 finalOrientation;

	// Check wether it should snap to ghost object or interpolate toward it.
	if(IsSnapRequired(ghostTransform, interpolatedObjectTransform, interpolatedObjectCmp))
	{
		// If the interpolated object is far enough from the ghost, snap it.
		finalPosition = ghostTransform.position;
		finalOrientation.ghostTransform.orientation;
	}
	else
	{
		// If the interpolated object is close enough to the ghost, keep interpolating.
		const float32 interpolationVelocity = interpolatedObjectCmp.smoothingFactor * elapsed_time;

		finalPosition.x = InterpolateFloat32(interpolatedObjectTransform.position.x, ghostTransform.position.x, interpolationVelocity);		
		finalPosition.y = InterpolateFloat32(interpolatedObjectTransform.position.y, ghostTransform.position.y, interpolationVelocity);
		finalOrientation = InterpolateFloat32(interpolatedObjectTransform.orientation, ghostTransform.orientation interpolationVelocity);
	}

	// Apply interpolation changes
	interpolatedObjectTransformCmp.position = finalPosition;
	interpolatedObjectTransformCmp.orientation = finalOrientation;
}
```

{: .box-note}
**Note:** The function `InterpolateFloat32` contains the logic for the interpolation algorithm. By the time of this post I implemented a Linear Interpolation, which is one of the most simple interpolation algorithms, but feel free to complicate it as much as you want.

## Prevent CPU spikes
Executing the server reconciliation process can be computationally expensive. When a correction is required, the client must run the entire simulation multiple times in the same tick, starting from the corrected tick until reaching the current tick. This can lead to performance issues, especially when the correction spans many ticks.

Imagine that the game runs at a fixed simulation tick rate of 60Hz and the RTT between the client and the server is equal to 80ms. To estimate the minimum number of re-simulations the client will need to run in case of having a correction, we can use the following formula:

![Minimum number of re-simulations formula](https://latex.codecogs.com/svg.image?&space;tickrate*RTT/1000)

{: .box-note}
**Note:** This formula estimates the *minimum* number of re-simulations. We are assuming ideal conditions, such as no server-side processing delay, zero network jitter and no packet loss. However, in real-world scenarios, the actual number will probably be higher.

Using our the exact numbers from our example we get:

`Min number of re-simulations = 60 * 80 / 1000 = 4.8.`

This means that every time the client finds a mismatch between its predicted state and the authoritative state from the server, it must re-simulate at least 5 ticks to reconcile. As a consequence a CPU spike might arise if the correction is fully applied in the same tick.

### Mitigation strategy: Spread re-simulations over multiple ticks
One possible solution is to limit the number of re-simulations executed each tick. Instead of processing all the re-simulations needed to reconcile in one tick, we can split the work over multiple ticks.

For example, if 6 ticks of re-simulation are required, we might choose to perform 2 reconciliations for the next three ticks. This solution introduces a small delay in reaching full synchronization, but it reduces the risk of having CPU spikes.

# Conclusion
Client-side prediction is a powerful technique thatg improves responsiveness, particularly for local player. However, it reduces the data consistency between the client and the server, introducing potential desynchronizations. Reconciliation techniques help restore this balance, but they come at a cost, namely, CPU overhead, additional complexity or potential correction artifacts.

Adjusting the different parameters of these systems is crucial to find the right balance for each game's specific needs.

To conclude, if you have questions about any aspect of this article, you want to contribute with new ideas or you just want to chat, please feel free to contact me.