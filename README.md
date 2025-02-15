# yamcs_link

Python library to facilitate creating applications interacting with a YAMCS server. Targets simple applications to speed up getting started and avoid reinventing the wheel.

Example application:
```
from yamcs_userlib import YAMCSObject, telemetry, telecommand, U8, U16, F32
from enum import Enum

#Constants
YAMCS_TC_PORT = 10000
YAMCS_TM_PORT = 10001
DIR_MDB_OUT = "/mdb_shared"
MDB_NAME = "test"
VERSION = "1.0"

#Dummy example app definition

class MyEnum(Enum):
    VALUE1=1
    VALUE2=2

class MyComponent(YAMCSObject):
    def __init__(self, name):
        YAMCSObject.__init__(self, name)

    @telemetry(1) #seconds period
    def my_telemetry1(self) -> MyEnum:
        return MyEnum.VALUE1.value

    @telemetry(2) #seconds period
    def my_telemetry2(self) -> U8:
        return 42

    @telecommand
    def my_command(self, arg1: U16, arg2: F32) -> U8:
        logging.info(f'MyComponent.my_command was invoked on {self.yamcs_name} with args {arg1}, {arg2}')
        return 0
    
#initialisation
yamcs_link = YAMCS_link("my_link", tcp_port=YAMCS_TC_PORT, udp_port=YAMCS_TM_PORT) 
my_component = MyComponent("component1")
yamcs_link.register_yamcs_child(my_component)

#Generate mdb is necessary for YAMCS to know how to interact with the app. 
# Make sure there is an automated process for yamcs to start up from those updated mdb
yamcs_link.generate_mdb(DIR_MDB_OUT, MDB_NAME, VERSION) 
#If the mdb is generated through some other scheme, manually do call update_index() between register_yamcs_child() and service()

# Main loop
try:
    while True:
        #Possible to do things here e.g. if the app doesn't inherit from YAMCS_link
        yamcs_link.service() #Send due TM and process pending command then return
        time.sleep(0.1) #Small delay to prevent busy-waiting
except KeyboardInterrupt:
    logging.info("Exiting main loop.")
finally:
    yamcs_link.shutdown() 
```

YAMCS_link runs the application, and any object inheriting from either YAMCSObject or YAMCSContainer can be registered in the link to attach its method tagged with the @telemetry or @telecommand decorators. Use YAMCSContainers if the application has hierarchical components that should be reflected in YAMCS, YAMCSObjects everywhere else. As long as the register_yamcs_child() chain is not broken between YAMCS_link and components, they will be connected to YAMCS. 

Events are the natural next improvement which will be added later on as @event methods which are called from within the components to trigger events.  

## Getting started
 
- To develop a new application, using the files from yamcs_link/src standalone is the least involved way to get started. There are no dependencies required (tested with python 3.14), and yamcs_link.py contains an example of main program if run directly. However, a properly configured YAMCS instance will be needed for the script to succeed.
- To test the demonstration script in yamcs_link.py, simply run run.sh at the root of the repository. [docker](https://docs.docker.com/engine/install) and [docker compose](https://docs.docker.com/compose/install/linux/#install-using-the-repository) are required. A YAMCS instance will be deployed, with which it will be possible to interact from any browser at 127.0.0.1:8090. YAMCS Studio can also be used instead of the web client. 

## Notes

Note that the YAMCS mission databases are generated as CSV files, which are then compiled into a single XLS by the yamcs_server application. The server application waits for the CSV files to appear in their dedicated folder (in a shared volume between both containers), then proceeds to launching YAMCS. 