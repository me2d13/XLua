From [JGregory's post](https://forums.x-plane.org/index.php?/forums/topic/154351-xlua-scripting/&do=findComment&comment=1460039)

# Script packaging and basic structure

XLua scripts are organized into "modules". Each module is a fully independent script that runs inside your aircraft.
* Modules are completely independent of each other and do not share data. They are designed for isolation.
* You do not need to use more than one module. Multiple modules are provided to let work be copied as a whole from one aircraft to each other.
* Each module can use one .lua script file or more than one .lua script file, depending on author’s preferences.

So, here are two methods you can use for this:
1. As you described, place each script in its own directory (module) with the script name matching the folder name. See the default 747 for examples.
2. You can also place multiple scripts in one directory (module). The "main" script file is loaded and run automatically. The other scripts in a directory will load and run based on naming convention used. If the script names include the directory name, they will load and run automatically (see the default C90 for examples), otherwise you will need to use `dofile(script_name)` to load and run the other scripts.

Subfolders in the scripts folder are not allowed. All modules must be within "scripts".

The file init.lua is part of the XLua plugin itself and should not be edited or removed.

# How a module script runs

When your aircraft is loaded (before the .acf and art files are loaded) the XLua plugin is loaded, and it loads and runs each of your module scripts.

When your module’s script is run, all Lua code that is outside of any function is run immediately. Your script should use this immediate execution only to:

* Create new datarefs and commands specific to your module and
* Find built-in datarefs and commands from the sim. All other work should be deferred until you receive additional callbacks.

Once the aircraft itself has been loaded, your script will receive a number of major callbacks. These callbacks run a function in your script if a function with the matching name is found. You do not have to implement a function for every major callback, but you will almost certainly want to implement at least some of them.

Besides major callbacks, one other type of function in your script will run: when you create or modify commands and when you create writeable datarefs, you provide a Lua function that is called when the command is run by the user or when the dataref is written (e.g. by the panel or a manipulator).

# Major callbacks

* `aircraft_load()` - run once when your aircraft is loaded. This is run after the aircraft is initialized enough to set overrides.
* `aircraft_unload()` - run once when your aircraft is unloaded.
* `flight_start()` - run once each time a flight is started. The aircraft is already initialized and can thus be customized. This is always called after aircraft_load has been run at least once.
* `flight_crash()` - called if XPlane detects that the user has crashed the airplane.
* `before_physics()` - called every frame that the sim is not paused and not in replay, before physics are calculated
* `after_physics()` - called every frame that the sim is not paused and not in replay, after physics are calculated
* `after_replay()` - called every frame that the sim is in replay mode, regardless of pause status.

# Global variables provided by XLua

* `SIM_PERIOD` contains the duration of the current frame in seconds (so it is always a fraction). Use this to normalize rates, e.g. to add 3 units of fuel per second in a per frame callback you’d do fuel = fuel + 3 * SIM_PERIOD
* `IN_REPLAY` evaluates to 0 if replay is off, 1 if replay mode is on.

# Datarefs

* `create_dataref("name", "type")` creates a read-only dataref.  Note that this dataref can be changed by your script, e.g. with x = 5.  It cannot be changed by X plane or other plugins.
* `create_dataref("name", "type", my_func)` creates a writeable dataref.  The function is called each time a plugin other than your script writes to the dataref.  It is OK to have the function do nothing.

# Commands

* `find_command("sim/operation/pause_toggle")`
* `create_command(name, description, function)`
* `replace_command(name, handler)`
* `wrap_command(name, before_handler, after_handler)`

Command objects have three methods:
* __once()__ runs the command exactly once.
* __start()__ starts holding down the command.
* __stop()__ stops holding down the command.

Custom Command Function (when using create_command())

    function command_handler(phase, duration)

The phase is an integer that will be 0 when the command is first pressed, 1 while it is being held down, and 2 when it is released. For any command invocation, you are guaranteed exactly one begin and one end, with one or more “held down” in the middle. But note that if the user has multiple joysticks mapped to the command you could get a second down while the first one is running.

The duration is how long the command has been held down in seconds, starting at 0.

`create_command` returns a command object but you can ignore that object if you just need to make a command and not run it yourself.

# Customizing commands

`replace_command` and `wrap_command` are used to customize existing (sim) commands:

* `replace_command(name, handler)` takes an existing command (e.g. one of X Plane’s commands) and replaces the action the command does with your handler. The handler has the same syntax as the custom command handler; it takes a phase and duration.
* `wrap_command(name, before_handler, after_handler)` takes an existing command (e.g. one of X Plane’s commands) and installs two new handlers the before handler runs before X Plane, and the after handler runs after. You can use this to “listen” for a command and do extra stuff without losing X Plane’s capabilities. You must provide both functions even if one does nothing.

# Timers

You can create a timer out of any function. Timer functions take no arguments.

* `run_at_interval(func, interval)` runs func every interval seconds, starting interval seconds from the call.
* `run_after_time(func, delay)` runs func once after delay seconds, then timer stops.
* `run_timer(func, delay, interval)` runs func after delay seconds, then every interval seconds after that.
* `stop_timer(func)` ensures that func does not run again until you re schedule it; any scheduled runs from previous calls are canceled.
* `is_timer_scheduled(func)` returns true if the timer function will run at any time in the future. It returns false if the timer isn’t scheduled or if func has never been used as a timer.
