# ooi-config
Configuration files (including instrument agent driver startup)

## Cabled Services

Cabled services are mainly managed on the Portland (West Coast) servers. The provide a number of utilities to control and manage data from the cabled arrays (Regional Scale Nodes and Coastal Endurance). 

Services are configured using the supervisor utility. The configuration can usually be found on the listed machine under the `~/supervisors` directory. A configuration file provides the details for managing the services (e.g. `pa1.conf`). 

### Initial Startup
Initial configuration of supervisor requires running the supervisor daemon which must be run under the port_agent environment:

```bash
/home/asadev/anaconda/envs/port_agent/bin/python /home/asadev/anaconda/envs/port_agent/bin/supervisord -c pa2.conf
```

### Maintenance
After `supervisord` is running on the system, `supervisorctl` can be used to control the services. 
For example, to configure the services on portland-01:

```bash
cd ~/supervisors/pa1
source activate instruments
supervisorctl -c pa1.conf
> start pa_monitor
> quit
```

### supervisor configuration

Supervisor processes are run on the following machines using the given configurations:

| location | configuration | purpose |
| -------- | ------------- | ------- |
| portland-01 | pa1/pa1.conf | port agents |
| portland-02 | pa2/pa2.conf | port agents |
| portland-03 | oms_aa_server/oms_aa_svr.conf | Alerts and Alarms |
| portland-03 | ia/services.conf | InstrumentAgent & MissionExecution |
| portland-04 | drivers/pad_drivers.conf | platform engineering streams |
| portland-05 | drivers/drivers.conf | instrument drivers |
| portland-06 | *drivers/drivers.conf* | *instrument drivers - backup* |
| antelope | port_agent_supervisor/antelope.conf | port agents |

The services run by supervisor are listed below:

| name | service | purpose | location |
| ---- | ------- | ------- | -------- |
| *reference designator* | port_agent | communication link between specific instrument and its corresponding instrument agent driver | portland-01 portland-02 antelope |
| *reference designator* | run_driver | instrument driver | portland-05 *portland-06* |
| events_shovel | shovel |  | portland-01 |
| pa_monitor | ooi_port_agent.tools.monitor | email notification service if network connectivity is lost to instruments | portland-01 portland-02 |
| oms_alerts_shovel | shovel | forwards OMS alerts/alarms to production server | portland-01 |
| oms_extractor | | creates engineering particles from engineering ports | portland-04 |
| omsaasvr | oms_alert_alarms_server | creates alert/alarms from OMS events | portland-03 |
| pa_stats_shovel | shovel | forwards statistics of port agent traffic to production server | portland-01 portland-02 |
| particles_shovel | shovel | forwards instrument packets to production server | portland-01 |

### port_agent

Port agent files control the flow of data between the instruments and the instrument agent drivers. This allows for both streaming data from the instrument into the system as well as command and control of the instruments by the driver. Each instrument has a dedicated port agent. The reference designator is used to identify the port agent in the supervisor utility. For example, `stop RS01SBPS-PC01A-07-CAMDSC102` will stop the port agent for the digital camera. Note that the VADCP and the MASSP require additional port agents; `RS01SBPS-PC01A-06-VADCPA101_5` manages the interface for the 5th beam, `-MCU` `-TURBO` `-RGA` manage the controls for the individual units on the MASSP. Port agents are distributed across two servers. The port agent is configured with the instrument address and reference designator. For example the CTDBPO108 is started with the following command:

`port_agent rsn 10.33.12.6 2101 2102 CE04OSBP-LJ01C-06-CTDBPO108`

### run_driver

The `run_driver` script configures and runs an instrument agent driver that manages the connection to a specific instrument. The instrument agent driver provides parsing of the live instrument feed as well as command and control (C2) of the instrument. C2 is currently disabled. The driver configuration includes the driver code, reference designator, events and particles queue and configuration. For example: NUTNRA102 is configured with the following command:

`run_driver mi.instrument.satlantic.suna_deep.ooicore.driver InstrumentDriver CE04OSPS-SF01B-4A-NUTNRA102 amqp://ooi-production:T1cZp68WanMZ53dQ@192.168.150.101/production?queue=ha.instrument_events amqp://ooi-production:T1cZp68WanMZ53dQ@192.168.150.101/production?queue=ha.instrument_particles /home/asadev/ooi-config-master/CE04OSPS-SF01B-4A-NUTNRA102.yaml`

Startup requires a YAML configuration file that can be found in `ooi-config/instrument_driver_startup`. Currently the yaml files are installed at portland-05:~/ooi-config-master (a link to ~/ooi-config/instrument_driver_startup). 

### events_shovel

Forwards instrument events to uFrame. Executes the following command:

`shovel "amqp://ooi-production:T1cZp68WanMZ53dQ@localhost/production" ha.instrument_events ha.instrument_events qpid://ooiufs03.ooi.rutgers.edu Ingest.instrument_events`

### pa_monitor

The port agent monitor provides email notification service in the event that network connectivity is lost to an instrument. The `monitor.conf` file is use to identify the email recipients and which connections are to be monitored. This service checks connectivity using `ping` hourly and only notifies when there is a failure. Executes the following command:

`python -m ooi_port_agent.tools.monitor monitor.conf`

### oms_alerts_shovel

Forwards OMS alerts (generated by `oms_extractor`) to production alert ingest path:

`shovel "amqp://ooi-production:T1cZp68WanMZ53dQ@localhost/production" ha.oms_alerts ha.instrument_events qpid://ooiufs03.ooi.rutgers.edu Ingest.instrument_oms_events`

### oms_extractor

Generates engineering particles from the platform engineering elements. Executes the following command:

`oms_extractor oms_extract_config.yml`

The YAML file references the node configuration files which are used to specify the mapping of each engineering
node to the corresponding OMS endpoint. These files are managed in the mi-instrument project, c.f. https://github.com/oceanobservatories/mi-instrument/tree/master/mi/platform/rsn/node_config_files

### omsaasvr

Converts OMS messages into OOI alert/alarms. Executes the following command:

`oms_alert_alarm_server.py omsaasvr_p1.cfg`

### pa_stats_shovel

Forwards instrument particle statistics (kB/sec) to uFrame. Executes the following command:

`shovel "amqp://ooi-production:T1cZp68WanMZ53dQ@localhost/production" port_agent_stats port_agent_stats qpid://uft20.ooi.rutgers.edu port_agent_stats`

### particles_shovel

Ingests live instrument data particles into OOI. Executes the following command:

`shovel "amqp://ooi-production:T1cZp68WanMZ53dQ@localhost/production" ha.instrument_particles ha.instrument_particles qpid://ooiufs03.ooi.rutgers.edu Ingest.instrument_particles`

## Cabled Test Services

| name | service | purpose | location |
| ---- | ------- | ------- | -------- |
| particles_shovel_uft4 | shovel | forwards instrument particles to test server (uframe-4-test) | portland-01 |
| particles_shovel_uft20 | shovel | forwards instrument particles to test server (uft20) | portland-02 |
| uframe_test_shovel | shovel | forwards instrument particles to test server (uft20) | portland-02 |
| uframe_2_test_shovel | shovel | forwards instrument particles to test server (uframe-2-test) | portland-02 |
| uframe_3_test_shovel | shovel | forwards instrument particles to test server (uframe-3-test) | portland-02 |

### particles_shovel_uft4

Test for ingesting live cabled instrument data to the test machine, `uframe-4-test`. Executes the following command:

`shovel "amqp://ooi-production:T1cZp68WanMZ53dQ@localhost/production" uft4_instrument_particles ha.instrument_particles qpid://uframe-4-test.ooi.rutgers.edu Ingest.instrument_particles`

### particles_shovel_uft20

Test for forwarding live cabled messages to the test machine, `uft20` which may no longer be operational.

### oms_alerts_shovel

Test for broadcasting alerts to test UI. Executes the following command:

`shovel "amqp://ooi-production:T1cZp68WanMZ53dQ@localhost/production" ha.oms_alerts ha.instrument_events qpid://ooiufs03.ooi.rutgers.edu Ingest.instrument_oms_events`
