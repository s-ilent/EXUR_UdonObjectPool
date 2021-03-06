# EXUR (EXclusive Use and Reusing objects) Manual

<!-- TOC depthfrom:2 depthto:3 orderedlist:false insertanchor:false -->

- [Elements](#elements)
    - [Basic structure](#basic-structure)
    - [EXUR Manager](#exur-manager)
    - [EXUR Handler](#exur-handler)
    - [User Program](#user-program)
- [API](#api)
    - [Methods](#methods)
    - [Handler simplified event API](#handler-simplified-event-api)
    - [Handler detailed event API](#handler-detailed-event-api)
    - [EXURReceiveEvent custom event API](#exurrecieveevent-custom-event-api)
    - [ManagerFailure details](#managerfailure-details)
- [Feature details](#feature-details)
    - [Tag feature](#tag-feature)
- [Internal design note](#internal-design-note)

<!-- /TOC -->

## Elements

### Basic structure 

![Objects and components structure](EXUR-object-component-structure.svg)

* You can add child GameObjects to Pooled GameObject.
* You can add other components to this structure.


### EXUR Manager
EXUR Manager manages pooled game objects.
It gathers child objects and treats as pooled objects.
To operate the pool User program calls EXUR Manager custom event (or as a method in U#).


### EXUR Handler
EXUR Handler operates one pooled object.
Target pooled object is a GameObject where the EXUR Handler is attached.
To operate each pooled object User program calls EXUR Handler custom event.

UdonBehaviour that hosts EXUR Handler must be the first UdonBehaviour component in the pooled GameObject.


### User Program
Library user prepares UserProgram if needed.

EXUR expects that has certain custom events and program variables.
Their name and type are defined by EXUR.
So you can make UserProgram with UdonGraph and UdonSharp.

These custom events and variables are optional.
You only need to implement what you need because it is just ignored if it doesn't exist.
You just implement you want to know the event happened.

UdonBehaviour components placed in Pooled GameObject are treated as User Program.
Also UdonBehaviours attached to children GameObject can be User Program.
(see `IncludeChildrenToSendEvent` option)


## API

### Methods

#### `Iwsd.EXUR.Manager.AcquireObject()`

Request to acquire one object from the pool.

#### `Iwsd.EXUR.Manager.AcquireObjectWithTag(string tag)`

Argument program variable: `string AcquireObjectWithTag_tag`

Request to acquire one object that has specified tag from the pool.
If no object has specified tag, Manager assigns the tag to free object (if available) and passes it to User Program.

To use this method, pooled GameObject must be properly setup. See [Tag feature](#tag-feature) for details.


#### `Iwsd.EXUR.Manager.AcquireObjectForEachPlayer()`

Request to acquire one object for each player.

It uses Player `displayName` as a tag to select object.


#### `Iwsd.EXUR.Handler.ReleaseObject()`


### Handler simplified event API

User Programs that placed on each pooled GameObject can react for these events if you implement these custom event.

| Category       | Custom-Event Name        | Comment                                        |
|----------------|--------------------------|------------------------------------------------|
| Simplified API |                          |                                                |
|                | EXUR_Reinitialize        | Called when you can use the object             |
|                | EXUR_Finalize            | Called when you should end to use and clean up |
|                | EXUR_OtherPlayerAcquired | Called when other player started to use        |
|                | EXUR_OtherPlayerReleased | Called when other player stopped to use        |


![Simplified state diagram](EXUR-simplified-state-diagram.svg)



### Handler detailed event API

If you want to know more detailed state transition, use these custom event.

You can use simplified and detailed events together.
(actually simplified events are "combined" aliases.)


| Category                   | Custom-Event Name                 | Comment                                    | Simplified to |
|----------------------------|-----------------------------------|--------------------------------------------|---------------|
| Result of initialization   |                                   | Spontaneous transition after Start         |               |
|                            | InitializedToOwn                  |                                            |               |
|                            | InitializedToIdle                 |                                            |               |
|                            | InitializedToUsing                |                                            | 3             |
| Transition while not owned |                                   | Another players start and stop using       |               |
|                            | StartedToUseByOthers              |                                            | 3             |
|                            | StoppedUsingByOthers              |                                            | 4             |
| Result of start request    |                                   | Happens after acquire request               |               |
|                            | FailedToUseByTimeout              |                                            |               |
|                            | FailedToUseByRaceCondition        |                                            | 3             |
|                            | EnterUsingFromWaiting             |                                            | 1             |
|                            | EnterUsingFromOwn                 |                                            | 1             |
| Result of stop request     |                                   | Happens after release request             |               |
|                            | ExitUsingByRequest                |                                            | 2             |
|                            | AttemptToReleaseNotOwnObjectError |                                            |               |
| Might happen while own     |                                   |                                            |               |
|                            | LostOwnershipOnUsing              |                                            | 2, 3          |
|                            | LostOwnershipOnIdle               |                                            |               |
| Retrieve by Master         |                                   | (InitializedToOwn is not categorized here) |               |
|                            | RetrievedAfterOwnerLeftWhileUsing |                                            |               |
|                            | RetrievedAfterOwnerLeftWhileIdle  |                                            |               |


Simplified column value

| in above table | translate to             |
|---------------:|--------------------------|
|              1 | EXUR_Reinitialize        |
|              2 | EXUR_Finalize            |
|              3 | EXUR_OtherPlayerAcquired |
|              4 | EXUR_OtherPlayerReleased |


![Detailed state diagram](EXUR-detailed-state-diagram.svg)


### EXUR_ReceiveEvent custom event API

Listener interface:

        [HideInInspector] public UdonBehaviour EXUR_EventSource;
        [HideInInspector] public string EXUR_EventName;
        [HideInInspector] public string EXUR_EventAdditionalInfo;
        public void EXUR_ReceiveEvent()
        {
            // your code 
        }

Set this listener UdonBehaviour to `Iwsd.EXUR.Manager.EventListener`

(TODO: describe above definition more "politely")


| EXUR_EventSource | EXUR_EventName            | EXUR_EventAdditionalInfo   | Comment                               |
|------------------|---------------------------|----------------------------|---------------------------------------|
| EXUR.Handler     | (see "Handler event API") | -                          | Aggregate and propagate Handler event |
| -                | InUseBySelf               | -                          | Already in use by myself              |
| -                | InUseByOthers             | -                          | Already in use by others              |
| -                | NoFreeObject              | -                          | No more free object                   |
| -                | ManagerFailure            | human readable description | program error etc.                    |

"-" means don't care. Actually it is null.

### ManagerFailure details

This is internal specification so it will be changed.

| Description header | Possible case                                |
|--------------------|----------------------------------------------|
| InternalError:     |                                              |
|                    | Multiple objects have identical tag          |
|                    | localTagBuffer is not clear though it's free |
| UserProgramError:  |                                              |
|                    | Specified tag is null                        |
|                    | Specified tag is empty                       |
|                    | No sibling UdonBehaviour (for Tag)           |
|                    | Does not have EXUR_Tag variable variable     |
|                    | Does not have EXUR_LastUsedTime variable     |


## Feature details

(Draft. TODO create sections and refine.)

* ownership
    * ownership is controlled on "Pooled GameObject"
    * Delay for safer sync variable writing
    * handler will be pass to user program by manager event after ownership obtained.
* IncludeChildrenToSendEvent handler option
    * to be exact "children" means "descendant" (like `GetComponentsInChildren()`)
    * note: child object ownership is not controlled by EXUR
* DeactivateWhenIdle handler option
    * pooled object when it becomes idle.
* additional note for features
    * synced variable free
        * to avoid network load
    * Try to use owned object if possible
* DebugText
    * optional
    * to view internal log
* Assignment algorithm described
    * try to use owned object first for speed
        * So previous user continues to holds ownership of released object until other client requires.
    * random index to avoid race condition
    * Least recently used (LRU) algorithm for tag

### Tag feature

(Draft. TODO Refine me.)

* tag is a string
* tag is used to identify object. It's like temporarily name.
* string.Empty means "not used". it can not be used usual value. (null is also)
* First sibling UdonBehavior must have certain variables
    * First sibling (second in the GameObject) UdonBehaviour must have program variables:
        * `[UdonSynced] string EXUR_Tag`
        * `[UdonSynced] int EXUR_LastUsedTime`
* EXUR library try to preserve and use identical object if possible 
* If no room for new acquired object, old object will be purge with Least recently used (LRU) algorithm
    * User can update EXUR_LastUsedTime. (Of course it should be done on owner client.)
* interface note: how to passing argument

## Internal design note

**Tag needs to be synced variable**

* so it implemented on another UdonBehavior.
* to let core (Handler) be synced variable free.
* Before obtaining ownership, it can not use synced variable. So it stores to local buffer.
It's uneasy(?) problem where to hold the local buffer.
It is possible to store it to user (sibling) Udon Program.
But we choose Handler because we want to be "clean" user program as much as possible.


**method name prefix policy**

* (i.e. custom event name) 
* to avoid conflict on user program.


**original concept**

![Original concept](EXUR-concept-state-diagram.svg)
