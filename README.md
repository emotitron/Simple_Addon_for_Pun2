# Simple Network Sync for PUN2
Author Contact:  
davincarten@gmail.com

Current Pre-Beta:  
https://github.com/emotitron/SNS_PUN2/releases

PhotonEngine Discord:  
https://discord.gg/egaRfd8  
Author's handle is **emotitron**.

Tutorial:  
https://docs.google.com/document/d/1moPBIt8cNe-h1uG01pvaOZQvrIjZfSBfDmVa793X6AQ/edit#

----
Many networking libraries available to Unity developers focus primarily on synchronization of fields and replicating events from one client to another. This often comes in the form of Syncvars and RPCs (Remote Procedure Calls). While this can lead to quick successes, it often becomes very entangled and cumbersome as the race conditions, cross dependencies and lack of deterministic order start to compound complexity. **In contrast, Simple Network Sync (SNS) extends PUN2 to operate on a simulation-based tick timing system that uses circular buffers**. 

**SNS uses mixed-authority snapshot interpolation.** PUN2 is a relay environment, so it is a slightly different architecture than Server Authority, which systems like Photon Bolt employ. As there is no central state authority, authority is distributed. Players typically are the authority over objects they are in control of.

**For example:** If a player shoots another, the shooter notifies the hit player’s health system of the hit. The hit player having authority over their own health applies the damage in response to the hit indication from the shooter. That hit player will also accordingly apply death, stun, etc. to themselves. It should be noted that this is inherently NOT cheat resistant. The PUN relay model is more about quick and easy dev, and is not the ideal library for highly competitive PVP games. Bolt or Quantum would be more suitable platforms for such games.

**The SNS system focuses on using Unreliable UDP with Keyframe and Delta frames, similar to how video streams work.** Rather than using Acks to negotiate eventual consistency between clients and a server (which would become very cumbersome in a relay environment where every client would need to negotiate reliability with every other client), SNS uses keyframes to achieve eventual consistency. Lost or late packets that produce disagreement in state between the owner and other clients will eventually resolve when forced updates (keyframes or changes) arrive.

# Simple Network Sync (SNS) Highlights:
* [Base set of “Just Works” Components and Interfaces](#Components)
* [Simulation-based tick system with discrete deferred timings](#Simulation)
* [Numbered State Buffers (Circular Buffer)](#Buffers)
* [Serialization to a unified bitpacked byte\[\] array](#Serialization)
* [Syncvars tied to the core simulation tick timing system](#Syncvars)
## <a name="Components"></a>Base set of “Just Works” Components and Interfaces
The first and most obvious benefit of the new extension library is the quantity of code that exists in the form of interoperational components. Without writing a single line of code you can have a networked scene that utilizes any combination of synced:
* Transforms
* Animators
* Hitscans & Projectiles
* Health/Vitals and automation of UI
* State (Attached, Visible, Thrown/Dropped, etc.)
* Visibility / Rigidbody changes in response to State changes
* Mounting to other objects
* Damage/Healing/Buff/Debuff zones
* Inventory and Inventory Items
* Health / Energy Pickups

These components can be derived and extended, or the base interfaces can be used to create components from scratch that interact with these systems.

## <a name="Simulation"></a>Simulation-based tick system with discrete deferred timings
The NetMaster singleton acts as a central Update Manager, so the entire system is polled every tick to produce a series of tick-based callbacks that very tightly control execution of PreSimulation, PostSimulaton, State Capture, Serialization, Deserialization,  State Application, Interpolation and Extrapolation code.  This eliminates race conditions and keeps all objects tightly synchronized to themselves and other objects with the same owner. As such, many things become largely deterministic in nature. For example, when a player triggers a hitscan, the shot will be queued. On the next tick the players transform, animator and weapon state will be captured and serialized in a predetermined order. The shot then accurately replicated on other clients by applying  the transform and animator state, then reproducing the hitscan in the same order they were captured on the owner.

## <a name="Buffers"></a>Numbered State buffers (Circular Buffer)
The core FixedUpdate based timing system captures the state of owned objects on a regular timing. These are stored in a ring array on the owner and are serialized to all clients, where they are then deserialized into ring arrays. These numbered states allow for very smooth interpolation/extrapolation of states even with packet loss and maintain synchronicity between all objects that share an owner.

## <a name="Serialization"></a>Serialization writes to a unified bitpacked byte[] array
This allows for extremely high levels of data compression, as well as allowing components to employ inline serialization logic. Writes start with a single timing from the NetMaster immediately after each Simulation (PostPhysX), and a single byte[] array is passed to all of the state producing NetObjects, which in turn pass it to every child SyncObject and PackObject (Synvars) component for serialization in a deterministic order. This produces one highly compressed and tightly ordered byte[], rather than having scattered objects calling Write() methods adhoc.

## <a name="Syncvars"></a>Syncvars tied to the core simulation tick timing system
PackObjects allow for fields to be tagged with attributes that will automatically generate code that syncs values from Owner to All on the same deferred timings as NetObjects/SyncObjects. This means you can simply synchronize fields rather than having to write your own SyncObjects. Syncvars have built-in compression options, and advanced users can write their own compression methods as well.

