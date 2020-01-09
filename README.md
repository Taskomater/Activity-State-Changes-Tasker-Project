# Activity State Changes Tasker Project

`Activity State Changes Tasker Project` for android provides a way to detect android app activity state changes like activity changes in the same app or between different ones or if an activity enters or exits the fullscreen mode. This majorly relies on logcat entries detected using the Tasker `Logcat Entry` Profile and requires Tasker to be granted `android.permission.READ_LOGS`.
##


### Contents:
- [Project Details](###Project-Details)
- [How Project Works](###How-Project-Works)
- [Activity States And Transitions](###Activity-States-And-Transitions)
- [Dependencies](###Dependencies)
- [Downloads](###Downloads)
- [Install Instructions For Termux In Android](###Install-Instructions-For-Termux-In-Android)
- [Usage](###Usage)
- [Current Features](###Current-Features)
- [Planned Features](###Planned-Features)
- [Issues](###Issues)
- [Worthy of Note](###Worthy-of-Note)
- [Finding Device Specific Logcat Entries](###Finding-Device-Specific-Logcat-Entries)
##


### Project Details:

Each android app has a package name which is a unique name that allows android to differentiate between different apps even if they have the same name. For example, the package name for Tasker is `net.dinglisch.android.taskerm`.  Each app can contain different activities inside it which are basically views or screens. For example Tasker has the homescreen activity, the settings activity, task edit activity and so on. 
Activities can be of the following types:
- Non-Fullscreen mode activities in which the status bar and navigation bar is always visible.
- Fullscreen mode activities in which the status bar and optionally the navigation bar is hidden. May require tapping the screen to hide/unhide the bars to enable/disable fullscreen mode. Example: Images or videos opened in gallery or messenger apps.
- Activities that can change their config which may include changing between Fullscreen and Non-Fullscreen mode. Example: Video in Chrome or Youtube app in a small windows or in fullscreen.

To find activity transitions or activity config changes, every activity changed needs to be tracked. For this the Tasker `Logcat Entry` Profile can be used. Following are some logcat entries that can be used. 

The logcat entry that should be added when any activity is started that is not in the activity stack:

```
ActivityManager: Displayed com.some.package/.some.activity
```

The log entry that should be added when any activity is displayed and the previous one is paused:
```
ActivityTrigger: ActivityTrigger activityPauseTrigger
```

The logcat entry that should be added when the current activity changes its config:

```
ActivityManager: Config change.*
```

The above logcat entries should ideally exist for almost every device if not all unless changes were made to the AOSP.

The `ActivityManager: Displayed` will not be logged for activities which are returned to after pressing back button and will only be logged when an activity is started which is not in the activity stack and so basically the entry cannot be used to detect all activity transitions accurately.

The `ActivityTrigger: ActivityTrigger activityPauseTrigger` can be used to reliable detect all activity transitions but since the entry itself does not contain the package and activity name, other ways need to be used to detect what activity is currently in focus. But this creates lots of problems because if the `Logcat Entry` Profile entry task takes too long to process an entry, then by the time the turn comes for queued tasks, the package and/or activity might have already changed for which those Logcat entries were generated for, resulting in false activity transitions being calculated.

Currently two ways are used to detect which activity is currently in focus:
- For non-root users the `GetCurrentAppAndActivity` function of the `Tasker Function` action is used, but the value returned by it is sometimes that of a previous activity since Tasker does not receive/calculate the new value fast enough and a wait action is required of `0.5-1s` before running it. The exact time of the `Wait` action at runtime will vary depending on other same priority tasks running in tasker or the device load itself and may even vary for different devices, and so activity transitions may not always be accurate.
- For root users, the `mFocusedActivity` value is extracted from the `dumpsys activity activities` command. This ideally should work for almost every device if not all. This is relatively more reliable for getting the current package and activity value accurately since its updated fast enough and a `Wait` action is not required. The full command to extract `GetCurrentAppAndActivity` value is `dumpsys activity activities | grep mFocusedActivity | sed -E 's/.*ActivityRecord\{[^ ]+ [^ ]+ ([^ ]+) [^ ]+\}.*/\1/'`.

But with both these methods false activity transitions will still occur in cases when the entry task is slow or queued because of higher priority tasks running in tasker and packages and/or activities have already changed. It is best to find device specific Logcat entries for all activity resumes which also contain the package and activity name in the format `package_name/activity_name`. These should normally exist in all devices but will vary depending on device manufacturer and android version. Check the [Finding Device Specific Logcat Entries](###Finding-Device-Specific-Logcat-Entries) section for more info on how to find them.

The `ActivityManager: Config change.*` does not require finding the currently focused activity because the last opened activity is used.

Now the above only detects if an activity is started or changed but does not detect whether the activity is currently in fullscreen mode or not. There is also a prerequisite for checking whether an activity is fullscreen or not. When an image or video is opened in fullscreen mode, the screen is not in a complete fullscreen mode if the status bar and/or navigation bar is still visible. The screen needs to be tapped for them to be hidden and at this point the activity will be considered to be in a fullscreen mode. Hence, whenever an activity is started or its config changed, a wait action needs to be used to give user the time to tap the screen and for android to update its fullscreen state. The wait by default for both states is currently `2s`.

After waiting, it can be checked if the activity is fullscreen or not. For this, there are two ways:
- For root users, the `mTopIsFullscreen` value is extracted from the `dumpsys window policy` command. If this is set to `true`, the activity is in fullscreen mode. This ideally should work for almost every device if not all. The full command to extract `mTopIsFullscreen` value is `dumpsys window policy | grep mTopIsFullscreen | sed -E 's/.*mTopIsFullscreen=([^ ]+) .*/\1/'`.
- For non-root users, the `mSystemUiVisibility` value of the [StatusBar.java](https://github.com/aosp-mirror/platform_frameworks_base/blob/master/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java) Class or [previously the PhoneStatusBar.java](http://gerrit.aospextended.com/plugins/gitiles/AospExtended/platform_frameworks_base/+/2a6ea9c2a1b52b0386270ec73e1e6d6a9b614a34
) Class is used. The `mSystemUiVisibility` variable stores bit flags. It will have the `View.SYSTEM_UI_FLAG_FULLSCREEN` or `View.SYSTEM_UI_FLAG_HIDE_NAVIGATION` flags set when an activity is in fullscreen or immersive mode and `View.SYSTEM_UI_FLAG_IMMERSIVE` or `View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY` flags when an activity is in immersive mode. These are the flags used by `inFullscreenMode` and `inImmersiveMode` functions of the `StatusBar.java` Class to check fullscreen or immersive mode respectively and the same bitwise operations are performed in tasker. But first the current `mSystemUiVisibility` value needs to be found. For that the logcat command is used. Whenever the `setSystemUiVisibility` function of the `StatusBar.java` Class is called, it logs it with `setSystemUiVisibility displayId=%d vis=%s mask=%s oldVal=%s newVal=%s diff=%s"`, where the value of the `newVal` variable is the current `mSystemUiVisibility` value as a hex string. So basically, what is done to check whether the current activity is in fullscreen mode or not is by finding the current `mSystemUiVisibility` value from the last `setSystemUiVisibility.*` logcat entry by extracting the `newVal` value and checking if the fullscreen flags are set or not. This `setSystemUiVisibility.*` logcat entry should ideally exist for almost every device if not all unless changes were made to the AOSP. Currently this method works if the entry is logged by either the `StatusBar` or `PhoneStatusBar` tag/component. However, if it is logged by other components in some devices, then that can be looked into based on feedback from users. 

The details for fullscreen and immersive modes can be found [here](https://developer.android.com/training/system-ui/immersive).

Once it is found if the activity is fullscreen or not, then the current activity state is calculated. After this, if the current or previous activity's package name responder tasks are found in the tasker config, then activity transitions are calculated from previous and current activity states and passed to the respective tasks.

### How Project Works:


**Logcat Entry Profiles**:
- `ActivityTrigger Activity Start Monitor` is trigger by a `Logcat Entry` Event to detect `activity_start` activity_state_change. This is the default way to detect if an activity is started. This is enabled by default.
	**Component:** `ActivityTrigger` **Filter:** `ActivityTrigger activityPauseTrigger`.
- `Custom Activity Start Monitor` is trigger by a `Logcat Entry` Event to detect `activity_start` activity_state_change. This should be used by users who found logcat entries for their device to detect if an activity is started or resumed that also contain the package and activity name. This is disabled by default and the values set are of the dev's device.
- `ActivityManager Activity Config Change Monitor` is trigger by a `Logcat Entry` Event to detect `activity_config_change` activity_state_change. This is the default way to detect if an activity's config has changed. This is enabled by default.

The 3 profiles above call the `Activity State Change Relay` Task that finds the currently opened package and activity for `activity_start` activity_state_change, either from `%lc_text` or with other ways mentioned earlier. Then it sets the `%ActivityStateChangeControllerCommand` variable with the `%current_package_and_activity` and `%activity_state_change` values which triggers the `Activity State Change Controller Command Monitor` Profile which calls the `Activity State Change Controller` Task to further process the activity state change and call any other required tasks for respected packages and activities. Note that either `ActivityTrigger Activity Start Monitor` or `Custom Activity Start Monitor` must be activated at the same time. If they are both activated, then this can cause duplication and result in fake transitions. 

The `Activity State Change Controller` Task first checks if the current_package_and_activity passed is a valid package and activity name in the format `package_name/activity_name`. If it is then it checks if the current activity is in fullscreen mode with status bars hidden. Then it sets the `current_activity_state` depending on `activity_state_change` and `fullscreen_mode`. Then it checks if the a `package_name Activity State Change Responder` Task exists in the Tasker config for the `current_package_and_activity` or `previous_package_and_activity`. If either of those exists, then it calculates the `previous_activity_task_activity_transition_par` and `current_activity_task_activity_transition_par` depending on activity transitions in the transitions table defined in [Activity States And Transitions](###Activity-States-And-Transitions). Then it calls the `previous_package Activity State Change Responder` Task if it exists with the `previous_activity` as `%par1` and `previous_activity_task_activity_transition_par` as `%par2`. Then it calls the `current_package Activity State Change Responder` Task if it exists with the `current_activity` as `%par1` and `current_activity_task_activity_transition_par` as `%par2`. Those tasks may respond appropriately to activity transitions but must not perform long running operations since this task will not finish until the called tasks are finished to maintain order and any queued tasks for this task will also be in waiting. Any long running operations that do not require ordered execution can be run inside the called tasks in additional tasks with `%priority - 1` so that the called tasks can return before the additional tasks finish.

The `Reset Activity State Change Variables On Monitor Start` Profile is called by the `Reset Activity State Change Variables On Monitor Start` Task when Tasker Monitor is started. It should ideally also be manually called when the `* Activity Start Monitor` and `* Activity Config Change Monitor` tasks are enabled/disabled. Resetting variables is required to prevent false activity transitions if activity states were stopped from being monitor either by tasker being killed or manually by the user.

The `package_name Activity State Change Responder` tasks must be created for each app whose activity transitions need to be handled. This is needed so that further processing and task calling is not done for each activity that is started and only for the apps required. It also separates app specific activities and activity transition logics into different tasks which creates a better design. 4 example tasks are provided by this project for `Chrome`, `Youtube`, `WhatsApp` and `F-Stop` apps that currently only increase/decrease brightness on fullscreen enter/exits.

Check [Activity State Changes Project Info](projects/Activity_State_Changes.prj.md) file for more info of the profiles and tasks.
##


### Activity States And Transitions

#### Activity States:
- `start_fullscreen`: fullscreen mode activity
- `start_non_fullscreen`: non-fullscreen mode activity
- `config_change_fullscreen`: an activity whose config was changed to fullscreen mode
- `config_change_non_fullscreen`: an activity whose config was changed to non-fullscreen mode

#### Activity Transitions:
- `start_fullscreen`: activity started in fullscreen mode
- `start_non_fullscreen`: activity started in non-fullscreen mode
- `exit_fullscreen`: activity exited from fullscreen mode
- `exit_non_fullscreen`: activity exited from non-fullscreen mode
- `restart_fullscreen`: activity with same name was started in fullscreen mode
- `restart_non_fullscreen`: activity with same name was started in non-fullscreen mode
- `config_change_fullscreen`: activity config was changed to fullscreen mode
- `config_change_non_fullscreen`: activity config was changed to non-fullscreen mode
- `config_rechange_fullscreen`: activity config was changed again to fullscreen mode
- `config_rechange_non_fullscreen`: activity config was changed again to non-fullscreen mode


#### Activity Transition Table:
```
If previous_activity==current_activity:
	#previous_activity_state     #current_activity_state         #current_activity_task_activity_transition_par
	start_non_fullscreen         start_non_fullscreen         -> restart_non_fullscreen
	start_non_fullscreen         start_fullscreen             -> restart_fullscreen
	start_non_fullscreen         config_change_non_fullscreen -> config_change_non_fullscreen
	start_non_fullscreen         config_change_fullscreen     -> config_change_fullscreen
	start_fullscreen             start_fullscreen:            -> restart_fullscreen
	start_fullscreen             start_non_fullscreen         -> restart_non_fullscreen
	start_fullscreen             config_change_non_fullscreen -> config_change_non_fullscreen
	start_fullscreen             config_change_fullscreen     -> config_change_fullscreen
	config_change_non_fullscreen config_change_non_fullscreen -> config_rechange_non_fullscreen
	config_change_non_fullscreen config_change_fullscreen     -> config_change_fullscreen
	config_change_non_fullscreen start_fullscreen             -> restart_fullscreen
	config_change_non_fullscreen start_non_fullscreen         -> restart_non_fullscreen
	config_change_fullscreen     config_change_fullscreen     -> config_rechange_fullscreen
	config_change_fullscreen     config_change_non_fullscreen -> config_change_non_fullscreen
	config_change_fullscreen     start_fullscreen             -> restart_fullscreen
	config_change_fullscreen     start_non_fullscreen         -> restart_non_fullscreen

If previous_activity!=current_activity:
	#previous_activity_state     #current_activity_state         #previous_activity_task_activity_transition_par #current_activity_task_activity_transition_par
	start_non_fullscreen         start_non_fullscreen         -> exit_non_fullscreen start_non_fullscreen
	start_non_fullscreen         start_fullscreen             -> exit_non_fullscreen start_fullscreen
	start_non_fullscreen         config_change_non_fullscreen -> exit_non_fullscreen config_change_non_fullscreen
	start_non_fullscreen         config_change_fullscreen     -> exit_non_fullscreen config_change_fullscreen
	start_fullscreen             start_fullscreen             -> exit_fullscreen     start_fullscreen
	start_fullscreen             start_non_fullscreen         -> exit_fullscreen     start_non_fullscreen
	start_fullscreen             config_change_non_fullscreen -> exit_fullscreen     config_change_non_fullscreen
	start_fullscreen             config_change_fullscreen     -> exit_fullscreen     config_change_fullscreen
	config_change_non_fullscreen start_fullscreen             -> exit_non_fullscreen start_fullscreen
	config_change_non_fullscreen start_non_fullscreen         -> exit_non_fullscreen start_non_fullscreen
	config_change_fullscreen     start_fullscreen             -> exit_fullscreen     start_fullscreen
	config_change_fullscreen     start_non_fullscreen         -> exit_fullscreen     start_non_fullscreen
```
##


### Dependencies:

- No specific dependencies other than requires Tasker to be granted `android.permission.READ_LOGS`. Either grant it over adb manually using `pm grant net.dinglisch.android.taskerm android.permission.READ_LOGS` command or use the script in [Tasker Package Utils](https://github.com/Taskomater/Tasker-Package-Utils) project which has a few more features.
##


### Downloads:

Download `Activity State Changes Tasker Project` latest release from [here](https://github.com/Taskomater/Activity-State-Changes-Tasker-Project/releases).
##


### Install Instructions For Tasker In Android:

1. Import `Activity_State_Changes-v*.prj.xml` Project file into Tasker.

##


### Usage:

1. Enable the `ActivityTrigger Activity Start Monitor`, `ActivityManager Activity Config Change Monitor`, `Activity State Change Controller Command Monitor` and `Reset Activity State Change Variables On Monitor Start` Profiles if not already action. You may optionally enable the `Custom Activity Start Monitor` instead of the `ActivityTrigger Activity Start Monitor` Profile if you found Activity resume logcat entries for your device. Do not enable both profiles together.

2. Optionally enable the use of non-root modes if you are a root user by disabling the `%use_root_mode` Variable Set action in the `Activity State Change Relay` and `Activity State Change Controller` Tasks or change their values from `1`.

3. Enable the `%lc_text` Flash action of the  `Activity State Change Relay` Task, the `%enable_debugging` Variable Set action of the `Activity State Change Controller` Task and optionally the `%action\n%%activity_transition` action of the `project_name Activity State Change Responder` Tasks. This should normally give you enough info on what is being run. The Flash actions will of course not be synchronized with what is being run.

4. Change the `Wait` action time in the `Activity State Change Relay` Task for the  `ActivityTrigger Activity Start Monitor` Profile conditional statement if you are using non-root mode.

5. If the fullscreen mode is not always being detected correctly, try to change the `Wait` action time in the `Wait For User To Hide Status Bar By Tapping Screen*` section of the `Activity State Change Controller` Task. Do not forget to tap the screen once to hide the status bar as soon as you enter a fullscreen activity for which you want to detect fullscreen mode. You may optionally not tap it to prevent the respective `project_name Activity State Change Responder` Task from running the fullscreen actions you defined.

6. Create `project_name Activity State Change Responder` Tasks for each app you want to handle activity transitions for. The tasks may respond appropriately to activity transitions but must not perform long running operations since  `Activity State Change Controller` Task will not finish until the called tasks are finished to maintain order and any queued tasks for it task will also be in waiting. Any long running operations that do not require ordered execution can be run inside the called tasks in additional tasks with `%priority - 1` so that the called tasks can return before the additional tasks finish.

7. If the default `ActivityTrigger Activity Start Monitor` with the `ActivityTrigger activityPauseTrigger` filter is never triggered, then find matching entries for `ActivityTrigger activityResumeTrigger` or other related pause/resume activity entries for your device in the logcat and update the profile. You may optionally find activity resume entries in your logcat that also contain the package and activity name and update the `Custom Activity Start Monitor` with them. In this case you also need to update the `Variable Search Replace` action in the `Custom Activity Start Monitor` Profile conditional actions in the `Activity State Change Relay` Task to extract the package name and activity name from the logcat entries and set them to  `%current_package_and_activity` in the format `package_name/activity_name`. 

8. If the default `ActivityManager Activity Config Change Monitor` is never triggered, try finding them in your logcat. If you can't, then contact the dev and ask him for help.
##


### Current Features:

- Detect activity transitions
- Detect if activities enter or leave fullscreen mode
- Sample tasks for handling activity transitions of some apps
##


### Planned Features:

- Not thought of yet.
##


### Issues:

- Some `package_name/activity_name` might be considered invalid. Currently the hyphen `-` character in activities is considered invalid by the `\p{javaJavaIdentifierPart}` character class of the validation regex for some unknown reason on the devs device. The regex may be changed in future.

##


### Worthy of Note:

- The activity name of activities defined in the `AndroidManifest.xml` of apps may start with a dot, which implies that the package name should automatically be prepended to their name. Depending on the method used to detect the current package and activity, the activity name may not be prepended with the package name and may start with a dot instead. So it is best to use `~` matches instead of `eq` conditional statements in the `project_name Activity State Change Responder` Tasks.

- You may increase the priority of `Logcat Entry` Profiles to a number higher than the default `5` to make them run as soon as they are trigerred to store the value of the current package and activity if other lower priority tasks are running in Tasker. But note that using `Wait` actions will prevent other tasks from executing until the profile entry tasks are complete and may create sluggish behaviour.

- The dumpsys commands are by default run in `Run Shell` actions with the root toggle enabled. Opening a root shell takes a tiny bit longer than normal non-root shells. If Tasker is installed as a system privileged app and is granted `android.permission.DUMP`, then root shell is not required and the root toggle may be disabled to slightly increase performance. You may use the script in [Tasker Package Utils](https://github.com/Taskomater/Tasker-Package-Utils) project to automatically install tasker as a system privileged app.
##


### Finding Device Specific Logcat Entries

There are a few ways to find device specific logcat entries for various things.

- Using Tasker `Logcat Event` Profile mechanism.
- Using the [Grab Timed And Filtered Logcat](https://github.com/agnostic-apollo/Tasker-Random-Stuff/tree/master/grab_timed_and_filtered_logcat) Task to grab a logcat for `x` seconds to a file. It also provides a way to filter tags/components and also provides a way to filter using regexes. A logcat file is much easier to read to see flow of things.
- If you have root access, then run `logcat | grep -E 'activity|trigger|resume|start|stop|config change|systemuivisibility|statusbar'` in a root shell in termux and then switch to multi-window and then switch between activities to see logcat changes in real time. You can pass any string in a regex to `grep` to filter entries you want to monitor.
- If you do not have root access, you may run the logcat command above over adb. If you are using windows, then `grep` command will not be available, install `cygwin` if required.

To find activity resume entries, adding part of the package or activity name to the filter regex can be helful to narrow down important logcat entries. For LG G5 7.0, the activity resuming entries match the following format:
```
LGImageQualityEnhancementService: activityResuming: package_name/activity_name
```
##


