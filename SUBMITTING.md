## Overview

YaST in terminal is organized in this script to run it and containers
that is used to run YaST.

### OBS Repos

- Container devel project is [YaST:Head:Containers](https://build.opensuse.org/project/show/YaST:Head:Containers)
- Container target project should be something under SUSE:ALP ( not yet created )
- scripts devel project is common [YaST:Head](https://build.opensuse.org/project/show/YaST:Head)
- scripts target project is common openSUSE:Factory from which it will be time to time synced to ALP.

For scripts there are [ci in place](https://ci.opensuse.org/job/yast-yast-in-container-master/).

### Notes

Containers base image can be what fits us, as ALP should be able to run older and also newer code streams.
So both SLE BCI or tumbleweed BCI can be used.
