rpi2casterd
===========

Hardware driver and web API for rpi2caster
------------------------------------------

This is a machine control daemon for the ``rpi2caster`` typesetting and casting software.
It is supposed to run on a Raspberry Pi (any model) with an output expander based on two
MCP23017 chips to provide 32 additional outputs. These are connected to solenoid valves,
which in turn send the pneumatic signals to a Monotype composition caster or tape punch.

The program uses ``Flask`` to provide a rudimentary JSON API for caster control.

``gpiozero`` library is used for GPIO control, with RPi.GPIO as a preferable backend. 

There are several available MCP23017 control backends:

1. SMBus (via ``smbus-cffi`` or ``smbus2`` package),
2. ``WiringPi`` library.


The daemon also controls several GPIO pins:

1. `ready LED (green)` - when lit, the control device and software is ready to use.
2. `working LED (green)` and `error LED (red)`, typically a dual-color common cathode LED, indicates the machine state - 
   green when the machine is working, red when the machine is stopping the pump, and orange when the machine is starting.
3. `motor start` and `motor stop` - pulse outputs for start/stop relays, connected with the original AllenWest motor starter 
   (their use is optional, more of a convenience).
4. `air` and `water` - for controlling air solenoid valve (prevents unnecessary air use when the machine is not working) 
   and cooling water valve/pump (ditto with water). Like motor control, this is more of a 'deluxe' feature and is not 
   necessary for caster operation.
5. `sensor` (photocell, e.g. TCST2103) input for getting the information about the machine cycle phase. When the sensor is 
   going ON, the air is fed into the machine; when the sensor is going OFF, the air is cut off and the control daemon 
   ends the signals sending sequence. This sensor is necessary for caster operation. Punching is timer-driven 
   and no sensor is needed.
6. `mode sense` input - when grounded, the interface works in the casting mode; when lifted (pulled up to 3V3), 
   the interface works in the punching mode. This input is typically connected with a 9-pin D-sub connector for the sensor, 
   with a jumper to the ground in the plug.
7. `shutdown` and `reboot buttons` - after one of these is held for 2 seconds, the LED flashes and the shutdown or reboot
   procedure begins.
8. `emergency stop button` - stops the machine as soon as possible and marks the emergency stop as activated; when that happens, 
   the client software has to clear the emergency stop first in order to be able to use the machine. 


The program uses ``Flask`` to provide a rudimentary JSON API for caster control.

Starting
--------

The interface needs to be started up in order to work. The startup procedure ensures that:

1. the interface is not busy, not stopping and not starting - has not been claimed by any other client,
2. air and (for casting only) water and motor is turned on, if the hardware supports this,
3. (for casting) the machine is actually turning; during this phase, the state LED lights up orange,
4. after the starting sequence is successfully finished, the state LED lights up green,
5. the interface will stay busy until released by the ``stop`` method.

Machine is started with a request from the client software. See the API section for details.


Stopping
--------

Stopping the interface ensures that:

1. if the pump is working, it is stopped (see the pump control section); during this phase the state LED lights up red,
2. air and (for casting) water and motor is turned off, if hardware supports this,
3. the state  LED is turned off, if hardware supports this,
4. the `testing_mode` flag is set to False,
5. the interface is released for the future clients to claim.

Machine is stopped when called by the client software, when the machine has been stalling (waiting for the signal 
from the cycle sensor for too long), or when emergency stop happens because of button press or client software request.


Pump control
------------

The software turns the pump on (sending ``NKS 0075`` + current 0075 justifying wedge position) or off.
Pump switch-off is done whenever the machine stops and the pump is marked as working. This ensures that after re-start, 
the pump will stay stopped.

During the pump switch-off procedure, an "alarm" LED (red) is lit to prompt the operator to turh the
machine's main shaft a few times. The interface will then send a ``NJS 0005`` + current 0005 justifying wedge position. 
This way, stopping the pump does not change the wedge position.


Motor control
-------------

When starting in the casting mode, the software activates the ``motor_start`` GPIO for a fraction of a second.
The GPIO can be coupled with a NO SPST relay connected with the original AllenWest electromagnetic starter.
Use a relay rated for at least 400V AC if your caster is wired for three-phase power (common in continental Europe).
The relay should be connected to the contacts marked "1" and "2" on the motor starter.

Similarly, when stopping in the casting mode, the software activates the ``motor_stop`` GPIO. This can be coupled 
with a NC SPST relay that breaks the current flow through the starter's coil. The relay should be connected instead 
of a jumper between one of the live wires and the contact marked as "2".


Air and water control
---------------------

The daemon can also control a solenoid valve to enable or disable air flow when the machine is working or stopped.
Air control works in all operation modes (casting, punching and testing).

Water control can be used in the casting mode for controlling a pump or solenoid valve for cooling water flow.


Sending signals
---------------

Based on the caster's current operation mode, signals are modified or not:

1. testing mode ensures that signals 1...14, A...N, 0075, S, 0005, O15 are sent to the machine as they are received
2. punching mode ensures that a combined signal O+15 is present only when less than 2 signals are received
3. casting mode ensures that the O15 signal is ommitted

Sending the signals can take place only when the interface has been previously started and claimed as busy;
otherwise, ``InterfaceNotStarted`` is raised in the casting mode, and the startup is done automatically
in the punching and testing modes.

The daemon behaves differently depending on the operation mode:


casting
_______

1. wait for a machine cycle sensor to turn ON,
2. activate the valves for specified signals,
3. wait until the cycle sensor goes OFF,
4. turn all the valves off,
5. check the pump state and justifying wedge positions, and update the current state,
6. return a reply to the request, allowing the client to cast the next combination.

However, a machine sometimes stops during casting (e.g. when the operator sees a lead squirt
and has to stop immediately to prevent damage). In case of emergency stop, the machine is stopped immediately
and the client software gets an error reply to the send request.


punching
________

This mode is fully automatic and driven by a configureble timer:

1. turn the valves on,
2. wait time_on for punches to go up,
3. turn the valves off,
4. wait time_off for punches to come back down,
5. check the pump state and justifying wedge positions, and update the current state,
6. return a success reply to the request.


testing
_______

The software just turns off the valves, then turns them on, sending the specified signal combination.


REST API documentation
======================

The API is typically accessed at ``http://[address]:[port]`` (typically ``23017``, as in MCP23017). 
Several endpoints are available:

``/`` - status: ``GET``: reads and ``POST`` changes the status, which is used mostly for setting the temporary ``testing_mode`` flag.

``/config`` - configuration: `GET` reads and `POST` changes the configuration

``/machine`` - machine start/stop/state:
 
``GET`` reads the state, ``PUT`` turns the machine on, ``DELETE`` turns the machine off, and ``POST`` turns it on or off
depending on the JSON data in the request (``{state: true}`` for starting, ``{state: false}`` for stopping).
The reply can either be ``{success: true, active: [true/false]}`` if successful, or ``{success: false, error_code: [EC], error_name: [EN]}``
if exception was raised. Error codes and names:

1. ``0: The machine was abnormally stopped.`` in case of emergency stop or machine stalling,
2. ``3: This interface was started and is already in use. If this is not the case, restart the interface.`` if the interface has already
   been claimed as busy,

``/motor``, ``/air``, ``/water``, ``/pump``, ``valves`` - motor, air, water, pump and solenoid valves checking/control. The verbs work as above.

``/emergency_stop``: 

``GET`` gets the current state, ``PUT`` (or ``POST`` with ``{state: true}`` JSON data) activates the emergency stop,
``DELETE`` (or ``POST`` with ``{state: false}`` JSON data) clears the emergency stop state, allowing the machine to start.
When emergency stop is activated, the server replies with ``{success: false, error_code: 0, message: 'The machine was abnormally stopped.'}``

``/signals``: 

``GET``: gets the last signals sent (unless the machine was stopped, which clears the signals),
``POST`` or ``PUT`` with ``{signals: [sig1, sig2...], timeout: x}`` (timeout is optional and overrides the default machine stalling timeout)
sends the specified signals, and ``DELETE`` turns off the valves. Emergency stop events are tracked and whenever the emergency stop was triggered,
the server will reply with an error message.

Possible error replies:

1. ``0: The machine was abnormally stopped.`` in case of emergency stop or machine stalling,
2. ``4: Trying to cast or punch with an interface that is not started.`` (only in casting mode, as punching/testing starts the interface automatically)
