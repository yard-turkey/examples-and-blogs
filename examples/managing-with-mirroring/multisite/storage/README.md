# Storage setup

## AWS
In AWS an extra disk needs to be created and attached to the worker or OCS instances.

## AWS security groups
The following ports must be opened up or all traffic must be allowed between workers at site1 to site2 and vice versa.

The traffic must be opened up to the specific networks. For example site1 worker security group would need to be allowed from source 10.2.0.0/16 to the ports below. The same would be for site2 to 10.0.0.0/16	
* 6789
* 3300
* 6800-7300
