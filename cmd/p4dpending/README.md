# p4dpending - P4D Commands without completion records

Based on the `go-libp4dlog` library, this tool will parse logs and produce a list of "pending" p4d commands in the log -
meaning commands with no completion record identified. So such commands were started processing, but did not complete by the end of the log.

Check the [releases](https://github.com/rcowham/go-libp4dlog/releases) page for the latest binary releases.

__*Contents:*__

- [p4dpending - P4D Commands without completion records](#p4dpending---p4d-commands-without-completion-records)
  - [Usage scenarios](#usage-scenarios)
    - [Analysing commands causing hang due to locking](#analysing-commands-causing-hang-due-to-locking)
  - [Running p4dpending](#running-p4dpending)
  - [Examples](#examples)
  - [Analysis using `jq`](#analysis-using-jq)
- [Building the p4dpending binary](#building-the-p4dpending-binary)

See [Project README](../../README.md) for instructions as to creating P4LOG files.

## Usage scenarios

This can be due to incomplete logs, or can be due to failures in the matching of start/complete records by the library. It is most often used
for validating the parsing of log files, and together with debug flags, can output lines which it failed to parse (useful for debugging the main library).

### Analysing commands causing hang due to locking

It can also be useful in scenarios such as a major hang due to process locks. On Linux, such situations are usually detected using the utility `lslocks` (and see https://github.com/perforce/p4prometheus/blob/master/scripts/monitor_metrics.py).

On Windows, because of the threading model used, it is not possible to identify which thread is holding which locks. Therefore using this utility can help identify the oldest uncompleted command in the p4d log which had not completed by the end of the log. This can at least indicate potential candidates to look at.

## Running p4dpending

The released binaries for this project are available for Linux/Mac/Windows. After downloading you may want to rename to just `p4dpending` (or `p4dpending.exe` on Windows)

It is a single executable `p4dpending` which will parse a text p4d text log file and generate a JSON output file
which identifies all "pending" commands - e.g. where no completion record has been found.

```
./p4dpending -h
usage: p4dpending [<flags>] [<logfile>...]

Parses one or more p4d text log files (which may be gzipped) and lists pending commands. Commands are produced in reverse chronological order.

Flags:
  -h, --help                     Show context-sensitive help (also try --help-long and --help-man).
      --debug=DEBUG              Enable debugging level.
      --json.output=JSON.OUTPUT  Name of file to which to write JSON if that flag is set. Defaults to <logfile-prefix>.json
      --debug.pid=DEBUG.PID      Set for debug output for specified PID - requires debug.cmd to be also specified.
      --debug.cmd=""             Set for debug output for specified command - requires debug.pid to be also specified.
      --version                  Show application version.

Args:
  [<logfile>]  Log files to process.
```

## Examples

    p4dpending log2020-02-01.log

will produce a `log2020-02-01.json` (stripping off `.gz` and `.log` from name and appending `.json`).

Also possible to parse multiple log files in one go:

    p4dpending --json.output logs.json log2020-02-*

Example output in .json file:

```
{"processKey":"45fe9561f979b1d60cddc23d399b6528","cmd":"user-sync","pid":22812,"lineNo":154918,"user":"jenkins",...
```

You can grep for the specified `pid` in the original log file, and compare and contrast the line no specified.

```
$ grep -n 'pid 22812' p4d.log
154919:	2022/02/22 02:01:38 pid 22812 jenkins@jenkins-swarm-jenkins-dev 127.0.0.1/10.5.41.173 [jenkins.p4-plugin/1.12.1/Linux (brokered)] 'user-sync -p -q /swarmJenkins/workspace/docker-development/...@2250393'
```
Note that the `lineNo` is the number of the preceding line ("Perforce server info:")

## Analysis using `jq`

Looking at the json file is easier with a bit of help from `jq` (https://jqlang.org/), e.g.

    jq -s 'sort_by(.lineNo  | tonumber)[] | {pid, cmd, app, lineNo, startTime, endTime}' p4d.json | less

or, filtering out syncs (which are normally expected to be long running):

    jq -s 'map(select(.cmd != "user-sync")) |sort_by(.lineNo  | tonumber)[] | {pid, cmd, app, lineNo, user, workspace, ip, startTime, endTime}' p4d.json | less

# Building the p4dpending binary

See the [Makefile](Makefile):

    make
or

    make dist

The latter will cross compile to create gzipped output files in the `bin` directory.
