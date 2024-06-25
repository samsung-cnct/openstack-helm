## Rally installation
### The easiest way to install Rally is by running its installation script:
wget -q -O- https://raw.githubusercontent.com/openstack/rally/master/install_rally.sh | bash
or using curl:
curl https://raw.githubusercontent.com/openstack/rally/master/install_rally.sh | bash

If you execute the script as regular user, Rally will create a new virtual environment in ~/rally/ and install in it Rally, and will use sqlite as database backend. If you execute the script as root, Rally will be installed system wide. For more installation options, please refer to the installation page.
### Setting up the environment and running a task from samples
First, you have to provide Rally with an OpenStack deployment that should be tested. This should be done either through OpenRC files or through deployment configuration files. In case you already have an OpenRC, it is extremely simple to register a deployment with the deployment create command:
rally deployment create --file=existing.json --name=existing
Finally, the deployment check command enables you to verify that your current deployment is healthy and ready to be tested:
![image](https://github.com/samsung-cnct/openstack-helm/assets/8689383/97463394-3667-4b82-bc53-51098e09b9e4)
Now that we have a working and registered deployment, we can start testing it. The sequence of subtask to be launched by Rally should be specified in a task input file (either in JSON or in YAML format). Let’s try one of the task sample available in samples/tasks/scenarios, say, the one that boots and deletes multiple servers:

{
    "NovaServers.boot_and_delete_server": [
        {
            "args": {
                "flavor": {
                    "name": "m1.tiny"
                },
                "image": {
                    "name": "^cirros.*-disk$"
                },
                "force_delete": false
            },
            "runner": {
                "type": "constant",
                "times": 10,
                "concurrency": 2
            },
            "context": {
                "users": {
                    "tenants": 3,
                    "users_per_tenant": 2
                }
            }
        }
    ]
}



To start a task, run the task start command (you can also add the -v option to print more logging information):
rally task start samples/tasks/scenarios/nova/boot-and-delete.json

### Rally input task format
Rally comes with a really great collection of plugins and in most real-world cases you will use multiple plugins to test your OpenStack cloud. Rally makes it very easy to run different test cases defined in a single task. To do so, use the following syntax:

{
    "<ScenarioName1>": [<config>, <config2>, ...]
    "<ScenarioName2>": [<config>, ...]
}

{
    "args": { <scenario-specific arguments> },
    "runner": { <type of the runner and its specific parameters> },
    "context": { <contexts needed for this scenario> },
    "sla": { <different SLA configs> }
}
### Running Task against OpenStack with read only users

There are two very important reasons from the production world of why it is preferable to use some already existing users to test your OpenStack cloud:

1. Read-only Keystone Backends: creating temporary users for running scenarios in Rally is just impossible in case of r/o Keystone backends like LDAP and AD.

2. Safety: Rally can be run from an isolated group of users, and if something goes wrong, this won’t affect the rest of the cloud users.

The information about existing users in your OpenStack cloud should be passed to Rally at the deployment initialization step. The difference from the deployment configuration we’ve seen previously is that you should set up the “users” section with the credentials of already existing users. Let’s call this deployment configuration file existing_users.json:

{
     "openstack": {
         "auth_url": "http://example.net:5000/v2.0/",
         "region_name": "RegionOne",
         "endpoint_type": "public",
         "admin": {
             "username": "admin",
             "password": "pa55word",
             "tenant_name": "demo"
         },
         "users": [
             {
                 "username": "b1",
                 "password": "1234",
                 "tenant_name": "testing"
             },
             {
                 "username": "b2",
                 "password": "1234",
                 "tenant_name": "testing"
             }
         ]
     }
}

rally deployment create --file existing_users --name our_cloud

After you have registered a deployment with existing users, don’t forget to remove the “users” context from your task input file if you want to use existing users, like in the following configuration file (boot-and-delete.json):

{
    "NovaServers.boot_and_delete_server": [
        {
            "args": {
                "flavor": {
                    "name": "m1.tiny"
                },
                "image": {
                    "name": "^cirros.*-disk$"
                },
                "force_delete": false
            },
            "runner": {
                "type": "constant",
                "times": 10,
                "concurrency": 2
            },
            "context": {}
        }
    ]
}

### Rally task templates
A nice feature of the input task format used in Rally is that it supports the template syntax based on Jinja2. This turns out to be extremely useful when, say, you have a fixed structure of your task but you want to parameterize this task in some way. For example, imagine your input task file (task.yaml) runs a set of Nova scenarios:

---
  NovaServers.boot_and_delete_server:
    -
      args:
        flavor:
            name: "m1.tiny"
        image:
            name: "^cirros.*-disk$"
      runner:
        type: "constant"
        times: 2
        concurrency: 1
      context:
        users:
          tenants: 1
          users_per_tenant: 1

  NovaServers.resize_server:
    -
      args:
        flavor:
            name: "m1.tiny"
        image:
            name: "^cirros.*-disk$"
        to_flavor:
            name: "m1.small"
      runner:
        type: "constant"
        times: 3
        concurrency: 1
      context:
        users:
          tenants: 1
          users_per_tenant: 1


## Create rally cron job on ubuntu
Put a shell script in one of these folders: /etc/cron.daily, /etc/cron.hourly, /etc/cron.monthly or /etc/cron.weekly.

If these are not enough for you, you can add more specific tasks e.g. twice a month or every 5 minutes. Go to the terminal and type:

crontab -e

add */15 * * * * rally task start /path/to/rallytask

here are some rally task to run:

{
    "GlanceImages.create_and_download_image": [
        {
            "args": {
                "image_location": "http://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img",
                "container_format": "bare",
                "disk_format": "qcow2"
            },
            "runner": {
                "type": "constant",
                "times": 10,
                "concurrency": 2
            },
            "context": {
                "users": {
                    "tenants": 2,
                    "users_per_tenant": 3
                }
            },
            "sla": {
                "failure_rate": {
                    "max": 0
                }
            }
        }
    ]
}

{% set flavor_name = flavor_name or "m1.tiny" %}
{
    "NovaServers.boot_and_delete_server": [
        {
            "args": {
                "flavor": {
                    "name": "{{flavor_name}}"
                },
                "image": {
                    "name": "^cirros.*-disk$"
                },
                "force_delete": false
            },
            "runner": {
                "type": "constant",
                "times": 10,
                "concurrency": 2
            },
            "context": {
                "users": {
                    "tenants": 3,
                    "users_per_tenant": 2
                }
            },
            "sla": {
                "failure_rate": {
                    "max": 0
                }
            }
        },
        {
            "args": {
                "flavor": {
                    "name": "{{flavor_name}}"
                },
                "image": {
                    "name": "^cirros.*-disk$"
                },
                "auto_assign_nic": true
            },
            "runner": {
                "type": "constant",
                "times": 10,
                "concurrency": 2
            },
            "context": {
                "users": {
                    "tenants": 3,
                    "users_per_tenant": 2
                },
                "network": {
                    "start_cidr": "10.2.0.0/24",
                    "networks_per_tenant": 2
                }
            },
            "sla": {
                "failure_rate": {
                    "max": 0
                }
            }
        }
    ]
}




