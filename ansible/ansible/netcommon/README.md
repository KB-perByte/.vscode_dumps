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

Look for the path where your cloned/(editable)/installed ansible sits and open VS code at the same location.
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

JSYK, F5 will generally help you to debug the application and ctrl + F5 will execute the application in VS Code.
With this, Whether you are debugging ansible network content or just simple ansible content it should be VS Code debugable.
![Alt text](./images/normal_f5.png?raw=true "Debugging ansible")

Now, it would need a decent understanding of ansible-connection process and the ansible-playbook processes (plural), to take things forward.

We would need to open ansible.netcommon in another VS Code instance or add it to a workspace. As it would need a different .vscode configuration.
Once, we have ansible.netcommon in our IDE, we can ...

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

and go the `..collections/ansible_collections/ansible/netcommon/plugins/connection/network_cli.py`
Considering we are dealing with network_cli based connection. It can be anything netconf, grpc, httpapi.

We have to be smart about how we want to place each line of this block in network_cli code.

```
import debugpy
debugpy.listen(3000)
debugpy.wait_for_client()
```

As ansible-playbook process that is spawn by the ansible-connection process itself, deals with the json rpc requests ultimately does all the api calls that take the execution forward. We have to make sure `debugpy.listen(3000)` is executed only once, else right after the first json rpc request happens, we end up with a `Address already in use` error.

Now, how to split the debugpy chunk within the network_cli code to make things work.

1. The `import debugpy` statement goes anywhere around the top-level imports
2. `debugpy.listen(3000)` is the line that should run just once, and we need to make sure that during our whole execution, we should never invoke it more than once, if done we would end up with the error `address already in use` and the debugging stops abruptly. The workaround is we either introduce a decorator that flags if it is executed once and place the listen(3000) inside it. But in network_cli.py we already have a decorator `ensure_connect`. Where we can gracefully place it in.

```
def ensure_connect(func):
    @wraps(func)
    def wrapped(self, *args, **kwargs):
        if not self._connected:
            debugpy.listen(3000) <--------------------------------------------------------------- we hack it here.
            self._connect()
        self.update_cli_prompt_context()
        return func(self, *args, **kwargs)
    return wrapped
```

3. The last `debugpy.wait_for_client()` should go anywhere you want to start the debug journey in netcommon. In our case we use the send method to hold this line, as it helps us look at the interaction that happens by the json rpc server to it's client.

```
    @ensure_connect
    def send(
        self,
        command,
        prompt=None,
        answer=None,
        newline=True,
        sendonly=False,
        prompt_retry_check=False,
        check_all=False,
        strip_prompt=True,
    ):
        """
        Sends the command to the device in the opened shell
        """
        debugpy.wait_for_client() <---------------------------------------------------------------- we hacked it here.
```

NOW WE ARE READY.

Now if we want to debug netcommon you can either run ansible-playbook command via cli and just do F5 in the vscode instance that has the debugpy statements in ansible.netcommon
OR
We can start ansible via ctrl+F5 and do F5 with ansible.netcommon

### Ansible via Vs Code

![Alt text](./images/ctrl_f5.png?raw=true "Ansible runs via Vs Code")

### ansible.netcommon debugging via Vs Code

![Alt text](./images/netcommon_f5.png?raw=true "Debug netcommon runs via Vs Code")

Well, the last step

Your inventory file should contain the specified variable, it helps.

```
[all:vars]
ansible_network_import_modules=True
```

Refer to the VS code debugging guide for more information-
https://code.visualstudio.com/docs/editor/debugging

Special mentions @NilashishC @Qalthos @ganeshrn

## Happy Debugging !!!
