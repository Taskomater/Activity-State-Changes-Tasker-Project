# Activity State Changes

## Export Info:
**Tasker Version:** `5.9`  
**Timestamp:** `2020-01-09 06.12.33`  



## Profile Names:
**Count:** `5`

- *`ActivityTrigger Activity Start Monitor`*
- *`Custom Activity Start Monitor`*
- *`ActivityManager Activity Config Change Monitor`*
- *`Activity State Change Controller Command Monitor`*
- *`Reset Activity State Change Variables On Monitor Start`*



## Scene Names:
**Count:** `0`




## Task Names:
**Count:** `9`

- *`Activity State Change Relay`*
- *`Activity State Change Controller`*
- *`Reset Activity State Change Variables`*
- *`com.whatsapp Activity State Change Responder`*
- *`com.fstop.photo Activity State Change Responder`*
- *`com.android.chrome Activity State Change Responder`*
- *`com.google.android.youtube Activity State Change Responder`*
- *`Fullscreen Activity Transition Screen Brightness Controller`*
- *`Anonymous (860)`*



## Profiles Info:

**#:** `1`  
**Name:** `ActivityTrigger Activity Start Monitor`  
**ID:** `855`  
**Entry Task:** `Activity State Change Relay`  
##


**#:** `2`  
**Name:** `Custom Activity Start Monitor`  
**ID:** `863`  
**Entry Task:** `Activity State Change Relay`  
##


**#:** `3`  
**Name:** `ActivityManager Activity Config Change Monitor`  
**ID:** `841`  
**Entry Task:** `Activity State Change Relay`  
##


**#:** `4`  
**Name:** `Activity State Change Controller Command Monitor`  
**ID:** `859`  
**Entry Task:** `Anonymous (860)`  
##


**#:** `5`  
**Name:** `Reset Activity State Change Variables On Monitor Start`  
**ID:** `847`  
**Entry Task:** `Reset Activity State Change Variables`  
##




## Tasks Info:

**#:** `1`  
**Name:** `Activity State Change Relay`  
**ID:** `839`  
**Collision Handling:** `Run Both Together`  
**Keep Device Awake:** `false`  
**Help:**
```
A task that should be called whenever an activity is started or resumed again or the activity's config changes like entering or exiting fullscreen mode. Ideally a "Logcat Entry" Profile Event should call this task so that the %lc_text variable can be parsed.

This task should basically find the currently opened package and activity and set it to %current_package_and_activity in the format "package_name/activity_name" and also set %activity_state_change to "activity_start" or "activity_config_change". Both variables are set to "%ActivityStateChangeControllerCommand" separated by a newline so that the "Activity State Change Controller Command Monitor" Profile can be triggered and can pass it to "Activity State Change Controller" Task to further process the activity state change and call any other required tasks for respected packages and activities. The "Activity State Change Command Monitor" Profile will also run the command tasks in order with its own queue. If new command tasks were run with "%priority - 1", then different instances of the same tasks would have run in parallel or might not run at all depending on priorities of other tasks running in tasker. If incrementally decreasing priorities were used, then other tasks in tasker would not have run if base priority was too high. A custom queue design could have been used but it's not needed since we don't care about return values of command tasks. This design is used because if the Logcat entry passed to this task does not contain the package and activity names and the Tasker "GetCurrentAppAndActivity" function or dumpsys command is used to find them, then in that case if processing an entry fully takes too long then by the time the turn comes for queued tasks, the package and/or activity might have already changed for which those Logcat entries were generated for resulting in false activity transitions being calculated. It is still best to find Logcat entries for the device and use them instead of other ways because false activity transitions will still occur in cases when this task is slow or queued because of higher priority tasks running in tasker and packages and/or activities have already changed.

By default this task is started by "ActivityTrigger Activity Start Monitor", "Custom Activity Start Monitor" and "ActivityManager Activity Config Change Monitor" Profiles. Note that either "ActivityTrigger Activity Start Monitor" or "Custom Activity Start Monitor" must be activated at the same time. If they are both activated, then this can cause duplication and result in fake transitions.

Check Activity State Changes Project docs to get more info on how this task should be called and what it should do.


This task does not take any parameters or return anything.
```
##


**#:** `2`  
**Name:** `Activity State Change Controller`  
**ID:** `858`  
**Collision Handling:** `Run Both Together`  
**Keep Device Awake:** `false`  
**Help:**
```
A task is called by the "Activity State Change Controller Command Monitor" Profile when the %ActivityStateChangeControllerCommand variable is set.

This task first checks if the current_package_and_activity passed is a valid package and activity name in the format "package_name/activity_name". If it is then it checks if the current activity is in fullscreen mode with status bars hidden either by using the dumpsys command's "mTopIsFullscreen" value if root mode is enabled or from the last "*StatusBar: setSystemUiVisibility" logcat entry's "newVal" bit flags. Then it sets the current_activity_state depending on activity_state_change and fullscreen mode. Then it checks if the a "package_name Activity State Change Responder" Task exists in the Tasker config for the current_package_and_activity or previous_package_and_activity. If either of those exists, then it calculates the previous_activity_task_activity_transition_par and current_activity_task_activity_transition_par depending on activity transitions in the transitions table. Then it calls the "previous_package Activity State Change Responder" Task if it exists with the previous_activity as %par1 and previous_activity_task_activity_transition_par as %par2. Then it calls the "current_package Activity State Change Responder" Task if it exists with the current_activity as %par1 and current_activity_task_activity_transition_par as %par2. Those tasks may respond appropriately to activity transitions but must not perform long running operations since this task will not finish until the called tasks are finished to maintain order and any queued tasks for this task will also be in waiting. Any long running operations that do not require ordered execution can be run inside the called tasks in additional tasks with "%priority - 1" so that the called tasks can return before the additional tasks finish.

Check Activity State Changes Project docs to get more info on how this task is called and what it does.


Input %par1:
"
current_package_and_activity
activity_state_change
"


This task does not return anything.
```
##


**#:** `3`  
**Name:** `Reset Activity State Change Variables`  
**ID:** `852`  
**Collision Handling:** `Abort New Task`  
**Keep Device Awake:** `false`  
**Help:**
```
A task that resets Activity States Changes Project variables.

This is called by the "Reset Activity State Change Variables On Monitor Start" Profile when Tasker Monitor is started. It should ideally also be manually called when the "* Activity Start Monitor" and "* Activity Config Change Monitor" tasks are enabled/disabled. Resetting variables is required to prevent false activity transitions if activity states were stopped from being monitor either by tasker being killed or manually by the user.


This task does not take any parameters or return anything.
```
##


**#:** `4`  
**Name:** `com.whatsapp Activity State Change Responder`  
**ID:** `853`  
**Collision Handling:** `Run Both Together`  
**Keep Device Awake:** `false`  
**Help:**
```
A task that responds to activity state changes for all the activities of the "com.whatsapp" package and is called by the "Activity State Change Controller" Task.

Check Activity State Changes Project docs to get more info on how this task is called and what it should do.


Input %par1:
"
activity
"

Input %par2:
"
activity_transistion
"

This task does not return anything.
```
##


**#:** `5`  
**Name:** `com.fstop.photo Activity State Change Responder`  
**ID:** `854`  
**Collision Handling:** `Run Both Together`  
**Keep Device Awake:** `false`  
**Help:**
```
A task that responds to activity state changes for all the activities of the "com.fstop.photo" package and is called by the "Activity State Change Controller" Task.

Check Activity State Changes Project docs to get more info on how this task is called and what it should do.


Input %par1:
"
activity
"

Input %par2:
"
activity_transistion
"

This task does not return anything.
```
##


**#:** `6`  
**Name:** `com.android.chrome Activity State Change Responder`  
**ID:** `856`  
**Collision Handling:** `Run Both Together`  
**Keep Device Awake:** `false`  
**Help:**
```
A task that responds to activity state changes for all the activities of the "com.android.chrome" package and is called by the "Activity State Change Controller" Task.

Check Activity State Changes Project docs to get more info on how this task is called and what it should do.


Input %par1:
"
activity
"

Input %par2:
"
activity_transistion
"

This task does not return anything.
```
##


**#:** `7`  
**Name:** `com.google.android.youtube Activity State Change Responder`  
**ID:** `857`  
**Collision Handling:** `Run Both Together`  
**Keep Device Awake:** `false`  
**Help:**
```
A task that responds to activity state changes for all the activities of the "com.google.android.youtube" package and is called by the "Activity State Change Controller" Task.

Check Activity State Changes Project docs to get more info on how this task is called and what it should do.


Input %par1:
"
activity
"

Input %par2:
"
activity_transistion
"

This task does not return anything.
```
##


**#:** `8`  
**Name:** `Fullscreen Activity Transition Screen Brightness Controller`  
**ID:** `864`  
**Collision Handling:** `Run Both Together`  
**Keep Device Awake:** `false`  
**Help:**
```
A helper task that can be used to set screen brightness if an activity enters or exits fullscreen mode. It ideally should be called by the "package_name Activity State Change Responder" Tasks.

Currently it increases screen brightness to 75% if its less than that when an activity enters fullscreen mode and battery level is higher than 20%. It resets brightness to the old level which was set before the activity entered fullscreen mode.
 
Support for setting screen brightness based on current light level set in Tasker in-built variable %LIGHT should be added later.

action must be set to "enter_fullscreen" or "exit_fullscreen"


Input %par1:
"
action
"

Returns:
"
result_code
"

If the task is successful, then result_code will contain "0".
Otherwise it will contain an appropriate exit code.
```
##


**#:** `9`  
**Name:** `Anonymous (860)`  
**ID:** `860`  
**Collision Handling:** `Abort New Task`  
**Keep Device Awake:** `false`  
**Help:** `-`
##

