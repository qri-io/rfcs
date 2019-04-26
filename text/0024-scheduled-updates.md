- Feature Name: scheduled updates
- Start Date: 2019-04-12
- RFC PR: [#42](https://github.com/qri-io/rfcs/pull/42)
- Issue: <!-- (leave this empty) -->

# Summary
[summary]: #summary

overhaul `qri update`, making it a daemonized service that schedules & executes dataset updates. By scheduling updates instead of waiting for manual triggering, qri can keep data up to date on behalf of a user, and provide notifications of significant changes to datasets.

The goals of this RFC are twofold:
* define `qri update`, a service for automating dataset updates to replace the current qri update command
* define the characteristics & requirements of running a "fog service"

# Motivation
[motivation]: #motivation

As of this writing, all of my datasets are stale. They haven't been updated because, frankly, I'm too busy/distracted to remember to update them. This is a more serious issue than it seems at first. To put things bluntly: every dataset I create with Qri means more work for me, not less. This property has gotten in the way of me creating & maintaining lots of datasets. Since our goal is to convince others that there is merit in us working together on data, this makes getting to the "collaboration" part hard. 

To keep datasets fresh, qri needs to automate the update process.  By _automate_ I mean: execute dataset updates as a background process, registered with the local operating system, according to some schedule I can specify. My computer should do the work of keeping datasets up to date for me, and should show me a log of what's been done for me, calling attention to significant changes.

The magnitude of this shift to automation is noticeable. scheduled updates mean Qri is _doing work for me_. It takes the process of dataset updating out of the category of chores, and opens up the possibliity that Qri can surprise me with new insights. 

This features transitions qri into territory best described as a _fog service_: a decentralized network of machines automatically producing local resources for a user that can aggregate to a broader network. Over time I'm hoping we can do more and more things under this "fog service" paradigm.

Building a fog service requires special consideration to get right. This feature proposes running parts of Qri as a background process on a user's primary computing device. In this setting, we need to take great care to avoid interrupting a user's day-to-day computing experience, while also handling changing connectivity & resource availability. There is a very real chance that as a consequence of an update a user scheulded 6 months ago, Qri will be slated to start an intense script on a laptop with 5% battery left at a coffee shop while connected to restricted Wifi, while the user is nervously trying to get some dockerized python script to process a report that's due in an hour. We should aim to shape both the technical foundations and user experince of Qri to do the right thing in that context.

This is a very high bar, but I think it's the standard we should work toward. Qri cron is both a big first step and a chance to iterate on meeting these requirements before including more parts of the Qri daemon in this "fog service" paradigm.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Here's a typical workflow for qri cron using datasets. This example shows working with the command line. As usual all of this would also be accessible over the local API. This change proposes breaking the current `qri update` API, replacing the current update command with a number of subcommands nested under `qri update`. 

The previous `qri update` command is an alias for `qri save --recall tf`. The english phrasing: "re-run that transform script and create a new version if something changes". This this command is now moved to `qri update run`. `run` is now like the refresh button on a web browser, manually invoking an update outside of it's scheduled execution time. Users should only need `qri update run` when they want to override the new default behaviour of having scheduled updates execute automatically.

Moving on to the new stuff. The first is `qri update schedule`:

```
$ qri update schedule me/dataset_name
scheduled me/dataset_name
updates every 2 days
next update: Tues. April 11, 11:30:00

WARNING: qri update is not currently running. start with:
$ qri update start
to start the qri update service. until then datasets will not auto-update.
```

In this case we've run `qri update schedule` on a dataset that contains a valid `meta.accrualPeriodicity` field (in this case `R/P2D`). Qri used this value to infer a default update periodicity. If no `meta.accrualPeriodicity` were set, Qri would have returned an error:

```
error: no periodicity specified, please provide a periodicity argument. For example, use:
$ qri update schedule me/dataset_name R/P1W
To an update every week. 
More info at https://qri.io/docs/cron#periodicity
```

We can use this same technique to override the dataset-provided periodicity, in this case setting updates to every 8 hours, and use the `--publish` flag to have cron publish to the registry on successful update:
```
$ qri update schedule me/dataset_name R/P8H --publish
  scheduled: every 8 hours
  next: Tues. April 11, 11:30:00
  4 hours from now
```

In the above example we're providing two arguments to `qri update schedule`: a dataset reference, followed by an [ISO 8601 Repeating Interval string](https://en.wikipedia.org/wiki/ISO_8601#Repeating_intervals) that specifies unbounded updates every 8 hours. 

Running `qri update schedule` for the same dataset name `b5/dataset_name` will overwrite the previous job entry with new values. update jobs establish their next execution date by examining the more recent of two time sources:
1. last update execution
2. dataset commit timestamp

In this case an update has never run, so the scheduled date is derived from the commit timestamp (in this example the dataset was committed 4 hours ago). Any time either a new dataset version is created or the update is run, the next update execution time is re-evaluated & scheduled. This way if a user manually runs `qri update run me/dataset_name`, update will reset to use that time as an interval basis, avoiding excessive updates.

The flags for `qri update schedule` are be a subset of those available for `qri save`. Anything `qri save` can do, update should work the same way. For example:
```
$ qri save me/github_stars --config=org,qri-io,repo,qri-io --secrets=token,$GITHUB_TOKEN
$ qri update schedule me/github_stars --config=org,qri-io,repo,qri-io --secrets=token,$GITHUB_TOKEN
```

We can see scheduled updates with `qri update list`:
```
$ qri update list
1. b5/dataset_name
   scheduled: every 8 hours
   next: Tues. April 11, 11:30:00
   4 hours from now
```
The conventions for `list` should follow all other "list command conventions" for pagination, but listing order should be chronological according to next scheduled update soonest update first, fathest update last.

We can start the qri update service like this:
```
$ qri update start
cron update service started. stop it with:
$ qri update stop
```

If the current operating system doesn't yet support a daemonized service, the deamon should start in-process and show a warning message explaining to run this command with some equivalent of `nohup`. We should work to make as few people as possible see that message. As shown in the dialog, the update daemon must be stoppable with `qri update stop`.

With cron running Qri will run a lightweight background loop that checks for update events and executes them, logging the results to the event log. To see a log of cron events, run:
```
$ qri update log
* me/dataset_name
  succeeded 3h4m ago
* me/github_stats
  failed 2d16h27m ago
  message: error: network timeout
* ...
```

This will show a list of successful & failed update executions

### Scheduling shell scripts

Scheduling datasets directly is the ideal workflow in terms of portability, but a new set of use cases open by adding the capacity to schedule & execute shell scripts within the same cron environment. By having qri schedule shell scripts, we can present a unified user experience for scheduling dataset updates, while supporting use cases where data is first generated through a series of commands, then adding the results to Qri. Shell scripts should be supported as a first-class citizen in `qri update`:

```
$ qri update schedule path/to/update_script.sh R/PT4H/2019-06-01T00:00:00Z
```

In this case we're case scheduling a cron job to run every 4 hours from now until June 1st, 2019. When qri encounters a file ending in a shell script extension (`.sh` on 'nix platforms), it copies the file into a directory within the fs repo and names the job according the the filename. `qri update list` now shows both tasks, ordered by which script is the next to execute:

```
$ qri update list
1. update_script.sh
   scheduled: every 1 hour
   next: Tuesday April 11, 7:30:00
   1 hour from now
2. me/dataset_name
   scheduled: every 8 hours
   next: Tues. April 11, 11:30:00
   4 hours from now
```

This scheduled shell script has a zero-or-more relationship with datasets. This means it's possible to run a script that doesn't update a qri dataset at all, or have a single script update multiple datasets. To change datasets, scripts will need to call `qri save`, which will record updates as if they we manually invoked with the qri event log.

In documentation we will _strongly_ recommend against running multiple updates in the same shell script, as such a script is no longer conceptually transactional. Users would have to roll back a partially-successful script by hand. Thankfully there's a full audit trail for this sort of thing. We'll allow this behaviour because there are a number of advanced use-cases where being able to perform multiple updates in the same script is useful, and because it's much easier to consider the shell script opaque.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### OS Support
`qri update start` registers a deamonized service with the host Operating System. We should prioritize Platform support in the following order:
1. darwin
2. unix/linux
3. windows

### Determining Resource requirements, Monitoring & Constraining Resource Consumption
Systems like the [go playground](https://play.golang.org) use techniques like running each execution request in a separate process to ensure no one script dominates the available resources of the service. A similar dynamic may be appropriate here, running the actual update command in a forked subprocess that the parent process can monitor. More research is needed before we go off & build such extensive monitoring setups. 

* restrict updates to sequential execution in the `qri update` daemon process
* record execution duration with start & stop times
* compose an RFC for performing static analysis on starlark scripts for network requirements
* follow up with further research into resource monitoring

It's worth noting that [`systemd`](https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html) & [`launchd`](https://launchd.info/) both support resource constraint as part of service definition.


### Integration with the Qri Event log
For some time we've been writing to a number of qri "events" to a structured data log file. `qri update` should use the event log to record results so we can later surface cron events in UI, potentially as part of a greater "feed" of updates.

### Recording Stdout & Stderr

In both cases, it would be really nice to be able to see log files from stdout and stderr. One of my favourite parts of the [circleci UI](https://circleci.com/gh/qri-io/qri/1501) is the capacity to open up each "step" and see it's console output. I think our UI should be positioned to do the same for each cron job.

### Interaction with `qri save`
qri save will need to notify the update service when any new dataset version is created, so it has the opportunity to re-calculate job execution times if it's scheduled for updates.

We'll also need to take care to sync the options for `save`, and `qri update run` to cut down on confusion. Ideally they could use an intersection of common input structs at the lib level. 

# Drawbacks
[drawbacks]: #drawbacks

### Platform specific code
This brings us firmly into the territory of managing what platforms we support.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

### "qri hooks"
For shell script support, we could instead focus on adding "hooks", like git has. Then, users could choose to run a shell script only when `qri ` has found new data. The hooks approach excludes use cases that will create new versions as the result of running a script, not just a consequence of a dataset update completing.

That being said, we've also had requests for webhook support, which indicates a number of users would like the oppotunity to tail on the creation of new data, regardless of how an update was generated. I think both scheduled updates _and_ hooks are features we should build.

# Prior art
[prior-art]: #prior-art

### Keybase
Keybase has an excellent [service implementation written in go](https://github.com/keybase/client/tree/master/go/service), and is in my opinion a leading example of a fog service. It's worth studying the keybase codebase in detail.

### Go Playground
The [go playground](https://github.com/golang/playground) has some great examples of sandboxed code execution.

### CircleCI steps UI
Great inspiration for what a proper cron job log with terminal output could look like.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

### building towards `qrid`
Qri update is our first step in a roadmap that expands the _entire_ qri runtime to execute as a _daemonized service_ ("qrid") scheduled & managed by a local operating system. With this in mind, lessons we learn about constraining resources will contribute directly to that effort.

### better error handling, classification and reporting
Debugging _why a cron job didn't succeed_ is going to come entirely to error messages. `error: i/o timeout` isn't enough to tell a user "hey when this cron job ran you didn't have an internet connection, and your transform script makes HTTP requests". It should come as close to that as possible. I'm hoping we can dedicate dev cycles in the future to improving our error message story, ideally integrate with [upcoming changes to errors in go](https://go.googlesource.com/proposal/+/master/design/29934-error-values.md).

### Multi-Tenancy
A user may be running multiple qri repositories, each with their own cron jobs. We'll need to come up with a way to deal with this in both the short and long term.

### Cron log file persistence
writing the output of a shell script that may print who-knows-what to a log file buried in a hidden folder sounds like a great way to eat up hard drive space & irritate users. We should figure out a ux for "cleanup", and think about weather "cleaning" should be a process that operates across all Qri serivces at once.

### Windows Support
We haven't yet figured how we'll do this on windows. Do we require a cygwin / mingw / msys environment? Do we support powershell?