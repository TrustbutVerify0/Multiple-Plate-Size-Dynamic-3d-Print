The problem: i have a 350x350 printer but multiple old plates with different textures and hologrm effects and dimensions that i want to use with the new printer.

This is configured and working on the SOVOL SV08
I use this to allow me to safely use different sized plates, ie a 256x256 plate on the SV08 350x350 bed.
I avoid bed crashes by configurein QGL dynamically and ALWAYS centering the plate on the bed. 
The macro's below take care of the calculations for the QGL points and purge position offset

To allow the defaul Klipper QGL to accept a points parameter we need to add this option to the default klipper qgl command; change the file below on the printer Klipper OS.
**/home/biqu/klipper/klippy/extras/quad_gantry_level.py
Replace this section with:

    def cmd_QUAD_GANTRY_LEVEL(self, gcmd):
        self.z_status.reset()
        self.retry_helper.start(gcmd)
        self.probe_helper.start_probe(gcmd)


this section

    def cmd_QUAD_GANTRY_LEVEL(self, gcmd):
                # Check for a POINTS parameter in the G-Code command
        points_str = gcmd.get('POINTS', None)
        if points_str is not None:
            try:
                # Parse the points string. Points are separated by ':'
                # and coordinates are separated by ','.
                new_points = [[float(coord) for coord in p.split(',')]
                              for p in points_str.split(':')]
                
                # The script requires exactly 4 points to function correctly
                if len(new_points) != 4:
                    raise gcmd.error(
                        "POINTS parameter must provide exactly 4 points")
                
                # Overwrite the configured points with the new ones
                self.probe_helper.probe_points = new_points
                gcmd.respond_info("Using custom points from G-Code command: %s"
                                  % (self.probe_helper.probe_points,))
            except ValueError:
                raise gcmd.error("Could not parse POINTS. Expected format: "
                                 "POINTS=x1,y1:x2,y2:x3,y3:x4,y4")

        self.z_status.reset()
        self.retry_helper.start(gcmd)
        self.probe_helper.start_probe(gcmd)


        
Now in the same file as you have your gcode_macro START_PRINT;

Add the GLOBAL CONFIGURATION VARIABLES to your printer config and replace your "gcode_macro START_PRINT" - as we are replacing  modify it to suit your printer;

		#====================================================================
		# GLOBAL CONFIGURATION VARIABLES
		#====================================================================
		
		[gcode_macro _PRINTER_CONFIG]
		description: Centralized printer configuration
		variable_park_positions: {
		    'pause': {'x': 0, 'y': 0, 'z': 10, 'e': 1},
		    'cancel': {'x': 0, 'y': 0, 'z': 20, 'e': 1},
		    'center': {'x': 175, 'y': 175}}
		variable_temperatures: {
		    'bed_mesh': 65,
		    'extruder_load': 230,
		    'extruder_preheat': 125,
		    'extruder_clean': 200,
		    'extruder_clean_cool': 130,
		    'temp_tolerance': 3}
		variable_speeds: {
		    'travel': 9000,
		    'pause_resume': 150,
		    'retract': 300,
		    'load': 300,
		    'load_fine': 150}
		variable_limits: {
		    'z_max': 345,
		    'bed_size_x': 350,
		    'bed_size_y': 350}
		variable_distances: {
		    'retract_pause': 1,
		    'retract_cancel': 1,
		    'retract_unload': 25,
		    'load_initial': 75,
		    'load_final': 30}
		variable_timings: {
		    'heat_soak': 0,
		    'purge_wait': 3000}
		gcode:
		    # This macro only holds variables
		
		#====================================================================
		# PRINT START/END MACROS
		#====================================================================
		
		gcode_macro START_PRINT]
		description: Enhanced start print macro with proper error handling
		variable_state: 'ready'
		gcode:
		    {% set config = printer['gcode_macro _PRINTER_CONFIG'] %}
		    {% set bed_target = params.BED_TEMP|default(60)|int %}
		    {% set extruder_target = params.EXTRUDER_TEMP|default(230)|int %}
		    {% set heatsoak = params.HEATSOAK|default(False)|lower == 'true' %}
		    {% set explicit_points = params.POINTS|default(none) %}
		    {% set plate_size = params.PLATE_SIZE|default(config.limits.bed_size_x)|int %}
			{% set zoffset = params.ZOFFSET|default(0)|float %}
		    
		    # Validate parameters
		    {% if bed_target < 0 or bed_target > 120 %}
		        {action_raise_error("Invalid bed temperature: " + bed_target|string)}
		    {% endif %}
		    {% if extruder_target < 0 or extruder_target > 300 %}
		        {action_raise_error("Invalid extruder temperature: " + extruder_target|string)}
		    {% endif %}
		    
		    M400
		    CLEAR_PAUSE
		    SET_GCODE_VARIABLE MACRO=START_PRINT VARIABLE=state VALUE='"starting"'
		
		    {% set plate_offset_x = (config.limits.bed_size_x - plate_size) / 2 %}
		    {% set plate_offset_y = (config.limits.bed_size_y - plate_size) / 2 %}
		
		    {% set purge_line_x_inset = 10 %}      # Inset from plate edge for X start
		    {% set purge_line_y_pos = 5 %}         # Y position relative to plate_offset_y
		    {% set purge_line_z_height = 0.5 %}    # Z height for purge line
		    {% set purge_extrusion_rate = 0.1328 %} # mm of filament per mm of travel (nozzle_diameter^2 * pi / 4) * extrusion_multiplier
		    {% set purge_length_percentage = 0.6 %} # Percentage of plate_size for purge line length (0.8 = 80%)
		    {% set purge_blob_e = 10 %}             # Amount of filament for initial blob
		
		    {% set purge_start_x = plate_offset_x + purge_line_x_inset %}
		    {% set calculated_purge_length_x = plate_size * purge_length_percentage %}
		    {% set purge_total_e = calculated_purge_length_x * purge_extrusion_rate %}
		    {% set purge_y = plate_offset_y + purge_line_y_pos %}
		
		    {% set qgl_inset = 30 %} # Inset from plate edge for QGL points
		
		    {% set qgl_p0_x = plate_offset_x + qgl_inset %}
		    {% set qgl_p0_y = plate_offset_y + qgl_inset %}
		    {% set qgl_p1_x = plate_offset_x + qgl_inset %}
		    {% set qgl_p1_y = plate_offset_y + plate_size - qgl_inset %}
		    {% set qgl_p2_x = plate_offset_x + plate_size - qgl_inset %}
		    {% set qgl_p2_y = plate_offset_y + plate_size - qgl_inset %}
		    {% set qgl_p3_x = plate_offset_x + plate_size - qgl_inset %}
		    {% set qgl_p3_y = plate_offset_y + qgl_inset %}
		
		    {% set dynamic_points = "%0.2f,%0.2f:%0.2f,%0.2f:%0.2f,%0.2f:%0.2f,%0.2f" % (qgl_p0_x, qgl_p0_y, qgl_p1_x, qgl_p1_y, qgl_p2_x, qgl_p2_y, qgl_p3_x, qgl_p3_y) %}
		  
		    # Check filament sensor
		    _CHECK_FILAMENT_SENSOR ;!! use your equivalent here instead i use a macro
		    
		    # Home if needed
		    _SAFE_HOME ;!! use your equivalent here instead i use a macro
		        
		    # Preheat extruder for bed leveling
		    {% if printer.extruder.temperature < config.temperatures.extruder_preheat %}
		        {action_respond_info("Preheating nozzle...")}
		        M104 S{config.temperatures.extruder_preheat}
		        _WAIT_FOR_TEMPERATURE SENSOR=extruder TARGET={config.temperatures.extruder_preheat}
		    {% endif %}
		    
		    # Heat bed
		    {% if printer.heater_bed.temperature < bed_target %}
		        {action_respond_info("Heating bed to " + bed_target|string + "°C...")}
		        M140 S{bed_target}
		        _WAIT_FOR_TEMPERATURE SENSOR=heater_bed TARGET={bed_target}
		    {% endif %}
		    M400
		    # Heat soak if requested
		    {% if heatsoak and config.timings.heat_soak > 0 %}
		        {action_respond_info("Heat soaking for " + config.timings.heat_soak|string + " minutes")}
		        G4 P{config.timings.heat_soak * 60000}
		    {% endif %}
		    
		
		    {% if explicit_points %}
		        {action_respond_info("Performing quad gantry level with explicit points: " + explicit_points)}
		        quad_gantry_level POINTS={explicit_points}
		    {% else %}
		        {action_respond_info("Performing quad gantry level with dynamic points: " + dynamic_points)}
		        quad_gantry_level POINTS={dynamic_points}
		    {% endif %}
		
		
		    # Bed mesh calibration with error handling
		    {action_respond_info("Calibrating bed mesh...")}
		    BED_MESH_CLEAR
		    
		    # Attempt bed mesh calibration
		    {% set mesh_success = False %}
		    BED_MESH_CALIBRATE ADAPTIVE=1
		    
		    # Verify mesh was created
		    {% if printer.bed_mesh.profile_name %}
		        {action_respond_info("Bed mesh calibration successful: " + printer.bed_mesh.profile_name)}
		        {% set mesh_success = True %}
		    {% else %}
		        {action_respond_info("Bed mesh calibration failed, attempting standard mesh...")}
		        # Try without adaptive if it failed
		        BED_MESH_CALIBRATE
		        {% if printer.bed_mesh.profile_name %}
		            {action_respond_info("Standard bed mesh calibration successful")}
		            {% set mesh_success = True %}
		        {% endif %}
		    {% endif %}
		    
		    {% if not mesh_success %}
		        {action_raise_error("Bed mesh calibration failed completely")}
		    {% endif %}
		    
			# SET ZOFFSET FROM SALICER PARAMS    
		    # Apply Z offset if specified
		    {% if zoffset != 0 %}
		        {action_respond_info("Applying Z offset: " + zoffset|string + "mm")}
		        SET_GCODE_OFFSET Z={zoffset}
		    {% endif %}
		    #/// SET ZOFFSET FROM SALICER PARAMS
		
			
		    M400  #await all phsical moves before printing more messages
		    
		    # Final heating
		    {action_respond_info("Final heating...")}
		    M140 S{bed_target}
		    M104 S{extruder_target}
		    _WAIT_FOR_TEMPERATURE SENSOR=heater_bed TARGET={bed_target}
		    _WAIT_FOR_TEMPERATURE SENSOR=extruder TARGET={extruder_target}
		    
		    #
		    # Dynamic Purge Line
		    # plate_offset_x + 10 to plate_offset_x + plate_size - 10
		    #
		    {action_respond_info("Performing dynamic purge line...")}
		    G90                 ; Set to absolute positioning
		    G1 X{purge_start_x} Y{purge_y} F{config.speeds.travel} ; Move to start of purge line
		    G1 Z{purge_line_z_height} F600     ; Move Z to purge height
		    M400               ; Wait for all moves to finish
		    G91                ; Set to relative positioning
		    M83                ; Set extruder to relative mode
		
		    # Initial blob purge
		    G1 E{purge_blob_e} F300        ; Extrude blob in place
		    G4 P500                         ; Dwell for 0.5 seconds to allow blob to form
		    
		    # Line purge
		    G1 X{calculated_purge_length_x} E{purge_total_e} F1800 ; Extrude along the line
		    G1 E-0.200 Z1 F600 ; Retract 0.2mm and raise Z by 1mm
		    M400               ; Wait for all moves to finish
		    G90                ; Return to absolute positioning
		
		    SET_GCODE_VARIABLE MACRO=START_PRINT VARIABLE=state VALUE='"printing"'
		    {action_respond_info("Print start sequence complete")}

Add in the new quad gantry level command, this dynamically replaces the built in qgl but we leave the original [QUAD_GANTRY_LEVEL_BASE] in the file for certain cases where we use a 350x350 plate, this change allows POINTS to be integrated:

    [gcode_macro QUAD_GANTRY_LEVEL]
    rename_existing: QUAD_GANTRY_LEVEL_BASE
    description: Enhanced QGL with temperature management and custom points support
    gcode:
        {% set config = printer['gcode_macro _PRINTER_CONFIG'] %}
        {% set target_temp = config.temperatures.bed_mesh %}
        {% set current_target = printer.heater_bed.target|int %}
        {% set points = params.POINTS|default(none) %}
        
        # Ensure bed is at proper temperature
        {% if printer.heater_bed.temperature < target_temp %}
            {action_respond_info("Heating bed for QGL...")}
            {% set heat_target = [current_target, target_temp]|max %}
            M140 S{heat_target}
            _WAIT_FOR_TEMPERATURE SENSOR=heater_bed TARGET={heat_target}
        {% endif %}
        
        _SAFE_HOME
        
        {% if points %}
            {action_respond_info("Performing quad gantry level with custom points: " + points)}
            QUAD_GANTRY_LEVEL_BASE POINTS={points}
        {% else %}
            {action_respond_info("Performing quad gantry level with default points...")}
            QUAD_GANTRY_LEVEL_BASE
        {% endif %}
        
        # Cool bed if it wasn't heated originally
        {% if current_target == 0 %}
            M140 S0
        {% endif %}

Now in Slicer of choice make these change to the gcode section. This is specific to Orca Slicer
ZOFFSET may also be needed because different plates have different thickness'
Set the plate x-y dimension as a single figure, ie a square of 310 as below   eg PLATE_SIZE=310 ZOFFSET=0.0

    SET_PRINT_STATS_INFO TOTAL_LAYER=[total_layer_count]
    START_PRINT PLATE_SIZE=310 ZOFFSET=0.0 EXTRUDER_TEMP=[first_layer_temperature] BED_TEMP=[first_layer_bed_temperature] FILAMENT=[filament_type] LAYER=[layer_height] BED_SIZE_X={print_bed_max[0]} BED_SIZE_Y={print_bed_max[1]}
    M118 BED_SIZE_X={print_bed_max[0]} BED_SIZE_Y={print_bed_max[1]}
    
    M400               ; Wait for all moves to finish
    ; Dynamic Purge Line
    ;    - no purge!!!1
    ; plate_offset_x + 10 to plate_offset_x + plate_size - 10
