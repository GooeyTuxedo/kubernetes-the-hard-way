# Prerequisites

## Digital Ocean Cloud Platform

This tutorial leverages the [Digital Cloud Platform](https://cloud.digitalocean.com/) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. [Sign up](https://try.digitalocean.com/freetrialoffer/) for $200 in free credits.


## Digital Ocean CLI

### Install the Digital Ocean CLI

Follow the Digital Ocean CLI [documentation](https://docs.digitalocean.com/reference/doctl/how-to/install/) to install and configure the `doctl` command line utility.

Verify the Digital Ocean CLI version is 1.93.1 or higher:

```
doctl version
```

### Contact customer service to raise account droplet limits

> The compute resources required for this tutorial exceed the Digital Ocean free trial account. Ya gonna need 6


### Set a Default Compute Region

This tutorial assumes a default compute region has been configured.

Edit your `~/.config/doctl/config.yaml` as follows
```
droplet:
    create:
        region: sfo3
vpcs:
    create:
        region: sfo3
```

> Use the `doctl compute regions list` command to view additional regions and zones.

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with synchronize-panes enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](images/tmux-screenshot.png)

> Enable synchronize-panes by pressing `ctrl+b` followed by `shift+:`. Next type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [Installing the Client Tools](02-client-tools.md)
