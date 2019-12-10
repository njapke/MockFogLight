# Geobroker Example

This project deploys an example application of [geobroker](https://github.com/MoeweX/geobroker) using [MockFog Light](https://github.com/OpenFogStack/MockFogLight).
One broker instance and two client instances are rolled out using docker images of the Geobroker Server and Geobroker MultiFileClient respectively.
The example data used by the client instances was generated using the [IoT Dataset Generator](https://github.com/MoeweX/IoTDSG).
As this whole project is a fork of MockFog Light, the setup and requirements are the same. Changes have only been applied to make Geobroker run
smoothly, and to remove some parts not needed by it. Changes are documented where appropriate.

## MockFog Light Setup

### Requirements
Before running any roles, you should setup a virtualenv and install all requirements

```fish
virtualenv venv
source venv/bin/activate.fish
pip install -r requirements.txt
```

In addition:
- aws credentials configured in `~/.aws/credentials`
- valid AWS ssh key stored at `./mockfog.pem`
- read the individual READMEs for each role (e.g. MockFog_Topology) -> do what they say

### Testbed Definition

The testbed has been changed to run one broker instance and two client instances, which connect to the broker.
It can be changed using the following steps:

- configure the topology via `testbed/topologies.py`
- create the testbed definition with `python testbed/generate_testbed_definition.py`

While the testbed definition supports setting multiple connections for nodes, the broker does not initiate any connections (settings are ignored) and
the clients only connect to the first connection (i.e. only one connection is supported by this setup). The clients also have to connect to a
server instance, since they do not work with incoming connections.

### MockFog Topology
This role:
- bootstraps (or destroys) an AWS infrastructure based on the generated testbed definition
- adds the **testbed_config** field to hostvars; useable via other roles and templates, e.g., `hostvars[inventory_hostname].testbed_config.bandwidth_out` (actually done on the fly via `ec2.py`)
- sets the name, internal_ip, and role tag of EC2 instances
Use with:
```fish
ansible-playbook --key-file=mockfog.pem --ssh-common-args="-o StrictHostKeyChecking=no" mockfog_topology.yml --tags bootstrap
ansible-playbook --key-file=mockfog.pem --ssh-common-args="-o StrictHostKeyChecking=no" mockfog_topology.yml --tags destroy
```

### MockFog Network
This role:
- configures delays via TC
Use with:
```fish
ansible-playbook -i inventory/ec2.py --key-file=mockfog.pem --ssh-common-args="-o StrictHostKeyChecking=no" mockfog_network.yml
```

### MockFog Application
This role:
- deploys application on nodes and starts it
- collects logs
Use with:
```fish
ansible-playbook -i inventory/ec2.py --key-file=mockfog.pem --ssh-common-args="-o StrictHostKeyChecking=no" mockfog_application.yml --tags deploy
ansible-playbook -i inventory/ec2.py --key-file=mockfog.pem --ssh-common-args="-o StrictHostKeyChecking=no" mockfog_application.yml --tags collect
```
Logs get the name of their instance added to their name while collecting.

## Additional Information

### CSV Files for Clients
The clients need a set of csv files generated by the IoT Dataset Generator to work. These files need to be zipped using the following structure:
`{ name of client node }.zip/multifile/[csv files]`, where the zip has the name of its corresponding client instance (which is set in the testbed
definition). These zip files then need to be put into the `MockFogLight/hiking` folder, since MockFog_Application will search for them there.

### Folder Structure on AWS Instances
Several files are copied to the AWS instances and then mounted into docker. What gets copied where is documented here.

#### Geobroker Server

First, the configuration files in `MockFogLight/mockfog_application_geobroker_server/templates` are copied to the broker instances, into
the folder `/geobroker_server/files`. The top folder can be changed using `MockFogLight/mockfog_application_geobroker_server/defaults/main.yml`.
This `files` folder is then mounted as `/files` into docker. Additionally, the `/geobroker_server/logs` folder is mounted as `/logs_sync` in
docker to collect the log files.

#### Geobroker Client

The mounting here is similar to the server. The same folders exist, except that `MockFogLight/mockfog_application_geobroker_client/defaults/main.yml`
sets the top folder as `/geobroker_client` on client instances. Additionally to what is mounted on server instances, the csv files for the client
instances are copied to `/geobroker_client/files/multifile`. Since the `/geobroker_client/files` folder is mounted as `/files` in docker, the
multifile folder is also available to the clients.
