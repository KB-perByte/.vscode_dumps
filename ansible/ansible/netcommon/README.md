# Debugging Ansible connection process with VS Code

Debugging Ansible connection and playbook process in the most interactive way.

#### Getting started:

So if you are here, you are way ahead of the idea of debugging just modules, you might want to debug ansible's execution end to end.
This guide is not a deep dive into the technicalities of how Ansible works, rather it just shows how you can debug Ansible -> ansible.netcommon -> any.collection
via VS Code.

Assuming you have an Ansible dev environment already set up:

ah!

If not,
make a virtual environment. A clean slate.
Got to your dev directory.

```
git clone https://github.com/ansible/ansible
cd ansible
```

you might not want to do `source hacking/env-setup`
rather, we do pip install as editable.

`pip install -e .`

Now clone, ansible.netcommon and the any.collection network collection in proper places. (no help provided)

```
❯ tree -L 2
.
├── ansible
│   ├── molecule
│   ├── netcommon
│   ├── pylibssh
│   ├── scm
│   ├── utils
│   ├── yang
├── cisco
│   ├── ios
│   ├── iosxr
│   └── nxos
├── junipernetworks
│   └── junos
├── network
│   ├── acls
│   ├── base
├── splunk
│   └── es
├── trendmicro
│   └── deepsec
```

If your environment looks something like this ^^^^ you should be good.

#### Where place what!

Look for the path where your clone (editable) installed ansible sits and open VS code at the same location.
and add the below-mentioned launch.json under .vscode

```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Module",
            "type": "python",
            "request": "launch",
            "module": "ansible",
            "args": ["playbook", "debugPlaybook.yml", "-vvv"],
            "cwd": "/home/test/playbook",
            "justMyCode": false,
            "subProcess": true
        }
    ]
}

```

**\*** START HERE #TODO

#### Configuring VS code:

_(Use the same setting with a different port for debugging multiple modules)_
Once we are in vs code with the specific module we wish to debug, either use the debug option followed by the settings sign tagged ‘Open launch.json’ to generate the launch configuration file

![Alt text](./images/image_debugPanel.png?raw=true "Debugging option")

_Or_

Use the following commands to the root of the collection for generating the same

```
┌[machine@machine]-(~/.a/c/a/c/i)
└> mkdir .vscode
┌[machine@machine]-(~/.a/c/a/c/i/.vscode)
└> nano launch.json
```

Drop the below config in your launch.json

```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Attach",
            "type": "python",
            "request": "attach",
            "port": 3000 ,
            "host": "127.0.0.1",
            "justMyCode": false
        },
    ]
}
```

Get debugpy and install it within the scope of your environment.

`pip install debugpy`

To start debugging go to the intended module and put the following lines on top of it,

```
import debugpy
debugpy.listen(3000)
debugpy.wait_for_client()
```

Well, the last step

Your inventory file should contain the specified variable

```
[all:vars]
ansible_network_import_modules=True
```

Now right after running your playbook using ansible-playbook

```
[machine@machine]-(~/W/debug_example)
└> ansible-playbook debug_playbook.yml
```

Expect the execution to be stuck at the task execution level waiting for you to manually start the module and add it to port 3000 (or any configured port).

Navigating to the module where debugpy import was used press F5 to start debugging

![Alt text](./images/image_debugging.png?raw=true "Debugging demo")

At this point, the debugger should hit your break points when in process.

For debugging ansible-navigator check a cool doc [here]: https://github.com/shatakshiiii/debuggingNavigator/blob/main/README.md

Refer to the VS code debugging guide for more information-
https://code.visualstudio.com/docs/editor/debugging

#### Works with:

- older Network Modules
- newer Network Resource Modules
- execution with Ansible Navigator

## Happy Debugging !!!
