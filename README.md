# Source Engine Tutorials: Serverside Ragdoll Decal Persistence

A repo for, and tutorial for, enabling decals on NPCs to persist after death onto their respective serverside ragdolls.  
It also fixes a bug with serverside ragdolls where impact decals won't appear from an NPC's final fatal injury.  
This is designed to be used with mods using Source SDK 2013 Single-player, and is easy to implement into mods forked from Mapbase.  
It should also (hopefully) be easy to implement into mods using other versions of Source as well, e.g. older or Alien Swarm branch.  

To make before/after comparison easier, the initial commit in this repo has the files unchanged from their standard SDK 2013 code.

Created by Hxdce.
****
****


## Known Issues

Presently, this has some small issues with Half-Life 2's Zombie NPCs:

* Not all of their death states will trigger serverside ragdolls spawning. -- **Fixed!** Will be included in a future update.
* If the zombie's death also kills their headcrab, the headcrab will detach, fall through the level geometry, then disappear. -- **Todo!**

Fixes are in development for these problems.
****
****


## How To Use

**If your mod simply uses Source SDK 2013, and doesn't alter the following files:**
* `c_baseentity.h`
* `c_baseentity.cpp`
* `baseentity_shared.h`
* `physics_prop_ragdoll.cpp`

Then the repo contents can simply be pasted into your mod's parent directory. (I.e. `./your-mod-name/`)  
After this, compile as normal. An easy way too see it in action is to set the convar `ai_force_serverside_ragdolls` to 1.

--  

**If your mod doesn't use Source SDK 2013, and/or modifies the aforementioned files:**

Follow the steps below with your mod's corresponding files.

--  
**Step 1**  
In `baseentity_shared.h`, add this after line 251:
```cpp
#define BASEENTITY_MSG_SNATCH_MODEL_INSTANCE 64
```

--  
**Step 2**  
In `c_baseentity.h`, in the class declaration for `C_BaseEntity`, add the following member variables and functions:
```cpp
bool m_bUseRagdollModelInstance = false;
ModelInstanceHandle_t m_RagdollModelInstance = MODEL_INSTANCE_INVALID;

virtual ModelInstanceHandle_t GetDecalModelInstance();
```

--  
**Step 3**  
In `c_baseentity.cpp`, in the member function `C_BaseEntity::ReceiveMessage(...)`, add this after the case for `BASEENTITY_MSG_REMOVE_DECALS`:
```cpp
case BASEENTITY_MSG_SNATCH_MODEL_INSTANCE:
    // Model instance snatching from input to this entity.
    int eIndex = msg.ReadLong();
    C_BaseEntity* c = ClientEntityList().GetEnt(eIndex);
	if (c) {
		c->SnatchModelInstance(this);
		// This will redirect any future impact decals to the ragdoll, 
		// allowing the impact decals from a killing shot to appear on it as well.
		c->m_bUseRagdollModelInstance = true;
		c->m_RagdollModelInstance = m_ModelInstance;
	}
	else {
		DevMsg("Tried to snatch model instance with nonexistent entity!\n");
	}
    break;
```
Add the following member function definition:
```cpp
ModelInstanceHandle_t C_BaseEntity::GetDecalModelInstance() {
	ModelInstanceHandle_t output = m_ModelInstance;
	if (m_bUseRagdollModelInstance && m_RagdollModelInstance != MODEL_INSTANCE_INVALID) {
		// Use our ragdoll model instance so any decals appear on it instead.
		output = m_RagdollModelInstance;
	}
	return output;
}
```
In the definition for `C_BaseEntity::AddStudioDecal(...)`, do as follows:
```cpp
if (doTrace && (GetSolid() == SOLID_VPHYSICS) && !tr.startsolid && !tr.allsolid)
{
	// Choose a more accurate normal direction
	// Also, since we have more accurate info, we can avoid pokethru
	Vector temp;
	VectorSubtract( tr.endpos, tr.plane.normal, temp );
	Ray_t betterRay;
	betterRay.Init( tr.endpos, temp );
// New code:
	ModelInstanceHandle_t i = GetDecalModelInstance();
	modelrender->AddDecal(i, betterRay, up, decalIndex, GetStudioBody(), true, maxLODToDecal);
// Old code:
	//modelrender->AddDecal( m_ModelInstance, betterRay, up, decalIndex, GetStudioBody(), true, maxLODToDecal );
}
else
{
// New code:
	ModelInstanceHandle_t i = GetDecalModelInstance();
	modelrender->AddDecal(i, ray, up, decalIndex, GetStudioBody(), false, maxLODToDecal);
// Old code:
	//modelrender->AddDecal( m_ModelInstance, ray, up, decalIndex, GetStudioBody(), false, maxLODToDecal );
}
```

Then, in the definition for `C_BaseEntity::AddColoredStudioDecal(...)`, do as follows:
```cpp
if (doTrace && (GetSolid() == SOLID_VPHYSICS) && !tr.startsolid && !tr.allsolid)
{
	// Choose a more accurate normal direction
	// Also, since we have more accurate info, we can avoid pokethru
	Vector temp;
	VectorSubtract( tr.endpos, tr.plane.normal, temp );
	Ray_t betterRay;
	betterRay.Init( tr.endpos, temp );
// New code:
	ModelInstanceHandle_t i = GetDecalModelInstance();
	modelrender->AddColoredDecal(i, betterRay, up, decalIndex, GetStudioBody(), cColor, true, maxLODToDecal);
// Old code:
	//modelrender->AddColoredDecal( m_ModelInstance, betterRay, up, decalIndex, GetStudioBody(), cColor, true, maxLODToDecal );
}
else
{
// New code:
	ModelInstanceHandle_t i = GetDecalModelInstance();
	modelrender->AddColoredDecal(i, ray, up, decalIndex, GetStudioBody(), cColor, false, maxLODToDecal);
// Old code:
	//modelrender->AddColoredDecal( m_ModelInstance, ray, up, decalIndex, GetStudioBody(), cColor, false, maxLODToDecal );
}
```


--  
**Step 4**  
In `physics_prop_ragdoll.cpp`, in the function CreateServerRagdollSubmodel, add the following before the return statement:
```cpp
// For ragdoll cleanup.
pRagdoll->AddSpawnFlags(SF_RAGDOLLPROP_USE_LRU_RETIREMENT);
s_RagdollLRU.MoveToTopOfLRU(pRagdoll);
```
Finally, in the function CreateServerRagdoll, add the following before the return statement:
```cpp
// Snatch the model instance from the NPC to its ragdoll to copy its impact decals,
// and redirect any further impact decals to this ragdoll.
EntityMessageBegin(pRagdoll);
	WRITE_BYTE(BASEENTITY_MSG_SNATCH_MODEL_INSTANCE);
	WRITE_LONG(pAnimating->entindex());
MessageEnd();
```

After this, compile as normal. An easy way too see it in action is to set the convar `ai_force_serverside_ragdolls` to 1.
