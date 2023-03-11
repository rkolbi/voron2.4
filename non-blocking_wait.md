Going from a blocking, uninterruptable, waiting routine to an interruptible, nonblocking, routine.  
Can see this in use at:
https://github.com/rkolbi/voron2.4/tree/main/Tap-Tap-Sensorless-Huvud
In the above example, WAIT_Quit has been integrated with the normal resume function to make it more stream lined for user control.

This first snippet is my heat-waiting routine, using for loops and G4 pauses. Minutes to wait are passed on macro call. While this works well and does it's job, there is no graceful way to exit this cycle.

```
[gcode_macro _HEAT_WAIT]
description: Heating cycle waiting routine
gcode:
    {% set MINUTES = params.MINUTES|default(10)|int %}
    {% set MSG = params.MSG|default("Warming...")|string %}
    {% for i in range(0, MINUTES) %}
        M117 {MSG} {MINUTES-i} minute remaining.
        {% for s in range(0, 60) %}
            SET_LED LED=nozzle INDEX=1 RED=.5 GREEN=0 BLUE=0
            SET_LED LED=nozzle INDEX=2 RED=0 GREEN=0 BLUE=.5
            G4 P500
            SET_LED LED=nozzle INDEX=1 RED=0 GREEN=0 BLUE=.5
            SET_LED LED=nozzle INDEX=2 RED=.5 GREEN=0 BLUE=0
            G4 P500
        {% endfor %}
    {% endfor %}
```



This second snippet bundle nearly duplicates the role of the above snipper, but the wait cycle can be exited by running the macro, "WAIT_Quit". Minutes to wait are passed on calling macro, "WAIT_Start". When the cycle has completed, or forced exited, the following process macro must be directly called within the 'FINAL ACTION' block in the "WAIT_Delayed" delayed macro.

```
[gcode_macro _WAIT_Variable]
variable_count: 300
variable_duration: 2
variable_waiting: False
# Total wait minutes is (duration * count) / 60 
gcode:

[delayed_gcode WAIT_Delayed]
gcode:
    {% set count = printer["gcode_macro _WAIT_Variable"].count|int %}
    {% set duration = printer["gcode_macro _WAIT_Variable"].duration|int %}
    M117 Pre-Print Heatsoaking... {((duration * count) / 60)|round(1)} minutes left.
    SET_GCODE_VARIABLE MACRO=_WAIT_Variable VARIABLE=count VALUE={count-1}
    {% if count > 0 %} _WAIT_Loop  {% endif %}
    {% if count == 0 %} 
        # FINAL ACION
        SET_GCODE_VARIABLE MACRO=_WAIT_Variable VARIABLE=waiting VALUE=False
        M118 Reached the end, do final action
        # FINAL ACION
    {% endif %}

[gcode_macro _WAIT_Loop]
gcode:
    {% set count = printer["gcode_macro _WAIT_Variable"].count|int %}
    {% set duration = printer["gcode_macro _WAIT_Variable"].duration|int %}
    UPDATE_DELAYED_GCODE ID=WAIT_Delayed DURATION={duration}
    {% if  count % 2 == 0 %}
        SET_LED LED=nozzle INDEX=1 RED=.5 GREEN=0 BLUE=0
        SET_LED LED=nozzle INDEX=2 RED=0 GREEN=0 BLUE=.5
      {% else %}
        SET_LED LED=nozzle INDEX=1 RED=0 GREEN=0 BLUE=.5
        SET_LED LED=nozzle INDEX=2 RED=.5 GREEN=0 BLUE=0
    {% endif %}

[gcode_macro WAIT_Start]
gcode:
    {% set MINUTES = params.MINUTES|default(15)|int %}
    {% set duration = printer["gcode_macro _WAIT_Variable"].duration|int %}
    {% set count = (MINUTES * 60) / duration %}
    SET_GCODE_VARIABLE MACRO=_WAIT_Variable VARIABLE=waiting VALUE=True
    SET_GCODE_VARIABLE MACRO=_WAIT_Variable VARIABLE=count VALUE={count}
    UPDATE_DELAYED_GCODE ID=WAIT_Delayed DURATION={duration}

[gcode_macro WAIT_Quit]
gcode:
    {% if printer["gcode_macro _WAIT_Variable"].waiting %}
        M118 STOPPING LOOP, SETTING COUNT TO 0
        SET_GCODE_VARIABLE MACRO=_WAIT_Variable VARIABLE=count VALUE=0
        UPDATE_DELAYED_GCODE ID=WAIT_Delayed DURATION=1
      {% else %}
        M118 Not in waiting state, nothing to do.
    {% endif %}
```

