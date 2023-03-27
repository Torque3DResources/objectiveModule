# objectiveModule
A module for the Torque3D Game Engine that implements an objectives tracking system

# General Objective/event system
## Expected Knowledge

### General background
https://en.wikipedia.org/wiki/Namespace
https://en.wikipedia.org/wiki/Callback_(computer_programming)
https://en.wikipedia.org/wiki/Finite-state_machine

### T3D specific 
(todo: change the below to https://docs.torque3d.org/general/readme links)

```
SimObject.isMethod(%method)
```
> Determines if a class/namespace method exists
> method:Name of the function to search for
> return True if the method exists, false if not

``` 
SimObject.call( %method, %arg1, %arg2, ect... )
```
> Calls a function
> method: Name of method to call.
> args: Zero or more arguments for the method.
> return: The result of the method call (as a string)

```
SimObject.getFieldValue(%fieldName, %index )
```
> Gets an object-variable value as a string
> fieldName: The name of the field.  If it includes a field index, the index is parsed out.
> index: Optional parameter to specify the index of an array field separately.
> return: The value of the given field or "" if undefined.
   
## Namespaces, inheritance, and callbacks
T3D uses a unique, flexible namespace archetecture for scripting using the following precidence rules
First, it looks for a specific objectName::callback
Second it will attempt to find an %object.class callback
Third it will attempt to find an %object.superClass callback
Failing that it will attempt to go through the source inheritance chain to find a more general namespace method definition

# Classes
## ObjectiveInfo
```
new scriptobject()
{
    class= "ObjectiveInfo"; //namespace
    name = ""; //Name of Objective. the object name
    displayName = ""; //Objective title
    description = ''; //Objective writeup
    locationMarker = ""; //.mis item associated with Objective
    achievement = ""; //metadata for achievements
    nextObjective = ""; //Objective to automatically trigger starting on sucess
    failObjective = ""; //Objective to automatically trigger starting on failure
    
    //conditions
    repeatable = false;
    timeToComplete = 0; //countdown in minutes
    collectionItem = ""; //item name required for completion
    collectionCount = ""; //item count required for completion
    
    //method pointers. 
    //each attempts to find one in the form of ObjectiveName::callback(). 
    //executed by doObjectiveCallback before callonmodules    
    onStart = ""; 
    onPause = ""; 
    onResume = ""; 
    onComplete = ""; 
};
```

# Methods

## General Bootup
```
createObjectiveManager()
```
> Creates a pair of arrayobjects for ".Objective" and related ".item" tracking respectively
>   referenced via ObjectiveModule.Objectives and ObjectiveModule.items

```
ObjectiveManager::initState()
```
> Basic example/fallback using a general mainObjective Objective and setting that to the Objective you're on   

```     
ObjectiveModule::onLoadMap()
```
> Executed post map-load, used in conjunction with fileio onSaveLoaded module callback to load objective state
    if it fails, uses initstate instead
    
## Objective module support 
```
doObjectiveCallback(%ObjectiveTag, %act)
```
> Generates callbacks of the form ObjectiveTag::on<State>()
> also executes any generics via callOnModules module::onObjective<State>("ObjectiveTag")

```
ObjectiveStateChanged()
```
> Client notification of Objective state changes via updateObjectiveForClient for all conections
            
## Objective status polling
```
onObjective(%ObjectiveTag)
```
> Objective is checked to see if it is either the current active or paused

```
isObjectiveActive(%ObjectiveTag)
```
> Objective is checked to see if it is the currently focused active one  

```
ObjectiveStatus(%ObjectiveTag)
```
> Polls the $ObjectiveManager.Objectives array for the state of a Objective
> current list: active, pause, complete, failed

```
curObjective() 
```
> Returns the tag of the currently active Objective

## Objective state setting
All the below call doObjectiveCallback(%ObjectiveTag, %act), and ObjectiveStateChanged() 

```   
startObjective(%ObjectiveTag)
```  
> Begins a Objective if it's not listed or is repeatable
> sets previous Objective to paused
> executes doObjectiveCallback
> starts any countdowns

```
pauseObjective(%ObjectiveTag)
```
> pauses a given Objective

```
resumeObjective(%ObjectiveTag)
```
> pauses the current Objective, and sets another to active if it exists

```
completeObjective(%ObjectiveTag)
```
> completes a Objective, if nextObjective exists, executes startObjective(%ObjectiveTag.nextObjective);
> then resumeObjective on the last non-completed Objective added to the stack

```
failObjective(%ObjectiveTag)
```
> fails a Objective, if failObjective exists, executes startObjective(%ObjectiveTag.failObjective);
> then resumeObjective on the last non-completed Objective added to the stack

## Objective state reporting

```
ObjectiveTrackerGUI
```
> gui element that extends playgui with objective status updates

```
objectiveModule::Playgui_onWake()
```
> callback that bolts on the ObjectiveTrackerGUI when playgui is awakened

```
clientCMDupdateObjectiveClock(%time)
```
> client PRC reciever to set the objective clock counting down or hidden

```
ObjectiveTrackerGUI::refresh(%this)
```
> updates the short form list of current objectives with tagged colors based on completion state

## Editor tooling
```
ObjectiveEditorGUI
```
> gui form that allows the user to edit ObjectiveInfo-classed objects

```
ObjectiveEditorGui::populateObjectivesList(%this)
```
> scans for ObjectiveInfo-classed objects and presents them in the ObjectiveEditorGUi list

```
ObjectiveEditorObjectiveList::onSelect(%this, %id, %text)
```
> Takes the selected ObjectiveInfo object and has the ObjectiveInspector in the EditorGUI inspect it for editing
> ObjectiveInfo::onInspect(%this, %inspector)
> callback from the ObjectiveInspector object, lets the ObjectiveInfo object manipulate inspector groups and fields for editing

```
GuiInspectorGroup::buildObjectiveListField(%this, %fieldName, %fieldLabel, %fieldDesc, %fieldDefaultVal, %fieldDataVals, %callbackName, %ownerObj)
```
> callback from the Inspector/InspectorGroup for the special field type "ObjectiveList". Creates a dropdown menu with a list of objectives for ObjectiveInfo object fields

# Module hooks

## fileIO module support
```
Objective::insertSaveData(%this)
```
> saves off the $ObjectiveManager.Objectives arrayobject in the form (Objective, status) under the <Objective> subsection

```
Objective::parseSaveData(%this)
```
> retrieves Objective status arrayobject

```
Objective::onSaveLoaded(%this)
```
> executes Objective callbacks for the various states to trigger any events/spawns et al that should have occured
> resumeObjective() the last active Objective
            
## Inventory module support
```
Objective::onSetInventory(%playerInst, %itemData, %currentAmount)
```
> hooks into the inventory module to sync tracking of Objective-relevant items to $ObjectiveManager.items
> completes an Objectives who's .collectionCount >= %currentAmount, including paused ones

```
Objective::onClearInventory(%playerInst)
```
> also resets the $ObjectiveManager.items

```
ObjectiveModule::onLoadMap()
```
> referenced above in general bootup