# utl-altair-slc-python-controlling-a-pump-using-siemens-S7-programming-logic-controllers-plc
Altair slc python controlling a pump using siemens S7 programming logic controllers plc
    %let pgm=utl-altair-slc-python-controlling-a-pump-using-siemens-S7-programming-logic-controllers-plc;

    %stop_submiission;

    Altair slc python controlling a pump using siemens S7 programming logic controllers plc

    To long to post on a list,wee github
    https://github.com/rogerjdeangelis/utl-altair-slc-python-controlling-a-pump-using-siemens-S7-programming-logic-controllers-plc

    [SIEMENS](https://support.industry.siemens.com/cs/document/109813592)
    https://support.industry.siemens.com/cs/document/109813592/how-can-you-test-or-simulate-the-plc-program-of-a-software-controller-with-plcsim-or-plcsim-advanced-?dti=0&lc=en-US

    PROBLEM (create this output)

      Cycle 1 (Pump ON ? OFF):

      ON at 25.1°C (11:26:23)
      OFF at 22.8°C (11:26:25)

      Duration: 2 seconds of cooling

      Temperature drop: 2.3°C

      Cycle 2 (Pump ON at end):
      ON at 26.2°C (11:26:29)

      Program ended before it could turn OFF

      Temperature was dropping when stopped

    /*
     _ __  _ __ ___   ___ ___  ___ ___
    | `_ \| `__/ _ \ / __/ _ \/ __/ __|
    | |_) | | | (_) | (_|  __/\__ \__ \
    | .__/|_|  \___/ \___\___||___/___/
    |_|
    */

    options validvarname=v7;
    options set=PYTHONHOME "D:\py314";
    proc python;
    submit;
    # siemens_plc_with_simulator_auto_stop.py
    """
    A complete, self-contained example that simulates a Siemens S7 PLC and communicates with it.
    No physical PLC or external software required - everything runs in Python!
    The program automatically terminates after 15 seconds.
    """

    import snap7
    import time
    import logging
    import threading
    import random
    from snap7 import type
    from snap7.server import Server

    # Setup logging
    logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

    # --- Configuration ---
    PLC_IP = '127.0.0.1'  # Localhost - everything runs on your PC
    PLC_RACK = 0
    PLC_SLOT = 1

    # Data Block configuration (simulated PLC memory)
    DB_NUMBER = 100
    TEMP_ADDRESS = 0      # Byte offset for temperature (2 bytes)
    PUMP_ADDRESS = 2      # Byte offset for pump command (1 byte, bit 0)
    PUMP_STATUS_ADDRESS = 3  # Byte offset for pump status feedback (1 byte)

    # Runtime configuration
    RUNTIME_SECONDS = 15  # Program will run for this many seconds before auto-stopping

    # Control configuration - ADJUSTED FOR DEMONSTRATION
    TEMP_LIMIT = 25.0     # LOWERED from 30 to 25°C to trigger pump sooner
    TEMP_DEADBAND = 2.0   # Pump turns off at 23°C

    class PLCSimulator:
        """
        A simple Siemens S7 PLC simulator that mimics a real PLC's memory and behavior.
        This runs as a server that Python can connect to and communicate with.
        """

        def __init__(self, ip='127.0.0.1', rack=0, slot=1):
            self.ip = ip
            self.rack = rack
            self.slot = slot
            self.server = None
            self.running = False

            # Simulated process variables
            self.temperature = 20.0  # Starting temperature in Celsius
            self.pump_command = False  # Command from HMI (what our script writes)
            self.pump_is_running = False  # Actual pump status simulated

            # Simulation parameters
            self.ambient_temp = 20.0
            self.heating_rate = 0.8    # INCREASED from 0.3 to 0.8 for faster heating
            self.cooling_rate = 1.0    # INCREASED from 0.5 to 1.0 for faster cooling

            # Create memory areas for the simulated PLC
            self.db_memory = bytearray(100)  # 100 bytes of simulated DB memory

            # Initialize default values
            self.update_db_memory()

        def update_db_memory(self):
            """Update the simulated DB memory with current values"""
            # Write temperature (2-byte integer, scaled by 10 for decimal precision)
            temp_int = int(self.temperature * 10)
            temp_bytes = temp_int.to_bytes(2, byteorder='big')
            self.db_memory[TEMP_ADDRESS:TEMP_ADDRESS+2] = temp_bytes

            # Write pump command (bit 0 of the byte at PUMP_ADDRESS)
            if self.pump_command:
                self.db_memory[PUMP_ADDRESS] = self.db_memory[PUMP_ADDRESS] | 0x01
            else:
                self.db_memory[PUMP_ADDRESS] = self.db_memory[PUMP_ADDRESS] & 0xFE

            # Write pump status (1 = running, 0 = stopped)
            self.db_memory[PUMP_STATUS_ADDRESS] = 1 if self.pump_is_running else 0

        def read_db_memory(self, start, size):
            """Simulate reading from PLC memory"""
            return bytes(self.db_memory[start:start+size])

        def write_db_memory(self, start, data):
            """Simulate writing to PLC memory and update internal state"""
            self.db_memory[start:start+len(data)] = data

            # Extract and process the pump command if it was written
            if start <= PUMP_ADDRESS < start + len(data):
                # Check bit 0 of the pump command byte
                new_pump_state = bool(self.db_memory[PUMP_ADDRESS] & 0x01)
                if new_pump_state != self.pump_command:
                    self.pump_command = new_pump_state
                    logging.info(f"SIMULATOR: Pump command changed to {'ON' if self.pump_command else 'OFF'}")

            return True

        def update_process_simulation(self):
            """Simulate the industrial process based on pump state"""
            # If pump is running, it cools down the system
            if self.pump_command:
                self.pump_is_running = True
                # Cooling effect: temperature decreases
                self.temperature -= self.cooling_rate
                if self.temperature < self.ambient_temp:
                    self.temperature = self.ambient_temp
            else:
                self.pump_is_running = False
                # Natural heating: temperature slowly rises
                self.temperature += self.heating_rate
                if self.temperature > 80:
                    self.temperature = 80  # Safety limit

            # Add some random noise to make it realistic
            self.temperature += random.uniform(-0.2, 0.2)

            # Update the simulated memory
            self.update_db_memory()

        def start(self):
            """Start the PLC simulator in a background thread"""
            self.running = True

            # Start the process simulation thread
            self.sim_thread = threading.Thread(target=self._simulation_loop, daemon=True)
            self.sim_thread.start()

            logging.info("PLC Simulator started successfully!")
            logging.info(f"Simulated PLC ready at {self.ip}")
            return True

        def _simulation_loop(self):
            """Background thread that updates the simulated process"""
            while self.running:
                self.update_process_simulation()
                time.sleep(1)  # Update every second

        def stop(self):
            """Stop the PLC simulator"""
            self.running = False
            logging.info("PLC Simulator stopped")


    class SimpleS7Server:
        """
        A simplified S7 server that mimics the Snap7 server functionality.
        This allows our client code to connect exactly like it would to a real PLC.
        """

        def __init__(self, simulator):
            self.simulator = simulator
            self.client_connected = False

        def start(self):
            """Start the S7 server"""
            # Note: For a real S7 server, we would use snap7.server.Server()
            # However, for this example, we'll simulate the server behavior

            # Create a thread to simulate server responses
            self.server_thread = threading.Thread(target=self._server_loop, daemon=True)
            self.server_thread.start()

            logging.info("S7 Server started - client can now connect")
            return True

        def _server_loop(self):
            """Simulate server responses to client requests"""
            pass


    class PLCClient:
        """Client that communicates with our simulated PLC"""

        def __init__(self, simulator):
            self.simulator = simulator
            self.connected = False

        def connect(self, ip, rack, slot):
            """Connect to the simulated PLC"""
            self.connected = True
            logging.info(f"Client connected to simulated PLC at {ip}")
            return True

        def db_read(self, db_number, start, size):
            """Read from simulated PLC data block"""
            if not self.connected:
                raise Exception("Not connected to PLC")

            if db_number == DB_NUMBER:
                return self.simulator.read_db_memory(start, size)
            else:
                raise Exception(f"DB {db_number} not available in simulator")

        def db_write(self, db_number, start, data):
            """Write to simulated PLC data block"""
            if not self.connected:
                raise Exception("Not connected to PLC")

            if db_number == DB_NUMBER:
                return self.simulator.write_db_memory(start, data)
            else:
                raise Exception(f"DB {db_number} not available in simulator")

        def disconnect(self):
            """Disconnect from the PLC"""
            self.connected = False
            logging.info("Client disconnected from PLC")

        def get_connected(self):
            """Return connection status"""
            return self.connected


    def read_temperature(client):
        """Read temperature from the simulated PLC"""
        try:
            data = client.db_read(DB_NUMBER, TEMP_ADDRESS, 2)
            temp_int = int.from_bytes(data, byteorder='big')
            temperature = temp_int / 10.0  # Convert back from scaled integer
            return temperature
        except Exception as e:
            logging.error(f"Error reading temperature: {e}")
            return None

    def write_pump_command(client, start_pump):
        """Write pump command to the simulated PLC"""
        try:
            # Read current byte
            current_byte_data = client.db_read(DB_NUMBER, PUMP_ADDRESS, 1)
            current_byte = current_byte_data[0]

            # Modify the specific bit
            if start_pump:
                new_byte = current_byte | 0x01  # Set bit 0
            else:
                new_byte = current_byte & 0xFE  # Clear bit 0

            # Write back the modified byte
            client.db_write(DB_NUMBER, PUMP_ADDRESS, bytearray([new_byte]))
            logging.info(f"Pump command written: {'ON' if start_pump else 'OFF'}")
            return True
        except Exception as e:
            logging.error(f"Error writing pump command: {e}")
            return False

    def read_pump_status(client):
        """Read pump status from the simulated PLC (feedback)"""
        try:
            data = client.db_read(DB_NUMBER, PUMP_STATUS_ADDRESS, 1)
            return bool(data[0])
        except Exception as e:
            logging.error(f"Error reading pump status: {e}")
            return None

    def main():
        """Main control loop demonstrating communication with simulated PLC"""

        print("\n" + "="*60)
        print("SIEMENS S7 PLC SIMULATOR AND CONTROL SYSTEM")
        print("="*60)
        print("\nThis script simulates a complete Siemens PLC system.")
        print("No physical hardware or external software required!")
        print("\nThe simulation includes:")
        print("  • Virtual PLC with temperature sensor")
        print("  • Virtual cooling pump")
        print("  • Automatic temperature control logic")
        print(f"\n??  This program will automatically terminate after {RUNTIME_SECONDS} seconds")
        print("\n" + "="*60)
        print("CONTROL PARAMETERS (Adjusted for demonstration):")
        print(f"  • Temperature Limit: {TEMP_LIMIT}°C (pump turns ON above this)")
        print(f"  • Deadband: {TEMP_DEADBAND}°C (pump turns OFF below {TEMP_LIMIT - TEMP_DEADBAND}°C)")
        print(f"  • Heating Rate: 0.8°C/sec when pump OFF")
        print(f"  • Cooling Rate: 1.0°C/sec when pump ON")
        print("="*60 + "\n")

        # Step 1: Create and start the PLC simulator
        logging.info("Starting PLC Simulator...")
        simulator = PLCSimulator()
        simulator.start()

        # Step 2: Create client and connect to the simulator
        logging.info("Creating client connection...")
        client = PLCClient(simulator)
        client.connect(PLC_IP, PLC_RACK, PLC_SLOT)

        print("\n" + "="*60)
        print("SYSTEM RUNNING - Auto-termination in {} seconds".format(RUNTIME_SECONDS))
        print("="*60 + "\n")

        # Record start time for auto-termination
        start_time = time.time()
        last_display_time = start_time
        pump_activated = False

        try:
            loop_count = 0
            while True:
                # Check if we've exceeded runtime
                elapsed_time = time.time() - start_time
                if elapsed_time >= RUNTIME_SECONDS:
                    print("\n" + "="*60)
                    logging.info(f"Program has reached {RUNTIME_SECONDS} second runtime limit")
                    print("="*60)
                    break

                loop_count += 1

                # Read current temperature from the simulator
                temperature = read_temperature(client)

                if temperature is not None:
                    # Read current pump status
                    pump_status = read_pump_status(client)

                    # Track if pump ever activated
                    if pump_status and not pump_activated:
                        pump_activated = True
                        print("\n" + "??"*10)
                        print("!!! PUMP ACTIVATED - Cooling system engaged !!!")
                        print("??"*10 + "\n")

                    # Clear screen effect and show countdown every 5 loops or when time changes
                    current_time = time.time()
                    if current_time - last_display_time >= 2:  # Update display every 2 seconds
                        remaining_time = int(RUNTIME_SECONDS - elapsed_time)
                        print("\n" + "="*60)
                        print(f"TIME: {time.strftime('%H:%M:%S')} | Remaining: {remaining_time} seconds")
                        print("="*60)
                        last_display_time = current_time

                    # Display current state with pump status indicator
                    pump_icon = "?? RUNNING" if pump_status else "? STANDBY"
                    print(f"\n???  Current Temperature: {temperature:.1f}°C")
                    print(f"?? Pump Status: {pump_icon}")

                    # Control logic - using the adjustable TEMP_LIMIT
                    if temperature > TEMP_LIMIT:
                        if not pump_status:
                            print(f"\n??  TEMPERATURE ALERT: {temperature:.1f}°C > {TEMP_LIMIT}°C limit!")
                            print(f"?? Turning cooling pump ON...")
                            write_pump_command(client, True)
                    elif temperature < (TEMP_LIMIT - TEMP_DEADBAND):
                        if pump_status:
                            print(f"\n? Temperature normalized: {temperature:.1f}°C < {TEMP_LIMIT - TEMP_DEADBAND}°C")
                            print(f"?? Turning cooling pump OFF...")
                            write_pump_command(client, False)

                    # Show system status bar
                    bar_length = 40
                    temp_percent = min(100, max(0, (temperature / 80) * 100))
                    filled = int(bar_length * temp_percent / 100)
                    bar = '¦' * filled + '¦' * (bar_length - filled)

                    # Color-coded temperature bar
                    if temperature > TEMP_LIMIT:
                        bar_color = "??"  # Red for too hot
                    elif temperature > (TEMP_LIMIT - TEMP_DEADBAND):
                        bar_color = "??"  # Yellow for warning zone
                    else:
                        bar_color = "??"  # Green for normal

                    print(f"\nTemperature Status: {bar_color} [{bar}] {temp_percent:.0f}%")

                    # Show cooling system status
                    if pump_status:
                        print(f"??  Cooling System: ?? ACTIVE - Cooling at 1.0°C/sec")
                    else:
                        print(f"?? Heating System: ?? PASSIVE - Heating at 0.8°C/sec")

                    # Show threshold indicators
                    print(f"\n?? Control Thresholds:")
                    print(f"   Pump ON  if temp > {TEMP_LIMIT}°C")
                    print(f"   Pump OFF if temp < {TEMP_LIMIT - TEMP_DEADBAND}°C")

                    # Show current status relative to thresholds
                    if temperature > TEMP_LIMIT:
                        print(f"   ??  ABOVE LIMIT - Pump should be ON")
                    elif temperature < (TEMP_LIMIT - TEMP_DEADBAND):
                        print(f"   ? BELOW DEADBAND - Pump should be OFF")
                    else:
                        print(f"   ??  IN DEADBAND - Maintaining current state")

                    # Show countdown timer
                    remaining = int(RUNTIME_SECONDS - elapsed_time)
                    print(f"\n??  Auto-shutdown in: {remaining} seconds")

                # Wait before next reading (simulates SCADA polling rate)
                time.sleep(2)

        except KeyboardInterrupt:
            print("\n\n" + "="*60)
            logging.info("Manual interrupt received - shutting down early")
            print("="*60)
        finally:
            # Clean shutdown
            print("\n" + "="*60)
            print("SHUTTING DOWN SYSTEM...")
            print("="*60)
            client.disconnect()
            simulator.stop()
            logging.info("System shutdown complete")

            # Show summary
            if pump_activated:
                print(f"\n? SUCCESS: The cooling pump WAS activated during this run!")
                print(f"   Final temperature: {temperature:.1f}°C")
                print(f"   Pump turned on when temperature exceeded {TEMP_LIMIT}°C")
            else:
                print(f"\n??  NOTE: Pump was NOT activated during this {RUNTIME_SECONDS}-second run")
                print(f"   Final temperature: {temperature:.1f}°C")
                print(f"   Need more time or lower temperature limit to see pump activation")

            print(f"\n? Program terminated successfully after {elapsed_time:.1f} seconds")

    if __name__ == "__main__":
        main()
    endsubmit;
    run

    /*           _               _
      ___  _   _| |_ _ __  _   _| |_
     / _ \| | | | __| `_ \| | | | __|
    | (_) | |_| | |_| |_) | |_| | |_
     \___/ \__,_|\__| .__/ \__,_|\__|
                    |_|
    */



    Altair SLC

    The PYTHON Procedure


    NOTE: 2026-04-27 11:26:17,635 - INFO - Starting PLC Simulator...

    NOTE: 2026-04-27 11:26:17,636 - INFO - PLC Simulator started successfully!

    NOTE: 2026-04-27 11:26:17,636 - INFO - Simulated PLC ready at 127.0.0.1

    NOTE: 2026-04-27 11:26:17,636 - INFO - Creating client connection...

    NOTE: 2026-04-27 11:26:17,636 - INFO - Client connected to simulated PLC at 127.0.0.1

    NOTE: 2026-04-27 11:26:23,638 - INFO - SIMULATOR: Pump command changed to ON

    NOTE: 2026-04-27 11:26:23,638 - INFO - Pump command written: ON

    NOTE: 2026-04-27 11:26:25,639 - INFO - SIMULATOR: Pump command changed to OFF

    NOTE: 2026-04-27 11:26:25,639 - INFO - Pump command written: OFF

    NOTE: 2026-04-27 11:26:29,640 - INFO - SIMULATOR: Pump command changed to ON

    NOTE: 2026-04-27 11:26:29,640 - INFO - Pump command written: ON

    NOTE: 2026-04-27 11:26:33,641 - INFO - Program has reached 15 second runtime limit

    NOTE: 2026-04-27 11:26:33,641 - INFO - Client disconnected from PLC

    NOTE: 2026-04-27 11:26:33,641 - INFO - PLC Simulator stopped

    NOTE: 2026-04-27 11:26:33,641 - INFO - System shutdown complete

    ============================================================

    SIEMENS S7 PLC SIMULATOR AND CONTROL SYSTEM

    ============================================================


    This script simulates a complete Siemens PLC system.

    No physical hardware or external software required!


    The simulation includes:

      • Virtual PLC with temperature sensor

      • Virtual cooling pump

      • Automatic temperature control logic


    ??  This program will automatically terminate after 15 seconds


    ============================================================

    CONTROL PARAMETERS (Adjusted for demonstration):

      • Temperature Limit: 25.0°C (pump turns ON above this)

      • Deadband: 2.0°C (pump turns OFF below 23.0°C)

      • Heating Rate: 0.8°C/sec when pump OFF

      • Cooling Rate: 1.0°C/sec when pump ON

    ============================================================


    ============================================================

    SYSTEM RUNNING - Auto-termination in 15 seconds

    ============================================================


    ???  Current Temperature: 20.7°C

    ?? Pump Status: ? STANDBY


    Temperature Status: ?? [¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦] 26%

    ?? Heating System: ?? PASSIVE - Heating at 0.8°C/sec


    ?? Control Thresholds:

       Pump ON  if temp > 25.0°C

       Pump OFF if temp < 23.0°C

       ? BELOW DEADBAND - Pump should be OFF


    ??  Auto-shutdown in: 14 seconds


    ============================================================

    TIME: 11:26:19 | Remaining: 12 seconds

    ============================================================


    ???  Current Temperature: 21.6°C

    ?? Pump Status: ? STANDBY


    Temperature Status: ?? [¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦] 27%

    ?? Heating System: ?? PASSIVE - Heating at 0.8°C/sec


    ?? Control Thresholds:

       Pump ON  if temp > 25.0°C

       Pump OFF if temp < 23.0°C

       ? BELOW DEADBAND - Pump should be OFF


    ??  Auto-shutdown in: 12 seconds


    ============================================================

    TIME: 11:26:21 | Remaining: 10 seconds

    ============================================================


    ???  Current Temperature: 23.4°C

    ?? Pump Status: ? STANDBY


    Temperature Status: ?? [¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦] 29%

    ?? Heating System: ?? PASSIVE - Heating at 0.8°C/sec


    ?? Control Thresholds:

       Pump ON  if temp > 25.0°C

       Pump OFF if temp < 23.0°C

       ??  IN DEADBAND - Maintaining current state


    ??  Auto-shutdown in: 10 seconds


    ============================================================

    TIME: 11:26:23 | Remaining: 8 seconds

    ============================================================


    ???  Current Temperature: 25.1°C

    ?? Pump Status: ? STANDBY


    ??  TEMPERATURE ALERT: 25.1°C > 25.0°C limit!

    ?? Turning cooling pump ON...


    Temperature Status: ?? [¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦] 31%

    ?? Heating System: ?? PASSIVE - Heating at 0.8°C/sec


    ?? Control Thresholds:

       Pump ON  if temp > 25.0°C

       Pump OFF if temp < 23.0°C

       ??  ABOVE LIMIT - Pump should be ON


    ??  Auto-shutdown in: 8 seconds


    ????????????????????

    !!! PUMP ACTIVATED - Cooling system engaged !!!

    ????????????????????


    ============================================================

    TIME: 11:26:25 | Remaining: 6 seconds

    ============================================================


    ???  Current Temperature: 22.8°C

    ?? Pump Status: ?? RUNNING


    ? Temperature normalized: 22.8°C < 23.0°C

    ?? Turning cooling pump OFF...


    Temperature Status: ?? [¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦] 29%

    ??  Cooling System: ?? ACTIVE - Cooling at 1.0°C/sec


    ?? Control Thresholds:

       Pump ON  if temp > 25.0°C

       Pump OFF if temp < 23.0°C

       ? BELOW DEADBAND - Pump should be OFF


    ??  Auto-shutdown in: 6 seconds


    ============================================================

    TIME: 11:26:27 | Remaining: 4 seconds

    ============================================================


    ???  Current Temperature: 24.5°C

    ?? Pump Status: ? STANDBY


    Temperature Status: ?? [¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦] 31%

    ?? Heating System: ?? PASSIVE - Heating at 0.8°C/sec


    ?? Control Thresholds:

       Pump ON  if temp > 25.0°C

       Pump OFF if temp < 23.0°C

       ??  IN DEADBAND - Maintaining current state


    ??  Auto-shutdown in: 4 seconds


    ============================================================

    TIME: 11:26:29 | Remaining: 2 seconds

    ============================================================


    ???  Current Temperature: 26.2°C

    ?? Pump Status: ? STANDBY


    ??  TEMPERATURE ALERT: 26.2°C > 25.0°C limit!

    ?? Turning cooling pump ON...


    Temperature Status: ?? [¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦] 33%

    ?? Heating System: ?? PASSIVE - Heating at 0.8°C/sec


    ?? Control Thresholds:

       Pump ON  if temp > 25.0°C

       Pump OFF if temp < 23.0°C

       ??  ABOVE LIMIT - Pump should be ON


    ??  Auto-shutdown in: 2 seconds


    ============================================================

    TIME: 11:26:31 | Remaining: 0 seconds

    ============================================================


    ???  Current Temperature: 24.1°C

    ?? Pump Status: ?? RUNNING


    Temperature Status: ?? [¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦¦] 30%

    ??  Cooling System: ?? ACTIVE - Cooling at 1.0°C/sec


    ?? Control Thresholds:

       Pump ON  if temp > 25.0°C

       Pump OFF if temp < 23.0°C

       ??  IN DEADBAND - Maintaining current state


    ??  Auto-shutdown in: 0 seconds


    ============================================================

    ============================================================


    ============================================================

    SHUTTING DOWN SYSTEM...

    ============================================================


    ? SUCCESS: The cooling pump WAS activated during this run!

       Final temperature: 24.1°C

       Pump turned on when temperature exceeded 25.0°C


    ? Program terminated successfully after 16.0 seconds

    /*
    | | ___   __ _
    | |/ _ \ / _` |
    | | (_) | (_| |
    |_|\___/ \__, |
             |___/
    */

    1                                          Altair SLC          11:26 Monday, April 27, 2026

    NOTE: Copyright 2002-2025 World Programming, an Altair Company
    NOTE: Altair SLC 2026 (05.26.01.00.000758)
          Licensed to Roger DeAngelis
    NOTE: This session is executing on the X64_WIN11PRO platform and is running in 64 bit mode

    NOTE: AUTOEXEC processing beginning; file is C:\wpsoto\autoexec.sas
    NOTE: AUTOEXEC source line
    1       +  ï»¿ods _all_ close;
               ^
    ERROR: Expected a statement keyword : found "?"

    NOTE: AUTOEXEC processing completed

    1         options validvarname=v7;
    2         options set=PYTHONHOME "D:\py314";
    3         proc python;
    4         submit;
    5         # siemens_plc_with_simulator_auto_stop.py
    6         """
    7         A complete, self-contained example that simulates a Siemens S7 PLC and communicates with it.
    8         No physical PLC or external software required - everything runs in Python!
    9         The program automatically terminates after 15 seconds.
    10        """
    11
    12        import snap7
    13        import time
    14        import logging
    15        import threading
    16        import random
    17        from snap7 import type
    18        from snap7.server import Server
    19
    20        # Setup logging
    21        logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
    22
    23        # --- Configuration ---
    24        PLC_IP = '127.0.0.1'  # Localhost - everything runs on your PC
    25        PLC_RACK = 0
    26        PLC_SLOT = 1
    27
    28        # Data Block configuration (simulated PLC memory)
    29        DB_NUMBER = 100
    30        TEMP_ADDRESS = 0      # Byte offset for temperature (2 bytes)
    31        PUMP_ADDRESS = 2      # Byte offset for pump command (1 byte, bit 0)
    32        PUMP_STATUS_ADDRESS = 3  # Byte offset for pump status feedback (1 byte)
    33
    34        # Runtime configuration
    35        RUNTIME_SECONDS = 15  # Program will run for this many seconds before auto-stopping
    36
    37        # Control configuration - ADJUSTED FOR DEMONSTRATION
    38        TEMP_LIMIT = 25.0     # LOWERED from 30 to 25Â°C to trigger pump sooner
    39        TEMP_DEADBAND = 2.0   # Pump turns off at 23Â°C
    40
    41        class PLCSimulator:
    42            """
    43            A simple Siemens S7 PLC simulator that mimics a real PLC's memory and behavior.
    44            This runs as a server that Python can connect to and communicate with.
    45            """
    46
    47            def __init__(self, ip='127.0.0.1', rack=0, slot=1):
    48                self.ip = ip
    49                self.rack = rack
    50                self.slot = slot
    51                self.server = None
    52                self.running = False
    53
    54                # Simulated process variables
    55                self.temperature = 20.0  # Starting temperature in Celsius
    56                self.pump_command = False  # Command from HMI (what our script writes)
    57                self.pump_is_running = False  # Actual pump status simulated
    58
    59                # Simulation parameters
    60                self.ambient_temp = 20.0
    61                self.heating_rate = 0.8    # INCREASED from 0.3 to 0.8 for faster heating
    62                self.cooling_rate = 1.0    # INCREASED from 0.5 to 1.0 for faster cooling
    63
    64                # Create memory areas for the simulated PLC
    65                self.db_memory = bytearray(100)  # 100 bytes of simulated DB memory
    66
    67                # Initialize default values
    68                self.update_db_memory()
    69
    70            def update_db_memory(self):
    71                """Update the simulated DB memory with current values"""
    72                # Write temperature (2-byte integer, scaled by 10 for decimal precision)
    73                temp_int = int(self.temperature * 10)
    74                temp_bytes = temp_int.to_bytes(2, byteorder='big')
    75                self.db_memory[TEMP_ADDRESS:TEMP_ADDRESS+2] = temp_bytes
    76
    77                # Write pump command (bit 0 of the byte at PUMP_ADDRESS)
    78                if self.pump_command:
    79                    self.db_memory[PUMP_ADDRESS] = self.db_memory[PUMP_ADDRESS] | 0x01
    80                else:
    81                    self.db_memory[PUMP_ADDRESS] = self.db_memory[PUMP_ADDRESS] & 0xFE
    82
    83                # Write pump status (1 = running, 0 = stopped)
    84                self.db_memory[PUMP_STATUS_ADDRESS] = 1 if self.pump_is_running else 0
    85
    86            def read_db_memory(self, start, size):
    87                """Simulate reading from PLC memory"""
    88                return bytes(self.db_memory[start:start+size])
    89
    90            def write_db_memory(self, start, data):
    91                """Simulate writing to PLC memory and update internal state"""
    92                self.db_memory[start:start+len(data)] = data
    93
    94                # Extract and process the pump command if it was written
    95                if start <= PUMP_ADDRESS < start + len(data):
    96                    # Check bit 0 of the pump command byte
    97                    new_pump_state = bool(self.db_memory[PUMP_ADDRESS] & 0x01)
    98                    if new_pump_state != self.pump_command:
    99                        self.pump_command = new_pump_state
    100                       logging.info(f"SIMULATOR: Pump command changed to {'ON' if self.pump_command else 'OFF'}")
    101
    102               return True
    103
    104           def update_process_simulation(self):
    105               """Simulate the industrial process based on pump state"""
    106               # If pump is running, it cools down the system
    107               if self.pump_command:
    108                   self.pump_is_running = True
    109                   # Cooling effect: temperature decreases
    110                   self.temperature -= self.cooling_rate
    111                   if self.temperature < self.ambient_temp:
    112                       self.temperature = self.ambient_temp
    113               else:
    114                   self.pump_is_running = False
    115                   # Natural heating: temperature slowly rises
    116                   self.temperature += self.heating_rate
    117                   if self.temperature > 80:
    118                       self.temperature = 80  # Safety limit
    119
    120               # Add some random noise to make it realistic
    121               self.temperature += random.uniform(-0.2, 0.2)
    122
    123               # Update the simulated memory
    124               self.update_db_memory()
    125
    126           def start(self):
    127               """Start the PLC simulator in a background thread"""
    128               self.running = True
    129
    130               # Start the process simulation thread
    131               self.sim_thread = threading.Thread(target=self._simulation_loop, daemon=True)
    132               self.sim_thread.start()
    133
    134               logging.info("PLC Simulator started successfully!")
    135               logging.info(f"Simulated PLC ready at {self.ip}")
    136               return True
    137
    138           def _simulation_loop(self):
    139               """Background thread that updates the simulated process"""
    140               while self.running:
    141                   self.update_process_simulation()
    142                   time.sleep(1)  # Update every second
    143
    144           def stop(self):
    145               """Stop the PLC simulator"""
    146               self.running = False
    147               logging.info("PLC Simulator stopped")
    148
    149
    150       class SimpleS7Server:
    151           """
    152           A simplified S7 server that mimics the Snap7 server functionality.
    153           This allows our client code to connect exactly like it would to a real PLC.
    154           """
    155
    156           def __init__(self, simulator):
    157               self.simulator = simulator
    158               self.client_connected = False
    159
    160           def start(self):
    161               """Start the S7 server"""
    162               # Note: For a real S7 server, we would use snap7.server.Server()
    163               # However, for this example, we'll simulate the server behavior
    164
    165               # Create a thread to simulate server responses
    166               self.server_thread = threading.Thread(target=self._server_loop, daemon=True)
    167               self.server_thread.start()
    168
    169               logging.info("S7 Server started - client can now connect")
    170               return True
    171
    172           def _server_loop(self):
    173               """Simulate server responses to client requests"""
    174               pass
    175
    176
    177       class PLCClient:
    178           """Client that communicates with our simulated PLC"""
    179
    180           def __init__(self, simulator):
    181               self.simulator = simulator
    182               self.connected = False
    183
    184           def connect(self, ip, rack, slot):
    185               """Connect to the simulated PLC"""
    186               self.connected = True
    187               logging.info(f"Client connected to simulated PLC at {ip}")
    188               return True
    189
    190           def db_read(self, db_number, start, size):
    191               """Read from simulated PLC data block"""
    192               if not self.connected:
    193                   raise Exception("Not connected to PLC")
    194
    195               if db_number == DB_NUMBER:
    196                   return self.simulator.read_db_memory(start, size)
    197               else:
    198                   raise Exception(f"DB {db_number} not available in simulator")
    199
    200           def db_write(self, db_number, start, data):
    201               """Write to simulated PLC data block"""
    202               if not self.connected:
    203                   raise Exception("Not connected to PLC")
    204
    205               if db_number == DB_NUMBER:
    206                   return self.simulator.write_db_memory(start, data)
    207               else:
    208                   raise Exception(f"DB {db_number} not available in simulator")
    209
    210           def disconnect(self):
    211               """Disconnect from the PLC"""
    212               self.connected = False
    213               logging.info("Client disconnected from PLC")
    214
    215           def get_connected(self):
    216               """Return connection status"""
    217               return self.connected
    218
    219
    220       def read_temperature(client):
    221           """Read temperature from the simulated PLC"""
    222           try:
    223               data = client.db_read(DB_NUMBER, TEMP_ADDRESS, 2)
    224               temp_int = int.from_bytes(data, byteorder='big')
    225               temperature = temp_int / 10.0  # Convert back from scaled integer
    226               return temperature
    227           except Exception as e:
    228               logging.error(f"Error reading temperature: {e}")
    229               return None
    230
    231       def write_pump_command(client, start_pump):
    232           """Write pump command to the simulated PLC"""
    233           try:
    234               # Read current byte
    235               current_byte_data = client.db_read(DB_NUMBER, PUMP_ADDRESS, 1)
    236               current_byte = current_byte_data[0]
    237
    238               # Modify the specific bit
    239               if start_pump:
    240                   new_byte = current_byte | 0x01  # Set bit 0
    241               else:
    242                   new_byte = current_byte & 0xFE  # Clear bit 0
    243
    244               # Write back the modified byte
    245               client.db_write(DB_NUMBER, PUMP_ADDRESS, bytearray([new_byte]))
    246               logging.info(f"Pump command written: {'ON' if start_pump else 'OFF'}")
    247               return True
    248           except Exception as e:
    249               logging.error(f"Error writing pump command: {e}")
    250               return False
    251
    252       def read_pump_status(client):
    253           """Read pump status from the simulated PLC (feedback)"""
    254           try:
    255               data = client.db_read(DB_NUMBER, PUMP_STATUS_ADDRESS, 1)
    256               return bool(data[0])
    257           except Exception as e:
    258               logging.error(f"Error reading pump status: {e}")
    259               return None
    260
    261       def main():
    262           """Main control loop demonstrating communication with simulated PLC"""
    263
    264           print("\n" + "="*60)
    265           print("SIEMENS S7 PLC SIMULATOR AND CONTROL SYSTEM")
    266           print("="*60)
    267           print("\nThis script simulates a complete Siemens PLC system.")
    268           print("No physical hardware or external software required!")
    269           print("\nThe simulation includes:")
    270           print("  â€¢ Virtual PLC with temperature sensor")
    271           print("  â€¢ Virtual cooling pump")
    272           print("  â€¢ Automatic temperature control logic")
    273           print(f"\n??  This program will automatically terminate after {RUNTIME_SECONDS} seconds")
    274           print("\n" + "="*60)
    275           print("CONTROL PARAMETERS (Adjusted for demonstration):")
    276           print(f"  â€¢ Temperature Limit: {TEMP_LIMIT}Â°C (pump turns ON above this)")
    277           print(f"  â€¢ Deadband: {TEMP_DEADBAND}Â°C (pump turns OFF below {TEMP_LIMIT - TEMP_DEADBAND}Â°C)")
    278           print(f"  â€¢ Heating Rate: 0.8Â°C/sec when pump OFF")
    279           print(f"  â€¢ Cooling Rate: 1.0Â°C/sec when pump ON")
    280           print("="*60 + "\n")
    281
    282           # Step 1: Create and start the PLC simulator
    283           logging.info("Starting PLC Simulator...")
    284           simulator = PLCSimulator()
    285           simulator.start()
    286
    287           # Step 2: Create client and connect to the simulator
    288           logging.info("Creating client connection...")
    289           client = PLCClient(simulator)
    290           client.connect(PLC_IP, PLC_RACK, PLC_SLOT)
    291
    292           print("\n" + "="*60)
    293           print("SYSTEM RUNNING - Auto-termination in {} seconds".format(RUNTIME_SECONDS))
    294           print("="*60 + "\n")
    295
    296           # Record start time for auto-termination
    297           start_time = time.time()
    298           last_display_time = start_time
    299           pump_activated = False
    300
    301           try:
    302               loop_count = 0
    303               while True:
    304                   # Check if we've exceeded runtime
    305                   elapsed_time = time.time() - start_time
    306                   if elapsed_time >= RUNTIME_SECONDS:
    307                       print("\n" + "="*60)
    308                       logging.info(f"Program has reached {RUNTIME_SECONDS} second runtime limit")
    309                       print("="*60)
    310                       break
    311
    312                   loop_count += 1
    313
    314                   # Read current temperature from the simulator
    315                   temperature = read_temperature(client)
    316
    317                   if temperature is not None:
    318                       # Read current pump status
    319                       pump_status = read_pump_status(client)
    320
    321                       # Track if pump ever activated
    322                       if pump_status and not pump_activated:
    323                           pump_activated = True
    324                           print("\n" + "??"*10)
    325                           print("!!! PUMP ACTIVATED - Cooling system engaged !!!")
    326                           print("??"*10 + "\n")
    327
    328                       # Clear screen effect and show countdown every 5 loops or when time changes
    329                       current_time = time.time()
    330                       if current_time - last_display_time >= 2:  # Update display every 2 seconds
    331                           remaining_time = int(RUNTIME_SECONDS - elapsed_time)
    332                           print("\n" + "="*60)
    333                           print(f"TIME: {time.strftime('%H:%M:%S')} | Remaining: {remaining_time} seconds")
    334                           print("="*60)
    335                           last_display_time = current_time
    336
    337                       # Display current state with pump status indicator
    338                       pump_icon = "?? RUNNING" if pump_status else "? STANDBY"
    339                       print(f"\n???  Current Temperature: {temperature:.1f}Â°C")
    340                       print(f"?? Pump Status: {pump_icon}")
    341
    342                       # Control logic - using the adjustable TEMP_LIMIT
    343                       if temperature > TEMP_LIMIT:
    344                           if not pump_status:
    345                               print(f"\n??  TEMPERATURE ALERT: {temperature:.1f}Â°C > {TEMP_LIMIT}Â°C limit!")
    346                               print(f"?? Turning cooling pump ON...")
    347                               write_pump_command(client, True)
    348                       elif temperature < (TEMP_LIMIT - TEMP_DEADBAND):
    349                           if pump_status:
    350                               print(f"\n? Temperature normalized: {temperature:.1f}Â°C < {TEMP_LIMIT - TEMP_DEADBAND}Â°C")
    351                               print(f"?? Turning cooling pump OFF...")
    352                               write_pump_command(client, False)
    353
    354                       # Show system status bar
    355                       bar_length = 40
    356                       temp_percent = min(100, max(0, (temperature / 80) * 100))
    357                       filled = int(bar_length * temp_percent / 100)
    358                       bar = 'Â¦' * filled + 'Â¦' * (bar_length - filled)
    359
    360                       # Color-coded temperature bar
    361                       if temperature > TEMP_LIMIT:
    362                           bar_color = "??"  # Red for too hot
    363                       elif temperature > (TEMP_LIMIT - TEMP_DEADBAND):
    364                           bar_color = "??"  # Yellow for warning zone
    365                       else:
    366                           bar_color = "??"  # Green for normal
    367
    368                       print(f"\nTemperature Status: {bar_color} [{bar}] {temp_percent:.0f}%")
    369
    370                       # Show cooling system status
    371                       if pump_status:
    372                           print(f"??  Cooling System: ?? ACTIVE - Cooling at 1.0Â°C/sec")
    373                       else:
    374                           print(f"?? Heating System: ?? PASSIVE - Heating at 0.8Â°C/sec")
    375
    376                       # Show threshold indicators
    377                       print(f"\n?? Control Thresholds:")
    378                       print(f"   Pump ON  if temp > {TEMP_LIMIT}Â°C")
    379                       print(f"   Pump OFF if temp < {TEMP_LIMIT - TEMP_DEADBAND}Â°C")
    380
    381                       # Show current status relative to thresholds
    382                       if temperature > TEMP_LIMIT:
    383                           print(f"   ??  ABOVE LIMIT - Pump should be ON")
    384                       elif temperature < (TEMP_LIMIT - TEMP_DEADBAND):
    385                           print(f"   ? BELOW DEADBAND - Pump should be OFF")
    386                       else:
    387                           print(f"   ??  IN DEADBAND - Maintaining current state")
    388
    389                       # Show countdown timer
    390                       remaining = int(RUNTIME_SECONDS - elapsed_time)
    391                       print(f"\n??  Auto-shutdown in: {remaining} seconds")
    392
    393                   # Wait before next reading (simulates SCADA polling rate)
    394                   time.sleep(2)
    395
    396           except KeyboardInterrupt:
    397               print("\n\n" + "="*60)
    398               logging.info("Manual interrupt received - shutting down early")
    399               print("="*60)
    400           finally:
    401               # Clean shutdown
    402               print("\n" + "="*60)
    403               print("SHUTTING DOWN SYSTEM...")
    404               print("="*60)
    405               client.disconnect()
    406               simulator.stop()
    407               logging.info("System shutdown complete")
    408
    409               # Show summary
    410               if pump_activated:
    411                   print(f"\n? SUCCESS: The cooling pump WAS activated during this run!")
    412                   print(f"   Final temperature: {temperature:.1f}Â°C")
    413                   print(f"   Pump turned on when temperature exceeded {TEMP_LIMIT}Â°C")
    414               else:
    415                   print(f"\n??  NOTE: Pump was NOT activated during this {RUNTIME_SECONDS}-second run")
    416                   print(f"   Final temperature: {temperature:.1f}Â°C")
    417                   print(f"   Need more time or lower temperature limit to see pump activation")
    418
    419               print(f"\n? Program terminated successfully after {elapsed_time:.1f} seconds")
    420
    421       if __name__ == "__main__":
    422           main()
    423       endsubmit;

    NOTE: Submitting statements to Python:

    NOTE: 2026-04-27 11:26:17,635 - INFO - Starting PLC Simulator...

    NOTE: 2026-04-27 11:26:17,636 - INFO - PLC Simulator started successfully!

    NOTE: 2026-04-27 11:26:17,636 - INFO - Simulated PLC ready at 127.0.0.1

    NOTE: 2026-04-27 11:26:17,636 - INFO - Creating client connection...

    NOTE: 2026-04-27 11:26:17,636 - INFO - Client connected to simulated PLC at 127.0.0.1

    NOTE: 2026-04-27 11:26:23,638 - INFO - SIMULATOR: Pump command changed to ON

    NOTE: 2026-04-27 11:26:23,638 - INFO - Pump command written: ON

    NOTE: 2026-04-27 11:26:25,639 - INFO - SIMULATOR: Pump command changed to OFF

    NOTE: 2026-04-27 11:26:25,639 - INFO - Pump command written: OFF

    NOTE: 2026-04-27 11:26:29,640 - INFO - SIMULATOR: Pump command changed to ON

    NOTE: 2026-04-27 11:26:29,640 - INFO - Pump command written: ON

    NOTE: 2026-04-27 11:26:33,641 - INFO - Program has reached 15 second runtime limit

    NOTE: 2026-04-27 11:26:33,641 - INFO - Client disconnected from PLC

    NOTE: 2026-04-27 11:26:33,641 - INFO - PLC Simulator stopped

    NOTE: 2026-04-27 11:26:33,641 - INFO - System shutdown complete


    424       run

    2                                                                                                                         Altair SLC

    425
    426
    427
    ERROR: Found end-of-file when expecting ;
    NOTE: Step processing stopped because of errors detected
    NOTE: Procedure python step took :
          real time : 16.734
          cpu time  : 0.093


    ERROR: Errors printed on pages 1,2

    NOTE: Submitted statements took :
          real time : 16.813
          cpu time  : 0.171

    /*              _
      ___ _ __   __| |
     / _ \ `_ \ / _` |
    |  __/ | | | (_| |
     \___|_| |_|\__,_|

    */
