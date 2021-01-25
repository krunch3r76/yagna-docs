---
description: Introducing log files
---

# Debugging by use of log file

When developing applications on Golem, sooner or later you will get to a point where something is not working as expected. Executing commands on the provider-hosted container seems to be one of the most error-prone areas. If something is going wrong on the provider side, we need to have a tool that helps us in problem identification. When we know what are the error details it usually is much easier to correct our code and thus eliminate the bug.

## Introducing log files

The good news is that Golem high-level API supports log files. The log files contain all sorts of interesting stuff. Among others you will find there:

* Requestor &gt; Provider file transfer details
* Output streams of executing commands on the Provider: `stdout` and `stderr`
* Provider &gt; Requestor file transfer details

## How to enable logging?

The logging infrastructure in Golem's high-level API is quite universal and thus complicated, but for most of the cases using the `SummaryLogger` will do the job. 

{% hint style="info" %}
The `SummaryLogger`is an event listener that listens to all the atomic events and combines them into the more aggregated track of events that is easier to grasp by humans.
{% endhint %}

To enable logging to a file we need two things:

```python
enable_default_logger(log_file=args.log_file)
```

`enable_default_logger` enables default logger that: 

* outputs to the Requestor application's `stderr` all the log messages with the level `INFO`
* if the `log_file` is specified, outputs to the file specified all the log messages with the level `INFO` and `DEBUG`

Then in the `Executor`we need to:

```python
    async with Executor(
        ...
        event_consumer=log_summary(log_event_repr),
        ...
    ) as executor:
```

The `log_summary` creates `SummaryLogger` instance that is passed as event consumer to the Executor we are creating.

## Let's have an error

{% hint style="info" %}
We are going to simulate some bug in the code. We will take fully working code,  modify it a bit \(create the bug in the code that makes the application go bad at the provider side\).and then by analyzing the log file we will track what is the root cause of the problem. In the end, we will fix the code and check the details of its work by the analysis of the log file.
{% endhint %}



Let's look at the code of yacat that is part of the Golem high-level Python API repository. All the code that enables logging is already there. We just need to use `--log-file` command-line switch.

{% embed url="https://github.com/golemfactory/yapapi/blob/master/examples/yacat/yacat.py" %}

In line 24 we are generating the body of `keyspace.sh` script to be executed on the provider:

```python
def write_keyspace_check_script(mask):
    with open("keyspace.sh", "w") as f:
        f.write(f"hashcat --keyspace -a 3 {mask} -m 400 > /golem/work/keyspace.txt")
```

Now we are simulating a typo in the code. Instead of

`hashcat --keyspace -a 3 {mask} -m 400 > /golem/work/keyspace.txt`

...let's have the following code

`hashcat --keeyspace -a 3 {mask} -m 400 > /golem/work/keyspace.txt`

so just additional `e` that have not been noticed. A typical developer day.

Save the file and run the yacat with some example mask and hash:

{% tabs %}
{% tab title="Unix  / MacOS" %}
```python
python3 yacat.py '?a?a?a' '$P$5ZDzPE45CLLhEx/72qt3NehVzwN2Ry/' --log-file log.txt
```
{% endtab %}

{% tab title="Windows" %}
```
python yacat.py ?a?a?a $P$5ZDzPE45CLLhEx/72qt3NehVzwN2Ry/ --log-file log.txt
```
{% endtab %}
{% endtabs %}

## Reading the log file

And now let's try to track the effect of our typo in the content of `log.txt` - the file that has been generated by yacat because of using `--log-file log.txt`.

First, let's find the first `ctx.run` command track. We can do it by searching the log.txt file \(from the top of the file, for example by hitting Ctrl-F in our favorite file viewer\) in search of 

```python
command={'run'
```

This way we can find log entry that looks like this:

```python
[2021-01-22 15:20:16,273 DEBUG yapapi.events] 
CommandStarted(agr_id='b39318a4c682010ea9be37201b8f428fad81b28cfe949ca11cca9912a465d34c', 
task_id='1', cmd_idx=3, command={'run': {'entry_point': '/bin/sh', 'args': 
['/golem/work/keyspace.sh'], 'capture': {'stdout': {'stream': {}}, 'stderr': {'stream': {}}}}})
```

now below that, there are the log entries we were looking for:

```python
[2021-01-22 15:20:16,383 DEBUG yapapi.events] 
CommandStdErr(agr_id='b39318a4c682010ea9be37201b8f428fad81b28cfe949ca11cca9912a465d34c', 
task_id='1', cmd_idx=3, 
output="hashcat: unrecognized option '--keeyspace'\n\x1b[31mInvalid argument specified.\x1b[0m\n\n")
[2021-01-22 15:20:16,385 DEBUG yapapi.events] 
CommandExecuted(agr_id='b39318a4c682010ea9be37201b8f428fad81b28cfe949ca11cca9912a465d34c', 
task_id='1', cmd_idx=3, command={'run': {'entry_point': '/bin/sh', 'args': 
('/golem/work/keyspace.sh',), 'capture': {'stdout': {'stream': {}}, 'stderr': {'stream': {}}}}}, 
success=False, message='ExeScript command exited with code 255')
```

There are two parts of those log entries that are most important:

`hashcat: unrecognized option '--keeyspace'\n\x1b[31mInvalid argument specified.\x1b[0m\n\n`

This is `stderr` message generated by running `hashcat` binary. It seems that the `hashcat` does not recognize the `--keeyspace` parameter. This is the error we simulated. 

{% hint style="info" %}
Congratulations! You just tracked the bug by analyzing the requestor log file.
{% endhint %}

The other interesting stuff here is

`ExeScript command exited with code 255`

By the above, we know that the effect of this `ctx.run` resulted with exit code having value of `255`. The value of `255` has been returned by `hashcat` on exit. If there is any additional information in the command exit code, we can track it here.

{% hint style="danger" %}
The `255` value, we have in the log file here is generated by equivalent of POSIX `exit(-1)` executed on the `hashcat` termination. The `-1` was converted to `unsigned int` and become`255`.
{% endhint %}

## Fixing the error

Now fix the `write_keyspace_check_script` to the correct form that is:

```python
def write_keyspace_check_script(mask):
    with open("keyspace.sh", "w") as f:
        f.write(f"hashcat --keyspace -a 3 {mask} -m 400 > /golem/work/keyspace.txt")
```

After saving the corrected `yacat.py` file and executing with the same command as above:

{% tabs %}
{% tab title="Unix  / MacOS" %}
```python
python3 yacat.py '?a?a?a' '$P$5ZDzPE45CLLhEx/72qt3NehVzwN2Ry/' --log-file log.txt
```
{% endtab %}

{% tab title="Windows" %}
```
python yacat.py ?a?a?a $P$5ZDzPE45CLLhEx/72qt3NehVzwN2Ry/ --log-file log.txt
```
{% endtab %}
{% endtabs %}

the problematic log fragment we analyzed above, becomes the following:

```python
[2021-01-22 16:22:17,217 DEBUG yapapi.events] 
CommandStarted(agr_id='bf8c484e362af417cbf5fc440281debd5c5cf772b433f53e0ecb90b4faac55f4', 
task_id='1', cmd_idx=3, command={'run': {'entry_point': '/bin/sh', 'args': 
['/golem/work/keyspace.sh'], 'capture': {'stdout': {'stream': {}}, 'stderr': {'stream': {}}}}})
[2021-01-22 16:22:18,303 DEBUG yapapi.events] 
CommandExecuted(agr_id='bf8c484e362af417cbf5fc440281debd5c5cf772b433f53e0ecb90b4faac55f4', 
task_id='1', cmd_idx=3, command={'run': {'entry_point': '/bin/sh', 'args': ('/golem/work/keyspace.sh',), 
'capture': {'stdout': {'stream': {}}, 'stderr': {'stream': {}}}}}, success=True, message=None)
```

As you see we have `success=True` with means that the `ctx.run` execution was successful.

{% hint style="info" %}
Congratulations! Error in the code has been corrected.
{% endhint %}

## Log files - what else is there?

In the log file, there is all the information you are already familiar with, because it is displayed as `INFO` messages at the `stdout` of the requestor applications. Between the familiar `INFO` entries, you will find additional `DEBUG` messages. Let's analyse most interesting of them.

### Requestor &gt; Provider file transfer details

The transferring files from Requestor to the Provider can be identified in the log as pair of `CommandStarted` and `CommandExecuted` similar to:

```python
[2021-01-22 16:22:16,773 DEBUG yapapi.events] 
CommandStarted(agr_id='bf8c484e362af417cbf5fc440281debd5c5cf772b433f53e0ecb90b4faac55f4', 
task_id='1', cmd_idx=2, command={'transfer': {'from': 
'gftp://0xf378a153b24fe98227ee153a0cb056faed155ba7/050a02b6976e59fbc1b9dfa6292d4f63942a820ee7dcc859af0a5efaea80c959', 
'to': 'container:/golem/work/keyspace.sh', 'format': None, 'depth': None, 'fileset': None}})
[2021-01-22 16:22:17,217 DEBUG yapapi.events] 
CommandExecuted(agr_id='bf8c484e362af417cbf5fc440281debd5c5cf772b433f53e0ecb90b4faac55f4', 
task_id='1', cmd_idx=2, command={'transfer': {'from': 
'gftp://0xf378a153b24fe98227ee153a0cb056faed155ba7/050a02b6976e59fbc1b9dfa6292d4f63942a820ee7dcc859af0a5efaea80c959', 
'to': 'container:/golem/work/keyspace.sh'}}, success=True, message=None)
```

The source file name is identified by `'from':'gftp://0xf378a153b24fe98227ee153a0cb056faed155ba7/050a02b6976e59fbc1b9dfa6292d4f63942a820ee7dcc859af0a5efaea80c959'`

{% hint style="info" %}
The `gftp` is internal Golem protocol for transferring files in the Golem network.
{% endhint %}

The target file name is identified by 

`'to': 'container:/golem/work/keyspace.sh'`

The `success=True` informs us that the operation was successful.

### stdout of executing commands on the Provider

The stdout of `ctx.run` will effect in `CommandStarted`, multiple `CommandStdOut` and one `CommandExecuted` entries. The yacat main `ctx.run` results in the log looks like this:

```python
[2021-01-22 16:22:36,986 DEBUG yapapi.events]
CommandStarted(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, command={'run': {'entry_point': '/bin/sh', 'args': 
['-c', 'rm -f /golem/work/*.potfile ~/.hashcat/hashcat.potfile; touch /golem/work/hashcat_0.potfile; 
hashcat -a 3 -m 400 /golem/work/in.hash ?a?a?a --skip=0 --limit=3009 --self-test-disable -o 
/golem/work/hashcat_0.potfile || true'], 'capture': {'stdout': {'stream': {}}, 'stderr': {'stream': {}}}}})
[2021-01-22 16:22:37,018 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='hashcat (v5.1.0) starting...\n\n')
[2021-01-22 16:22:37,018 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4',
task_id='2', cmd_idx=3, output='\x1b[2K\rParsed Hashes: 1/1 (100.00%)\x1b[2K\rParsed Hashes: 1/1 (100.00%)')
[2021-01-22 16:22:37,018 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='\x1b[2K\rCounted lines in /golem/work/in.hash...')
[2021-01-22 16:22:37,019 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='Counting lines in /golem/work/in.hash...')
[2021-01-22 16:22:37,019 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='OpenCL Platform #1: Intel(R) Corporation\n========================================\n* 
Device #1: Intel(R) Core(TM) i7-4790K CPU @ 4.00GHz, 5347/21388 MB allocatable, 4MCU\n\n')
[2021-01-22 16:22:37,048 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='\x1b[2K\rSorting hashes...\x1b[2K\rSorted hashes...\x1b[2K\r
Removing duplicate hashes...\x1b[2K\rRemoved duplicate hashes...\x1b[2K\rSorting salts...\x1b[2K\rSorted salts...\x1b[2K\r
Comparing hashes with potfile entries...\x1b[2K\rCompared hashes with potfile entries...')
[2021-01-22 16:22:37,048 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='\x1b[2K\rGenerated bitmap tables...')
[2021-01-22 16:22:37,048 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='\x1b[2K\rGenerating bitmap tables...')
[2021-01-22 16:22:37,963 DEBUG yapapi.events] 
CommandStdOut(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, output='\x1b[2K\rHashes: 1 digests; 1 unique digests, 1 unique salts\n
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates\n\nApplicable optimizers:\n* 
Zero-Byte\n* Single-Hash\n* Single-Salt\n* Brute-Force\n\nMinimum password length supported by kernel: 0\n
Maximum password length supported by kernel: 256\n\n\x1b[33mATTENTION! Pure (unoptimized) 
OpenCL kernels selected.\x1b[0m\n\x1b[33mThis enables cracking passwords and salts > length 32 but for the 
price of drastically reduced performance.\x1b[0m\n\x1b[33mIf you want to switch to optimized OpenCL kernels, 
append -O to your commandline.\x1b[0m\n\x1b[33m\x1b[0m\nWatchdog: Hardware monitoring interface not found 
on your system.\nWatchdog: Temperature abort trigger disabled.\n\n
Initializing device kernels and memory...\x1b[2K\rInitializing OpenCL runtime for device #1...')

...

[2021-01-22 16:22:40,649 DEBUG yapapi.events] 
CommandExecuted(agr_id='ac6c0ea05bd668ee6afc5d2cf255e3f6ac40eabe78b4a313e18d521eaa07dfa4', 
task_id='2', cmd_idx=3, command={'run': {'entry_point': '/bin/sh', 'args': 
('-c', 'rm -f /golem/work/*.potfile ~/.hashcat/hashcat.potfile; touch /golem/work/hashcat_0.potfile; 
hashcat -a 3 -m 400 /golem/work/in.hash ?a?a?a --skip=0 --limit=3009 --self-test-disable -o 
/golem/work/hashcat_0.potfile || true'), 'capture': {'stdout': {'stream': {}}, 'stderr': {'stream': {}}}}}, success=True, message=None)
```

The `output` parts of the `CommandStdOut` contains the `stdout` of the `ctx.run`. For example the beginning of the `hashcat` execution can be traced by `output='hashcat (v5.1.0) starting...\n\n'` .

### Provider &gt; Requestor file transfer details

Getting files back from the provider to the requestor will leave a trace such as:

```python
[2021-01-22 16:22:18,304 DEBUG yapapi.events] 
CommandStarted(agr_id='bf8c484e362af417cbf5fc440281debd5c5cf772b433f53e0ecb90b4faac55f4', 
task_id='1', cmd_idx=4, command={'transfer': {'from': 'container:/golem/work/keyspace.txt', 'to': 
'gftp://0xf378a153b24fe98227ee153a0cb056faed155ba7/0GdLl2vUcEYgO7ExgkVuttc5oqeAmU90Im911Eqs5LZXzD9ehlYCUGH7p5oII9jSg', 
'format': None, 'depth': None, 'fileset': None}})
[2021-01-22 16:22:18,513 DEBUG yapapi.events] 
CommandExecuted(agr_id='bf8c484e362af417cbf5fc440281debd5c5cf772b433f53e0ecb90b4faac55f4', 
task_id='1', cmd_idx=4, command={'transfer': {'from': 'container:/golem/work/keyspace.txt', 'to': 
'gftp://0xf378a153b24fe98227ee153a0cb056faed155ba7/0GdLl2vUcEYgO7ExgkVuttc5oqeAmU90Im911Eqs5LZXzD9ehlYCUGH7p5oII9jSg'}}, 
success=True, message=None)
[2021-01-22 16:22:18,517 DEBUG yapapi.events] 
GettingResults(agr_id='bf8c484e362af417cbf5fc440281debd5c5cf772b433f53e0ecb90b4faac55f4', task_id='1')
[2021-01-22 16:22:18,519 DEBUG yapapi.events] DownloadStarted(path='/golem/work/keyspace.txt')
[2021-01-22 16:22:18,521 DEBUG yapapi.events] DownloadFinished(path='keyspace.txt')
```

The source filename is identified by `'from': 'container:/golem/work/keyspace.txt'` and the target filename is identified by `DownloadFinished(path='keyspace.txt')`.

## Docker exec - an alternative approach 

Analyzing the requestor logs gives us the full picture of what is going on under the hood of Golem, but in some cases, it might be easier to start with executing a simple `docker exec`that executes commands on the container. The syntax of this docker command is:

```bash
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

This command can be used to execute the `ctx.run` commands directly on the provider container. This is standard stuff so we will leave you with a reference to docker documentation [https://docs.docker.com/engine/reference/commandline/exec/](https://docs.docker.com/engine/reference/commandline/exec/)

## Closing words

Besides requestor logs there are also provider side logs. The suggested approach for the developer to know the provider side logs is to use our testing environment:

{% page-ref page="interactive-testing-environment/" %}


