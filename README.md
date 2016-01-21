# ***The ultimate golden release upgrade guide***
Community effort to write an upgrade guide for dCache 2.13

HTML preview:
* https://htmlpreview.github.io/?https://github.com/dCache/upgrade-guide-213/blob/master/HTML/upgrade-guide-213.html

# ***How to get from dCache 2.10 to dCache 2.13***

**Authors**  
* Gerd Behrmann \<behrmann@ndgf.org\>  
* Paul Millar \<paul.millar@desy.de\>  
* Marc Caubet \<mcaubet@pic.es\>

## About this Document

The dCache golden release upgrade guides aim to make migrating between
golden releases easier by compiling all important information into a
single document. The document is based on the release notes of versions
2.10, 2.11, 2.12 and 2.13. In contrast to the release notes, the guide
ignores minor changes and is structured more along the lines of
functionality groups rather than individual services.

Although the guide aims to collect all information required to upgrade
from dCache 2.10 to 2.13, there is information in the release notes not
found in this guide. The reader is advised to at least have a brief look
at the release notes found at [dCache.org](http://www.dcache.org/).

There are plenty of changes to configuration properties, default values,
log output, admin shell commands, admin shell output, etc. If you have
scripts that generate dCache configuration, issue shell commands, or
parse the output of shell commands or log files, then you should expect
that these scripts need to be adjusted. This document does not list all
those changes.

## About this Release

dCache 2.13 is the sixth golden (long term support) release of dCache.
The release will be supported at least until summer 2017. This is the
recommended release for the second LHC run.

As always, the amount of changes between this and the previous golden
release is staggering. We have tried to make the upgrade process as
smooth as possible, while at the same time moving the product forward.
This means that we try to make dCache backwards compatible with the
previous golden release, but not at the expense of changes that we
believe improve dCache in the long run.

The most visible change when upgrading is that most configuration
properties have been renamed, and some have changed default values. Many
services have been renamed, split, or merged. These changes require work
when upgrading, even though most of them are cosmetic - they improve the
consistency of dCache and thus ease the configuration, but those changes
do not fundamentally affect how dCache works.

dCache pools from version 2.10 can be mixed with services from dCache 2.13. This enables a staged roll-out in which non-pool nodes are upgraded     to 2.13 first, followed by pools later. It is however recommended to     upgrade to the latest version of 2.10 before performing a staged     roll-out.

Upgrading directly from versions prior to 2.10 is not officially supported. If you run versions before 2.10, we recommend upgrading to 2.10 first.     That is not to say that upgrading directly to 2.13``     ``is     not possible, but we have not explicitly tested nor documented such     an upgrade path.

Downgrading after upgrade is not possible without manual intervention:
In particular database schemas have to be rolled back *before*
downgrade. If pools use the Berkeley DB meta data back-end, these need
to be converted to the file back-end *before* downgrading and then back
to the Berkeley DB back-end after downgrade.

## Executive summary
### Highlights from dCache release 2.13:
#### Highlights introduced on dCache 2.11:

* Faster deletion speeds
* Better integration of alarms service, easier configuration, and predefined alarms
* Multiple GIDs in kpwd
* Less clutter in GLUE 2 StorageShare
* Improved pool response during HSM request bursts
* Reduced latency on write
* Automatic throttling of SRM request rate when database request queue is full
* Decoupling of transfer managers from srm database

#### Highlights introduced on dCache 2.12:
* New site description property to add local branding.
* Admin shell supports Ctrl-C to interrupt commands.
* Alarms service can share domains with other services.
* Improved attribute handling in Chimera.
* Shorter session identifiers.
* FTP and DCAP cell names are unique across restarts.
* Info provider publishes all global network interfaces.
* Improved login broker registration and SRM door selection, including improved IPv6 support.
* Improved and more robust NFS proxy mode.
* On the fly checksum calculation for NFS and xrootd.
* Improved nearline storage Service Provider Interface for more flexible drivers.
* Precomputed repository statistics in pools.
* HTTP and xrootd uploads block until post-processing completes.
* Reduced memory consumption in pools using Berkeley DB.
* Site specific srmPing attributes.
* Parameterized WebDAV templates.
* More uniform access logs.
* Localized timestamps in pinboard.
* Improved bash completion.

#### Highlights introduced on dCache 2.13:
* major overhaul of admin interface,
* robust internal delete and flush notification,
* info-provider generates timestamps on published objects,
* pool migration module maintains last-access-time,
* pool implements strict LRU garbage collection,
* improved shutdown in space-manager,
* reserving space from a linkgroup may be authorised by GID,
* better scalability in FTP doors,
* new admin commands for nfs service,
* recording of FTS-specific information,
* xrootd doors support the protocol's asynchronous open.
* support for custom HTTP response headers

### Incompatibilities
* All properties marked forbidden or obsolete are now no longer recognised by dCache. All properties that were deprecated in 2.10 are now marked forbidden.
* The poolmanager.conf file should be regenerated using the `save` command before upgrading.
* Instatiation of WAR files in the httpd service has changed. Custom `httpd.conf` configurations will have to be updated.
* Local customizations to the WebDAV directory listing template will likely have to be updated.
* Third party xrootd channel handler plugins from previous releases are not compatible with dCache 2.13, except for the [XRootD CMS-TFC Release](https://www.dcache.org/downloads/xrootd4j/index.shtml). Contact the vendor for updates plugins. Please check [Installation of Plugins](#Steps|outline) for further information.
* Local modifications to logback.xml should be reapplied to the upstream version.
* The authorization rules for space reservations without an owner has changed.
* The webadmin interface no longer provides access to the info service. The same functionality is available through other httpd plugins.
* Java 8 is required for running dCache 2.13. Both the Oracle JDK and OpenJDK are supported.
* The chimera schema is updated upon upgrade.
* The `admin` shell has been reimplemented and the user interface has changed.
* The `broadcast` service has been dropped and needs to be removed from the layout file.
* The `dir` service has been dropped and needs to be removed from the layout file.
* The `loginbroker` service has been dropped and needs to be removed from the layout file.
* Support for JMS brokers (ActiveMQ and OpenMQ) has been removed.
* Support for Terracotta load balancing has been removed.
* The `httpd` service reads certificates from `/etc/grid-security` and no longer uses Java Key Store files.
* The `dcache     import` commands have been removed.
* `pnfsmanager` and `cleaner` services need to be configured according to the presence of a `replica` or `spacemanager` service.
* dCache now defaults to expect a `spacemanager` service. If not present, several services need to be configured not to expect a `spacemanager`.
* The default cell name of the `spacemanager` service was changed from `SrmSpaceManager` to `SpaceManager`.
* The default database names and owners for all services have changed.
* The pool Berkeley DB meta data format has changed. No change is required upon upgrading, but downgrading is not possible without explicitly converting the format.

## Services
This section describes renamed, collapsed, split, and removed services.
A summary of the changes is contained in the following table.

|Service|Configuration / Remark|
|:------|:---------------------|
|*admin*|New admin interface. Add '-s legacy' to use old method.|
|*broadcast*|Service removed|
|*loginbroker*|Service removed|
|*dir*|Service removed|
|*httpd*|JGlobus for TLS. Support for multiple httpd instances.|
|*pnfsmanager*|  |
|*spacemanager*|`Default                         dcache.enable.space-reservation=true `|
|*srmspacemanager*|It has been renamed to “spacemanager”|

## Removed Services
----------------

### BroadCast Service
#### Broadcast service replaced by publish-subscribe messaging

The `broadcast` service is no longer supported and must be removed from
the layout file on upgrade.

The functionality of the broadcast service have been replaced by an
integrated multicast ability in the cell messaging system. Sending cells
*publish* messages on a common *topic* while receiving cells *subscribe*
to these topics. The topic is a regular cell address and is analogous to
a multicast group in networking. This cell address is not bound to any
particular cell. Instead the new TOPIC route directs messages published
to a topic to all subscribers.

The system is transparent and does not require any configuration.

### LoginBroker service
#### LoginBroker service replaced by publish-subscribe messaging

The `loginbroker` service is no longer supported. This service acted as
a directory service for doors and was used by `srm`, `info`, and `httpd`
to find doors. The functionality has been replaced by publish-subscribe
messaging.

To control which doors are used by `srm`, `httpd` and the info provider,
a new tag system has been introduced. Doors may be tagged by setting the
`*.loginbroker.tags` property, and the consuming services may in turn
filter doors by those tags. E.g. will only use doors that has one of the
tags defined by `srm.protocols.tags`.

By default, the `srm` service only uses doors tagged as `srm` and the
info-provider only publishes doors tagged as `glue`. By default, all
doors are tagged as both `srm` and `glue`.

As part of this change, both the publishing and consuming services have
been extended with several new `lb` commands. Doors can dynamically
change the tags they publish, as well as stop publishing read or write
capabilities (thus preventing SRM from using those doors). Similarly,
consuming services like `srm` have been extended with commands to list
collected information about doors:

    lb disable [-read] [-write]  # suspend publishing capabilities
    lb enable [-read] [-write]  # resume publishing capabilities
    lb ls [-l] [-protocol=<string>]<...>  # list collected login broker information
    lb set tags <tags><...>  # set published tags
    lb set threshold <load>  # set threshold load for OOB updates
    lb set update <seconds>  # set login broker update frequency
    lb update # refresh login broker information

### Dir service
#### dir service removed → DCAP directory service is auto instantiated

The `dir` service used by DCAP to provide directory listing has been
removed. The corresponding cell is now automatically instantiated
whenever a DCAP door is created. Upon upgrade, the `dir` service must be
removed from the layout file.

### Other removed services
#### Support for JMS removed

Several years ago, an alternative messaging system based on the Java
Messaging Services specification was introduced in dCache. This allowed
external message brokers like ActiveMQ and OpenMQ to be used with
dCache. This feature never caught on and we have removed all support for
JMS.

#### Support for Terracotta removed

The `srm` service has supported a load-balancing system called
Terracotta. This system supported running multiple SRM instances in a
load-balancing setup. To our knowledge, this system has never been
utilized by any site, and our internal evaluation showed that the
overhead was relatively large. To avoid wasting developer resources on
an unused feature, all support for Terracotta has been removed

#### 'dcache import' command removed

Since certificates no longer need to be imported into Java Keystore
files, the `dcache     import` commands have been removed from the
`dcache` script.

#### DataNucleus property file inaccessible

The DataNucleus property file for low level database configuration is no
longer exposed. The configuration properties
`alarms.db.datanucleus-properties.path`, `billing.db.config.path`, and
`httpd.alarms.db.config.path` are obsolete.

## What's new


### Changes affecting several services
#### Site description property

There are several places where dCache provides a short, human-readable
description of this dCache instance or where this would make sense: the
info-provider, the webdav directory listing and the webadmin interface.
This release introduces the `dcache.description` property that is used
as a default in these places.

#### Short session strings

FTP, WebDAV and NFS doors now use BASE–64 encoding for the session
counter. For such doors, sessions have the form
`door:<cell>@<domain>:<counter>`. The `<counter>` part used to be
base–10 encoded (i.e., a normal integer number). This part is now a
base–64 number. The result is shorter log lines and billing entries.

#### Improved shutdown

When a dCache domain shuts down, it asks each cell in turn to stop. As a
backup procedure, dCache will wait up to one second for all threads
associated with that cell to terminate before explicitly interrupting
any of that cell’s remaining threads. The `dcache     stop` command,
which triggers the domain shutdown, will wait up to ten seconds for the
domain to finish shutting down. After this, it kills the java process
outright. Therefore domain shutdown must finish before this deadline to
achieve a clean shutdown.

The shutdown procedure of several services (billing, httpd, loginbroker,
pool, poolmanager, replica) have been improved by explicitly stopping
background activity. Previous versions of dCache before 2.11 relied on
dCache’s backup procedure for terminating left-over activity. By
explicity stopping background activity, shutting down each of these
services can now take less than one second.

Shutting down the pnfsmanager service has also improved. It is now
faster, cleaner and safer. Previously, pnfsmanager also relied on the
backup procedure to terminate on-going activity; this was unfortunate
not only because it slowed down the shutdown but also that such activity
could attempt to use the database after the database connection pool was
stopped. This could result in errors being logged and risked partial
processing of requests. This was fixed on dCache 2.11.

When shutting down services that use a thread-pool for processing
messages (gplazma, pinmanager, pool, srm), the shutdown sequence now
ensures all threads have stopped before closing resources. As with
pnfsmanager, previous versions of dCache risked concurrent activity
continuing after database connections have been stopped. This is now
fixed.

#### Improved HTTP services

Jetty is the web-server framework used by the webdav, httpd and srm
services. In HTTP 1.1, a network connection is kept open unless the
client explicitly requests it be closed. To the user, subsequent
requests appear to be processed faster as there is no overhead from
establishing a new TCP connection and SSL/TLS/GSI context.

Before dCache version 2.11, each HTTP connection occupies a thread, even
when waiting for the client to send the next request. Such occupied
threads, cannot process requests from other clients. This made tuning
dCache for optimum performance difficult: many clients connected but not
issuing requests will result in high memory usage, but reducing this
number can impact on the server’s ability to handle concurrent requests.
Safe configurations will likely lead to dCache under performing under
high load. With v2.11 and above, dCache will free up the thread while
waiting for the client’s next request, making it easier to obtain
optimum performance.

A separate improvement is that dCache reacts better when faced with many
clients attempting to connect at (nearly) the same time.

In HTTP 1.1, if the client decides to keep the connection open, the
server may still close it later on. In dCache, the policy is to close
any connection that remains idle for too long. Idle means no traffic
flowing in either direction; either dCache is waiting for the next
client request or the client is waiting for dCache’s response.

By default, srm service will close connections after one minute of
inactivity; the webdav service will close idle connections after five
minutes of inactivity; and, for httpd service, connections are closed
after 30 seconds of inactivity. These idle timeouts may be adjusted
through dCache configuration properties.

Jetty also has the concept of being low on resources. This is triggered
when all request processing threads are active and there are more
clients waiting to connect; for example, when many clients attempting to
connect at (more or less) the same time.

When Jetty detects it is low on resources, it can react by reducing the
timeout for idle connections. By more aggressively closing idle
connections, the service can “degrade gracefully” when faced with a
sudden inrush of client connections; the users experience only a slight
delay in receiving replies as the necessary resources are quickly freed.
The srm and webdav services reduce idle connection to 20 seconds and 30
seconds respectively, whereas httpd does not react.

In previous versions of dCache, when low on resources, the more
aggressive idle-disconnection policy would affect only new connections;
the existing idle connections would enjoy longer timeouts. With dCache
v2.11+, all SRM or WebDAV connections experience the reduced
idle-connection threshold.

#### Improved deletion speed

The database updates needed to delete a file has been reduced, making
deletions faster. The effect is most pronounced if the file contained
many non-empty levels. This affects the pnfsmanager and nfs services.

#### Unified mover timeout handling in doors

Doors have several timeouts related to starting a mover on a pool,
however the precise semantics were not unified. With this version, all
doors use common timeout concepts for pool selection, mover creation and
mover queuing.

#### Unique FTP and DCAP cell names

DCAP and FTP doors instantiate a cell per client connection. The names
of these cells are formed by concanating the door’s name with a counter.
This counter used to reset on every restart. With this release, the
names are derived from a time stamp and with high property the names are
unique accross restarts. To reduce the length of the names, the counter
is base 64 encoded.

#### Shorter session identifiers

Most log messages include a session identifier and link all log messages
generated by a common user session. These session identifiers are
usually formed by combining a cell name with a counter. For FTP and DCAP
doors this lead to session identifiers with two counters, resulting in
needlessly long identifiers. With this release such redundancy is
avoided.

#### Default FTP internal network interface changed

FTP doors can act as proxies for the data channel between pools and
clients. When establishing such a proxy, the FTP door listens for a
connection from the pool. Before, the interface the door would listen on
on multi-homed hosts would default to the interface the FTP client
connected over. Since the client connection has no relationship to which
network pools are accessible from, this release changes the door such
that it listens to the interface to which the host name resolves. Like
before, the default can be overriden by setting the `ftp.net.internal`
property.

#### loginbroker and SRM gain IPv6 support

The loginbroker service acts as a registry for doors and their network
interfaces.

With this release, doors register both IPv4 and IPv6 interfaces with
their loginbrokers. The discovery of network interfaces is now dynamic
such that interfaces that disappear or appear after a door is started
will be removed from or added to the registration in the loginbrokers.

Doors now also accept a hostname or FQDN for their `*.net.listen`
property. When registering with a loginbroker, both the name and the
resolved IP address is published. The info-provider and the SRM will use
this name rather than doing a reverse lookup on the IP address as they
did before. If an IP address is used in the configuration, it is the
door than now performs a reverse lookup.

The SRM will prefer a TURL that retains the protocol family the SRM
client used. That is, for an IPv6 client an IPv6 TURL is preferred, and
for an IPv4 client an IPv4 TURL is preferred. The SRM will fall back to
the other protocol family if necessary. Note however that TURLs still
contain the host name as published by the doors.

#### Compress events logs

Event logs are a debugging feature that record low-level events in an
easy to parse format. Event logs are disabled by default. If enabled,
the logs are rotated daily and starting with this release the rotated
logs are automatically compressed.

#### Localized time stamps in pinboard log

Timestamps in the in-memory pinboard log used to follow a 12 hour
format. The format has now been localized, meaning it will follow the
conventions of the current local.

#### Renamed and improved wait-for-cells script

The wait-for-cells script uses the info service to determine whether the
supplied list of cells is currently running. If not, the script will
wait a defined period for the cells to appear before terminating with an
error. This release improces the script, strips the .sh suffix and moves
it to `/usr/sbin/`.

#### Improved bash completion script

The script implementing bash completion for the `dcache` command has
been extended and now supports all sub-commands.

#### Updated third party libraries

A large number of third party libraries have been updated. The
improvements in those libraries are too many to list here, although it
many cases not immediately apparent when using dCache.

#### Increase precision of timing gauges

The precision of timing gauges used by services like `srm` and
`pnfsmanager` has been improved.

### Admin changes

#### Admin shell got major face lift

The admin shell has been reworked entirely. We have taken several design
cues from the PostgreSQL shell,l `psql`, that most dCache admins are
familiar with.

Most notably, the command set is syntactically divided into built-in
commands affecting the shell itself and commands sent to connected
dCache cells. The former are all prefixed with a backslash while the
latter are not. This syntactic separation means that the shell commands
are available at any time, even when connected to a cell. E.g. the `cd`
command got replaced by `\c`, allowing one to connect to a different
cell without first having to disconnect from the current cell. Thus
there is no need for a `..` command.

The shell provides built-in help for the shell commands using `\?`. This
help is separate from the regular help output of the connected cell. The
latter is available using the new `\h` command (the classic `help`
command still works, but will not provide color highlighting).

All built-in shell commands provide full tab completion, including for
the arguments. E.g. `\c` provides tab completion on the cell name; it
will complete on all well-known cells, all local cells of the domain of
the connected cell, and on domain names of fully qualified addresses.

The new `\l` command lists cells. By default it lists well known cells
and cells of the domain of the connected cell. Wildcards for fully
qualified addresses however match all cells, e.g. `\l     p*@*` lists
all cells starting with the letter *p* (`\l     p*@` is a shorthand for
the same); and `\l     *@fooBar` lists all cells of domain `fooBar`
(with the shorthand `\l     @fooBar`).

As part of the face lift, both the shell prompt and the welcome message
got updated. The prompt is partially configurable using the new
`admin.prompt` property.

The old shell provided several utility commands at the "root" level.
These commands are no longer supported, but most of them can be mimicked
using some of the new commands introduced.

We provide a brief summary of the new commands in the tables below, as
well as a comparison of the old and new commands.

Cell address wildcards

|Pattern|Meaning|
|:------|:------|
|cell@domain|Fully qualified cell address.|
|cell|Well known cell.|
|GLOB@domain|All cells matching GLOB in domain.|
|cell@GLOB|All cells called cell in any domain matching GLOB.|
|cell@|Short for cell@\*, i.e. any cell called cell in any domain.|
|@domain|Short for \*@domain, i.e. any cell in the domain called domain.|
|@|Short for \*@\*, i.e. any cell in the entire system.|
|\*@domain1:cell@domain2|Source routed cell path. The asterix is not a wildcard!|

  
Admin shell commands illustrated

|Command|Meaning|
|:------|:------|
|`\?`|Show a summary of shell commands.|
|`\?` CMD|Show help of shell command CMD.|
|`\h`|Show a summary of commands of connected cell.|
|`\h` CMD|Show help of command CMD of connected cell.|
|`\c` CELL|Connect to well-known cell.|
|`\c` CELL@DOMAIN|Connect to fully qualified cell.|
|`\c` CELL USER|Connect to cell as user.|
|`\l`|List all well-known cells and cells local to the currently connected domain.|
|`\l` GLOB|List all cells matching the cell wildcard.|
|`\q`|Quit shell.|
|`\s` CELL CMD|Send CMD to cell.|
|`\s` CELL1,CELL2 CMD|Send CMD to cell1 and cell2.|
|`\s` GLOB CMD|Send CMD to all cells matching the cell wildcard.|
|`\sn` CMD|Send CMD to pnfs manager (name space).|
|`\sp` CMD|Send CMD to pool manager.|
|`\sl` PATH CMD|Send CMD to all pools containing the file described by PATH.|
|`\sl` PNFSID CMD|Send CMD to all pools containing the file described by PATH.|

Keyboard shortcuts

|Key|Meaning|
|:--|:------|
|\^C|Interrupt currently running command.|
|\^D|Quit shell.|
|\^R|Search history.|
|TAB|Complete command or argument.|

Tricks

|Command|What it does|
|:------|:-----------|
|`\s System@ kill                         System`|Restart all domains.|
|`\sl                         /path/to/file rep ls $1`|List all replicas of the file.|
|`\sl                         /path/to/file rep rm $1`|Remove all removable replicas of the file.|
|`\sl                         /path/to/file rep set sticky -o=me -l=60000 $1`|Add a 60s sticky bit owner by me to all replicas of the file.|
|`\sn cacheinfoof                         /path/to/file`|Get cache locations of the file.|
|`\sn cacheinfoof                         /path/to/file`|Get cache locations of the file.|

Replacements for legacy commands

|Old|New|Remark|
|:--|:--|:-----|
|`flags ls` *file*|`\sn flags ls` *file*|Output format is different.|
|`flags remove` *file* *key*|`\sn flags remove` *file* -*key*|  |
|`flags set` *file* *key* *value*|`\sn flags set` *file* -*key*=*value*|  |
|`getpoolbylink`|  |  |
|`modify poolmode                         enable` *pool*[,*pool*…]|`\s` *pool*,[*pool*…] `pool enable`|  |
|`modify poolmode                         disable` *pool*[,*pool*…][*code* [*message*]] [*options*]|`\s` *pool*,[*pool*…] `pool disable` [*code* [*message*]][*options*]|  |
|`p2p` *pnfsid* *source* *target*|`\s` *source* `migration cache                         -pnfsid=`*pnfsid* *target*|  |
|`p2p` *pnfsid*|`\sp replicate` *pnfsid* localhost|The new form is non-blocking.|
|`pnfs map` *path*|`\sn pnfsidof` *path*|  |
|`quota query`|  |  |
|`repinfoof` *file*|`\sl` *file* `rep                         ls $1`|Output format is different.|
|`set deletable` *pnfsid*|`\sn flags set` *pnfsid* `-d=true`  
 `\sl` *pnfsid* `ret set cached $1`|Note that neither form touches the sticky bit.|
|`set sticky` *file*|`\sl` *file* `rep                         set sticky $1 on`|  |
|`set sticky` *pnfsid* `-target=`*pool*|`\s` *pool* `rep                         set sticky` *pnfsid* on|No equivalent command when using path.|
|`set unsticky` *file*|`\sl` *file* `rep                         set sticky $1 off`|  |
|`set unsticky` *pnfsid* `-target=`*pool*|`\s` *pool* `rep                         set sticky` *pnfsid* off|No equivalent command when using path.|
|`uncache` *file*|`\sl` *file* `rep                         rm -force $1`|  |
|`uncache` *pnfsid* `-target=`*pool*|`\s` *pool* `rep                         rm -force` *pnfsid*|No equivalent command when using path.|

#### Admin shell provides better scripting support

The new admin shell detects whether it is connected to a pseudo
terminal. If it isn't, it typically means the input and output are
redirected such as when connecting from a script. In this case the admin
shell disables all styling: No welcome message, no prompt, no echo, no
color highlighting, no command history. This makes it much easier to
script the admin shell.

#### Admin shell supports Ctrl-C

The Ctrl-C key combination may now be used to unblock the shell and make
it available to read the next command. The default timeout has been
increased to 5 minutes as pressing Ctrl-C is now preferred.

Note that pressing Ctrl-C does not actually cancel the execution of the
command. The key combination merely cause the shell to stop waiting for
the result.

#### Admin shell automatically creates history file

Admin shell has had support for persistent command history for a long
time. This feature was however only enabled if one manually created the
file used to store the command history. This file is now created
automatically if it doesn't exist.

#### New legacy subsystem provides compatibility with old scripts

To support old scripts, the old shell is available as the `legacy` ssh
subsystem. One can connect to this subsystem by adding the
`-s     legacy` option to the ssh command line.

We do however recommend that scripts are updated to use the new shell.

#### Admin no longer hard-codes cell addresses

Several new properties allow communication details to be configured:

    admin.service.poolmanager=${dcache.service.poolmanager}
    admin.service.poolmanager.timeout=30000
    (one-of?MILLISECONDS|SECONDS|MINUTES|HOURS|DAYS)admin.service.poolmanager.timeout.unit=MILLISECONDS

    admin.service.spacemanager=${dcache.service.spacemanager}
    admin.service.spacemanager.timeout=30000
    (one-of?MILLISECONDS|SECONDS|MINUTES|HOURS|DAYS)admin.service.spacemanager.timeout.unit=MILLISECONDS

    admin.service.pnfsmanager=${dcache.service.pnfsmanager}
    admin.service.pnfsmanager.timeout=30000
    (one-of?MILLISECONDS|SECONDS|MINUTES|HOURS|DAYS)admin.service.pnfsmanager.timeout.unit=MILLISECONDS

    admin.service.acm=${dcache.service.acm}
    admin.service.acm.timeout=30000
    (one-of?MILLISECONDS|SECONDS|MINUTES|HOURS|DAYS)admin.service.acm.timeout.unit=MILLISECONDS

#### Many admin shell commands now accept hex and octal values

Many admins shell commands now accept hexadecimal and octal numeric
arguments. Hexdecimal arguments must be prefixed by `0x` and octal
arguments by `0`.

#### Improved pcells compatibility

Several changes have been made to simplify maintaining compatibility
with the pcells GUI. As part of these changes, the admin service now
redirects pcells requests to the appropriate dCache cell, allowing
pcells to be used even if `pnfsmanager`, `poolmanager` or `spacemanager`
use non-default names.

#### Improved cell message communication

The internal cell communication has been heavily restructured. Most of
these changes are internal and - except for maybe minor latency
improvements - are invisible to the user. There are however also several
bug fixes to corner cases, as well as improvements in logging and
routing failure handling. One may observe more routing errors being
logged; such errors happened in previous versions too, but were not
always logged.

Several changes to the message routing logic were made. These changes
mostly affect non-default topologies, enabling use cases not possible in
previous versions. When matured, these use cases will be documented
separately.

Route maintenance logic has been updated. The most immediately
observable changes are the new output format of the `route` command in
the `System` cell, the fact that route deletion requires the full route
to be specified, and that duplicate routes are allowed (messages are
routed through the first route that matches).

#### Admin ping command provides RTT

The `ping` command in the `System` cell is now blocking and measures the
round trip time.

#### Webadmin HTML contains dCache version

The webadmin now includes a meta tag reporting the dCache version.

#### Webadmin features improved navigation bar

The navigation bar now disables the links to disabled pages.

#### Webadmin no longer exports the info service

The `info` service can no longer be access through webadmin. The `info` service 
is still exposed through the `/info` path of the `httpd` service and this is the 
preferred interface for scripts and other services to query the info service.

#### Updated httpd service configuration

The `httpd` service has been heavily refactored. The `set update`
command that was lost in previous releases is once again exposed in the `admin` shell.
The arguments to the `set webapp` command have changed, which means that sites 
having a custom `httpd.conf` file likely have to adjust the configuration on update.

The `httpd.container.webapps.tmp-dir` property is obsolete as WAR files are no longer unpacked.

### SpaceManager changes

#### Robust flush notification for spacemanager

Similar to the delete notification, the unreliable flush notification
previously sent by pools was replaced by a reliable notification from
`pnfsmanager`. The new `pnfsmanager.enable.space-reservation` controls
whether these notifications are sent. One consequence is that if
`spacemanager` is down, flush to tape will halt.

#### dCache expects spacemanager by default

The defaults have changed such that it is assumed that space
reservations are supported unless explicitly disabled. In other words,
the property `dcache.enable.space-reservation` now defaults to true.

This change has been made because it is a safer default. In a system
without space management, forgetting to set the above property to false
simply results in a non-functioning system. In a system with space
management, forgetting to set this property to true results in lost
notifications, causing silent database inconsistencies. Thus `true` is
the safer default.

Sites without space manager will have to set
`dcache.enable.space-reservation` to false on all doors and other head
nodes.

As before, one has to have a `spacemanager` instance in the layout file
to support space reservations. The change in defaults is merely whether
the rest of dCache expects space manager to be running or not.

#### Space manager has been renamed

The default space manager name has been changed to `SpaceManager`. In
previous versions, the default was `SrmSpaceManager`, but since the
space manager service is independent of the SRM service, this name was
misleading. For compatibility, a temporary alias was added to allow the
admin service to use both names.

#### Space manager supports less intrusive link group update command

The space manager `update     link groups` command no longer blocks the
space manager while the link groups are updated. The command returns
once the updated information is being used.

#### Improved shutdown logic in space manager

If space manager discards requests during shutdown, the requesting
service is now notified about the discarded messages. This allows for
faster recovery after a space manager service restart.

#### Space manager provides better select failures

Space manager now provides better error messages when unable to find a
suitable link group of write pool. It also provides better error code to
distinguish permission and configuration errors from other errors. Doors
like DCAP, NFS and FTP make use of these error codes to decide whether
to retry requests or which error codes to send to clients.

#### Space manager changes authorization policy for unowned reservations

In previous releases, unowned reservations could be released by anybody.
Since this is unlikely to be desirable, this release changes the
semantics such that reservations without an owner can only be released
by the admin.

#### Link group authorization by GID

The link group authorization file now supports authorization by GID.
This allows space manager to be use with anonymous protocols like plain
DCAP, plain xrootd and plain NFS.

### Transfermanager changes

#### Transfer Manager service removes runtime database configuration

The admin commands `set     dbUrl`, `set     dbUser` and `set dbpass`
have been dropped.

#### Transfer Manager changes

Transfer Manager now generates IDs based on the current time rather than
by taking the next SRM request ID. This means the transfermanagers
service no longer needs access to the SRM database.

### InfoProvider changes

#### Info provider supports configurable service names

The names of the `pinmanager` and `poolmanager` services used by the
info provider are now configurable.

#### Info provider generates timestamp attributes

Info provider now adds a `GLUE2EntityCreationTime` attribute to all the
GLUE 2 entities it creates.

#### Network interface scope published

The info service enumerates the network interfaces of all doors.
Previously these interfaces were ordered to list an external interface
first. With this release, the scope of the interface (e.g. global, site
local, link local, etc). is included instead.

The info-provider is updated to make use of the scope to only publish
global network interface and to export all of them.

### PinManager changes

#### Pinmanager provides better error messages

Pin manager now distinguishes between several different causes for
failing to pin a file.

### PoolManager & Pool changes

#### Pool migration module maintains last access time

The new `-atime` option to the migration commands allow migrated files
to keep the original access time. This allows the garbage collection
order to be maintained.

#### Pool garbage collector implements strict LRU

Previous versions of the sweeper implemented an approximation of
least-recently-used (LRU). Starting with version 2.13 it implements real
LRU.

#### New pool safe-guards in case of repository errors

Additional safe guards to automatically disable a pool in case of fatal
errors have been added. A pool will now also stop flushing files to tape
if it has been marked dead.

#### Better HTTP error handling in pools

Previous versions would disconnect the client on every error. Starting
with version 2.13, the pool will try to keep the connection alive.

#### Pool Berkeley DB meta data format change

The Berkeley DB library used by pools for meta data has been upgraded.
The update is forward compatible in that the new version can read the
old meta data. However once upgraded, the pool cannot be downgraded to
previous versions. If a downgrade is required, the meta data needs to
first be converted to the file backend before the downgrade, and then
back to the db backend after the downgrade.

#### Asynchronous pool selection

Pool selection requests in doors may now be asynchronous, meaning the
door does not need to maintain a thread for the duration of pool
selection. This is currently only utilized by the FTP door.

#### Pool improvements

Some pool error messages have been expanded slightly to give some
contextual information and to distinguish between otherwise identical
errors.

The migration module is idempotent, which means running the same command
twice has the same affect as running it once, assuming no pool has gone
offline between running the command. While being idempotent is a useful
feature, sometimes one might want multiple copies of a file, which
simply running the migration module command multiple times does not
achieve.

From dCache v2.11, the migration module commands now support the
`-replicas` option. This allows the admin to specify the desired number
of replicas. The pool will then create that number of replicas
sequentially.

Previously, the HSM interface processed all HSM requests sequentially:
one at a time, using a single core. This is particularly bad for staging
requests as preparing the pool for the staged file may require IO. From
dCache 2.11, the HSM now processes requests concurrently, which allows
CPU and IO activity to overlap and spreads this load over multiple
cores.

The output from admin commands `rh     ls`, `st     ls` and `rm ls` are
now sorted by creation time.

Previously, just before a pool starts accepting a new file, it will
request the expected checksum from pnfsmanager. In the majority of cases
there is no checksum, so this slows down dCache. From dCache 2.11, this
step is skipped; uploads are now (slightly) faster. A client may still
supply the file’s checksum prior to upload (using FTP) and dCache will
verify the checksum once the upload has completed.

#### Pool HSM commands provide helpful error messages

Thanks to Onno Zweers from SARA for contributing a patch that allows the
`hsm     set` and `hsm     unset` commands to provide more helpful error
messages.

#### Automatic gap calculation for small pools

Pools have a gap setting that reduces the risk of running out of space
during uploads. Once less space than the gap is free, pools will no
longer accept new files. In previous releases, small pools less than the
size of the default gap would be inaccessible, often leading to
confusion during testing. With this release, small pools automatically
reduce the gap.

#### Improved nearline storage SPI

The nearline storage service provider interface has been enhanced to
allow nearline storage drivers to query the logical path of a file being
flushed. The interface has also been extended such that it is easier to
report error codes on failures.

The packaging of the SPI has been changed to make it easier to develop
third party drivers without having to depend on the entire dCache code
base.

#### Improved network interface selection in pools

The logic used to select a network interface for a transfer has been
improved. This in particular affects partially multi-stacked
installations.

#### Consistent csm info output

With this release, the output format of the `csm info` command in pools
is consistent with the option values of the `csm     set` command.

#### Updated default checksum policy for new pools

Newly created pools now default to on-transfer checksumming rather than
on-write. For large files, the on-write checksum policy often leads to
timeouts and throughput issues.

The on-restore policy is now enabled by default.

#### Explicit port range for xrootd and http movers

The new configuration properties `pool.mover.http.port.{min,max}` and
`pool.mover.xrootd.port.{min,max}` allow the port range for HTTP and
xrootd to be controlled.

#### Precomputed repository statistics

The pool `rep ls -s` command used to recalculated per storage group
statistics on every invocation. For large pools this was quite slow,
leading to timeouts in the admin shell and in the statistics service.
With this release the result is precomputed.

#### Much improved support for control channel free protocols

dCache was born with protocols having seperate control channel and data
channels. This has proven to be challenge for protocols like HTTP and
xrootd that do not maintain a control channel to the door for the
duration of the transfer. In previous releases, there was no way for
clients using these protocols to know whether post-processing failed or
when it completed. Thus a client would be unaware of e.g. checksum
failures or failures to register a file in Chimera, and a client that
queried the door directly after it completed uploading would typically
be met with an error.

Starting with this release, xrootd and http uploads will block until
post-processing has completed. Any errors are propagated back to the
client by the pool, and once the client is disconnected, it is
guaranteed that the file is fully registered in Chimera.

#### Reduced memory consumption in pools with Berkeley DB backend

The memory consumption of pools using the Berkeley DB backend has been
drastically reduced. The on-disk representation is unchanged.

#### Pool manager drops obsolete configuration command

The obsolete `psu     set linkGroup attribute` command has been removed.
The command had no effect. Sites are encouraged to check the pressence
of this command in their poolmanager.conf file before updating. If
pressent, the command can be removed by running `save` in the
poolmanager service.

### Chimera / NameSpace changes

#### pnfsmanager improvements

As part of the general shutdown improvements, during shutdown the
pnfsmanager service will reject all queued requests and return a “No
Route To Cell” error. Requests that pnfsmanager has started processing
are given 500 ms to complete before being forcefully terminated. This
improvement was introduced in dCache 2.11.

#### Chimera cleans old leaked tag inodes upon upgrade

A bug in Chimera caused tag inodes to be leaked upong directory
deletion. The bug was fixed in 2.10, however to reduce the impact of the
fix, already leaked inodes were not deleted. When upgrading to 2.13,
such inodes are automatically removed. Depending on the number of tag
inodes, this procedure may take a while (from minutes to hours).

As always, if an `nfs` service has been placed ahead of the
`pnfsmanager` service in a domain, the database schema update has to be
triggered manually by running `dcache     database update` before
starting dCache.

#### Chimera relaxes permission checks on stat

In compliance with POSIX, Chimera no longer requires execute permissions
for the containing directory to stat a file.

#### Name space services provides timing gauges for Chimera calls

The `info` command of the `pnfsmanager` service has been extended to
provide timing information for Chimera calls.

#### Chimera file attribute access optimized

This release optimizes how file attributes are updated in Chimera. This
change reduces database load and latency on such updates.

Permission checks have been altered to always allow the owner of a file
to update its attributes.

In previous releases, the last access time in Chimera was updated by
pools when a pool is accessed. With this release, the atime is
maintained by doors. Besides being a more correct implementation of
atime, the new implementation is faster.

### Alarms changes

#### Embeddable alarms service

Threading issues preventing the alarm service from running in the same
domain as other services have been corrected so that it is now safe to
do so. Two new properties have been added:

    `alarms.limits.workers` # Number of worker threads for processing incoming logging events.
    `alarms.db.alarms-only                 `

Only store alarms to the persistent store

#### Alarms service improvements

During the transition from 2.10 to 2.13, the alarms service has received
substantial improvements. Most relevant improvements were performed on
dCache 2.11.

A domain may now host both the alarm service and other services. As a
consequence, the alarm-specific `logback-server.xml` file (located in
the `/var/lib/dcache/alarms/` directory for RPM packages) has been
removed. As log messages generated by the alarm service are handled in
the same fashion as other services, the `service.log` file has also been
removed.

Alarm definitions are now either predefined or custom. Predefined alarms
are automatically generated within dCache without requiring a match
against some regular expression; these are the alerts defined by the
dCache developers. Custom alarms are those that are must match some
regular expressions; these are defined by the admin and are comparable
to the alarms defined in earlier dCache releases.

All predefined alarms are sent at the ERROR level, so it is not
necessary to set `dcache.log.level.remote` to below this level for the
alarm service to receive all predefined alarms. It may still be
necessary to adjust this property if custom alarms match against
messages logged at a lower threshold. While this is supported, the same
performance caveats exist as with earlier releases.

It is no longer necessary to mirror the `alarms-custom-definitions.xml`
file between the alarms and httpd services. However, when using XML
storage, the requirement to provide some shared storage between the
nodes hosting alarm and httpd services continues.

The structure of the custom alarm definition or filter has been
simplified. It now consists only of the following elements:

1. type name 
2. keyWords 
3. regex 
4. regexFlags 
5. matchException 
6. depth 

Note that the predefined Severity level, along with logger and thread
ids, have been removed as filter attributes.

When an alarm fires, it is assigned one of four priorities (these
replace the severity concept in earlier versions). In decending order,
these are `critical`, `high`, `moderate` and `low`. By default, all
alarms have `critical` priority. This default priority may be altered in
the admin interface and individual alarms may be assigned a lower
priority through the priority map.

The priority map is stored in the `alarms-priority.properties` file,
which is located in the `/var/lib/dcache/alarms` directory in RPM
packages. The priority map may be adjusted either by editing this file
directly (and requesting dCache reloading it) or via the admin
interface.

Note that, while there is still a Severity column in the database, it is
no longer used.

The previous dCache shell commands to add, modify, remove and send
alerts are still present. Their options have been modified slightly to
reflect changes in this release. There is a new `dcache     alarm list`
command. This lists all predefined alarms.

The alarm service is now a well-known cell by default.

The alerts admin commands have been extended to allow creating custom
alerts and assigning priority to alerts. The commands
`predefined     ls` and `send` have also been added to the admin
interface; these behave as the similarly named `dcache` shell commands.

The optional alarm cleaner daemon now runs as part of the alarms
service, not in the httpd service. The default values for its properties
have been moved from `httpd.properties` to `alarms.properties` and
renamed accordingly.

There are a number of new and modified alarms properties which are
documented in the `alarms.properties` defaults file. Most significant of
these are those for enabling email notification, which no longer
requires direct editing of logback configuration files.

The alarms web page is basically the same, except that the “Severity”
label has been replaced by “Priority”. That filter choice affects only
alarms and not unmarked logging events. Since this is not a schema
attribute, it is not reflected in the actual table, but alarms will be
sorted there by priority first.

### Billing changes

The billing service no longer logs a warning on shutdown. This change
was performed on dCache 2.11. The billing log format has been extended
to allow InfoMessage log entries to include the session identifier. This
is particularly useful when understanding which SRM session is
responsible for billing entries. The default format is not altered, so
sites must adjust their configuration to include session information.

##### Scripts

The startup time for the different `dcache     billing` commands has
been reduced.

### GPlazma changes

Update gPlazma Login result printer to use a human-friendly label rather
than a dot-separated-number for the SHA–512-with-RSA algorithm.

The kpwd plugin, which supports the proprietary kpwd-format files, has
been updated to support multiple gids. Users with multiple gids have a
comma-separated list of gids instead of the single gid value, with the
primary gid the first gid in the list. A comma-separated list of gids is
valid for both `login` and `password` lines.

### Info changes

The info-provider groups together pools that GLUE can report together
for storage accounting. Such groups are called Normalized Access
Storages (NASes). Previously, info tried to group pools as much as
possible, which leads to confusion over the names. Now, a NAS represents
all the pools accessible by the same set of links.

### Replica Manager changes

Replica manager now correctly shuts down the database connection pool
when dCache is shutting down the domain.

### Door Changes

#### SRM service

##### Timing gauges support asynchronous processing

The `srm` provides gauges for measuring the executing time of various
request. Some of these requests are asynchronous and as a result
appeared to be much faster than they really are. The gauges have now
been extended to support asynchronous processing and the measured time
now reflects the true processing time of such request.

##### Scoped door selection in SRM

From dCache release 2.12, the SRM tries to take where the client
connected from into account when it selects a door.

The selection logic has been modified such that it will select a network
interface with the smallest possible scope as big as the scope of the
interface the client connected over. E.g. if the client connected on a
network interface with global scope, the SRM will select a door with
global scope. If the client connected over a site local interface, the
SRM will prefer a door with a site-local interface, but will fall back
to selecting an interface with global scope if necessary.

##### SRM enables history logging by default

The SRM supports writing a detailed log of all request state changes to
the SRM database. This information is very valuable when one wants to
reconstruct the events related to a transfer, for instance, when
debugging why a particular transfer failed.

In previous releases this feature was disabled by default due to fear
that it would overload the SRM database. Recent releases have however
improved database performance in SRM to the point that we now change to
enabling history logging by default.

At the same time we changed the default for when data is persistet to
the database. This means that although all state transitions are by
default recorded in the database, the database is not necessarily
updated upon every state transition. This further reduces the database
overhead of history logging.

##### Site specific srmPing attributes

The SRM protocol allows the srmPing reply to contain site specific
key-value pairs. With this release, custom key-value pairs may be added
in the dCache configuration using the `srm.ping-extra-info` properties.

##### srmGetSpaceTokens results are cached by SRM service

Clients such as FTS query the available space tokens prior to every
upload. This used to cause a query message to be sent to the
`spacemanager` service every time, with `spacemanager` querying its
database. Since the list rarely changes, this release introduces a
short-lived cache for this information.

##### srm DB improvements

The srm service now includes a self-throttling behaviour when clients
make more requests that the database can cope with. Previously to dCache
2.11, when overloaded, all database updates were dropped and a message
like `Persistence of     request <request> skipped, queue is too long`
is logged. The result is that the database could hold incomplete or
inaccurate information about SRM activity.

Such inaccuracy was acceptable as the database was used for debugging
information. However, with the adoption of a separate upload directory
and some support for recovering user activity after a restart, this is
no longer true. Moreover, such debugging information is likely very
useful when understanding any unexpected behaviour during an overload
situation, so needs to have some accuracy in particular under such
conditions.

SRM database updates naturally fall into two types: major and minor.
Major updates record new requests and transitions to a final state
(SUCCESS, FAILURE, CANCELLED); minor updates record changes to some
non-final state. In an overload situation, minor updates may be dropped
but major updates will always be processed. If necessary, processing the
SRM request will be delayed so that major updates are stored. This has a
secondary effect of creating a back-pressure on clients, slowing them
down and allowing the srm service to recover.

The admin command `ls` and `ls completed` have been refactored. There is
now a single command `ls` that accepts the option `-completed=<n>` to
limit the number of completed operations.

###### Closing SRM connections

With the adoption of Jetty v9, the GSI implementation has been rewritten
to use Java’s built-in SSL support in a more direct fashion. The most
immediate effect of this is that, by default, the shutdown sequence of
SSL and GSI connections has changed. Before, the server would simply
close its end of the network connection; now, the server issues a CLOSE
alert that notifies the client of the impending close before closing its
end of the connection.

Sending a CLOSE alert is part of the SSL v3 and subsequent TLS protocol
specifications. Previous versions of dCache were in error by not sending
such alerts.

Although most SSL/TLS clients expect the CLOSE alert, the dCache SRM
clients (prior to v2.10.7) considers any alert as an indicator that the
operation has failed. Depending on the client configuration, it might or
might not retry the operation several times if this happens. In either
case, the return code and output will reflect a failed operation,
whereas the operation may have succeeded.

This has been fixed with SRM-client release v2.10.7: this version
accepts the CLOSE alert but does not require it. It is compatible with
dCache v2.11 and earlier versions.

To allow a smooth transition, dCache includes the
`srm.enable.legacy-close` configuration property. When set to `true`
(the default) the srm service will close connections without first
issuing the CLOSE alert, in keeping with previous dCache behaviour.

Future versions of dCache will change the default and eventually remove
this option. Therefore sites are invited to upgrade their clients.

#### WebDAV service

##### Support for multiple httpd instances

With the improved pcells decoupling from dCache, it is now possible to
have several `httpd` services in the same dCache instance. One needs to
give each a unique name, but that's the only configuration tweak needed.

##### httpd uses JGlobus for certificate handling

When hosting the webadmin service, `httpd` supports HTTP over TLS.
dCache now uses JGlobus for TLS, or more specifically, for the
certificate handling required for TLS. The immediate consequence is that
one no longer has to import host and CA certificates into a Java
Keystore file. The `httpd` service can now read those directly from your
`/etc/grid-security` directory.

##### WebDAV doors always use JGlobus for https

The WebDAV door previously had the choice of two different https
implementations: The plain `https` setting would use Java’s TLS
implementation, while the `https-jglobus` setting would use the TLS
implementation for JGlobus.

In recent years, the two choices actually both used the TLS
implementation bundled with Java. The difference between the two was
merely how they dealt with certificate handling and internal
abstractions. In dCache 2.11, the two code bases were further
consolidated to the point that in this release we adopt JGlobus
certificate handling for the plain `https` setting. The `https-jglobus`
setting is deprecated.

One consequence is that the WebDAV door now uses the certificates in
`/etc/grid-security/` rather than those imported using the
`dcache     import` commands.

One of the benefits of the JGlobus certificate handling is that it
supports VOMS attributes.

##### WebDAV door supports parameterised directory templates

The HTML generated by the WebDAV door is cusomizable using a templating
language. Starting with this release, configuration properties prefixed
with `webdav.templates.config` are exposed to the template language.
Several such properties have been predefined. Consult the information in
the `webdav.properties` file for details.

Sites that have already customized the template will likely have to
update the template.

##### WebDAV blocks access to files being uploaded

When generating a directory listing, the WebDAV door now greys out files
that are still being uploaded, preventing the user from clicking a
download link that would fail anyway.

Sites that have customized the HTML template will likely have to update
the template.

##### Support for custom HTTP response headers

Support has been added to the webdav door and the pool so that custom
HTTP response headers may be configured. To demonstrate this, the Server
response header now has the value `dCache/<version>` where `<version>`
is the installed dCache version.

##### Robust delete notification for spacemanager, replica, and pinmanager

The `spacemanager`, `replica`, and `pinmanager` services rely on
receiving notifications for when files are deleted. Previously, these
were sent by both `pnfsmanger` and `cleaner`, but both used unreliable
notifications. This in particular affected `spacemanager`: If the
`spacemanager` was shut down while `pnfsmanager` or `cleaner` were
running, the services could quickly get out of sync, resulting in leaked
files in `spacemanager`.

This notification has now been replaced by a robust notification from
`cleaner`. The deleted file is not removed from the trash table until
the receiving services have been notified and have confirmed the
notification. Whether the notification is send is controlled by the new
`cleaner.enable.space-reservation` and `cleaner.enable.replica`
settings. These settings must be in sync with whether space management
or replica management is used in this dCache instance.

It is assumed that every dCache instance has a `pinmanager`. If you
don't have one, the easiest solution is to add this service upon
upgrade.

#### XRootD service

##### Third party xrootd plugins have to be ported to Netty 4

Any third party channel handler plugin has to be ported to Netty 4 and
xrootd4j 2 before it is compatible with dCache 2.12. Please contact the
third party vendor for further details.

Information to developers: If you develop channel handler plugins, it is
of utmost importance that your plugins do not perform blocking
operations on the Netty threads. Any blocking operation has to be
offloaded to a thread pool maintained by the plugin. Please contact
support@dcache.org for details.

##### Asynchronous open for xrootd

Pool selection in dCache is unbounded in that it may trigger staging
from tape or internal replication of files. Yet many xrootd clients have
very short timeouts. To avoid that clients time out during staging,
dCache xrootd doors now use the asynchronous reply format of the xrootd
protocol. In this format, the server immediately responds to an
kXR\_open request, telling the client that the actual response will be
provided asynchronously at some point in the future.

#### NFS service

##### NFS door improvements

The NFS door and mover now shows correct units, nanoseconds (ns), when
showing performance statistics for the different NFS operations.

The database query used to gather information for `df` command on an NFS
mount has been improved. However, this remains an expensive operation.

nfs4j has been updated, which adds new improvements such as:

* Support for the `all_root` export option. This treats all requests
from the specified client machines as having come from the root user.
Created files and directories have the same ownership as the parent
directory.

* We ensure that file attributes (e.g., file size) are always
up-to-date. Previously, zero file size was reported for a short period
after closing a file written through NFS.

* Some performance boosts when checking filenames.

* Make deletes through NFS v3 faster.

##### NFS doors use NFS to access pool data

An NFS door can act as a data-proxy between the NFS client and the
pools. In previous releases, the NFS door internally used the DCAP
protocol between the door and pools. Starting with this release, NFS
doors use the NFS 4 protocol instead. The use of NFS provides more
robust proxy support in case of network failures.

As part of this change, the lifetime of NFS movers has changed.

Additionally, NFS has seen many performance improvements in this
release.

##### New NFS export options

Added a `nopnfs` export option to doors to force the use of proxy mode.

Added `anonuid` and `anongid` to map all accesses to a particular UID
and GID, e.g. `/data     hostA(rw,all_squash,anonuid=17,anongid=18)`.

##### New admin commands for NFS service

Previous versions showed information about all transfers, pools and
clients as part of its info output. This tended to clutter the output
and also caused a lot of unnecessary information to be collected by the
`info` service.

With this release, NFS doors no longer show this information in the
`info` output. Instead that information can be queried using several new
admin commands:

    show clients [<host>]  # show active NFSv4 clients
    show pools [<pool>]  # show pools to pNFS device mapping
    show proxyio # show proxy-io transfers
    show transfers # show active transfers

##### New thread pool settings for NFS service

The NFS service provides the following new settings for controlling the
message thread pool size:

    nfs.cell.limits.message.threads.max=8
    nfs.cell.limits.message.queue.max=1000

#### Further Door Changes

##### More uniform access logs

Most doors in dCache support generating a so called access log that
records any interaction with clients talking to the doors. The logs
share many similarities, but used slightly different formatting and
attribute nasafmes for the same concepts. This has now been cleaned up.
If you developed parsers for the access logs, you most likley have to
update these.

##### Log FTS-specific information

FTS, since v3.2.34, will send the Job-ID, File-ID and retry-count to
dCache. Here is an example of this information:
`job-id=bb62f96e-23da-48c1-bd6f-e0737588733b;file-id=996117;retry=0`.

How FTS provides this information varies depending on the protocol. For
SRM and WebDAV, a special HTTP header is used; for FTP the information
is part of the `CLIENTINFO` command. In all cases, the information is
available in the access log (`<domain>.access` file).

The following shows FTS sending an SRM `prepareToPut` command:

    level=INFO ts=2015-06-15T09:13:18.035+0200
    event=org.dcache.srm.request [...] request.method=srmPrepareToPut
    [..] request.token=-2147481808 status.code=SRM_SUCCESS
    client-info=job-id=bb62f96e-23da-48c1-bd6f-e0737588733b;file-id=996117;retry=0
    user-agent="fts_url_copy/3.2.34 gfal2/2.10.0 srm-ifce/1.23.1
    gSOAP/2.7"

The following shows the same information in an FTP session:

    level=INFO ts=2015-06-15T09:13:19.769+0200
    event=org.dcache.ftp.response
    session=door:GFTP-prometheus-AAUYiTEsOCg command="ENC{SITE
    CLIENTINFO scheme=gsiftp;appname=\"fts_url_copy\";appver=\"3.2.34
    (gfal2
    2.10.0)\";job-id=bb62f96e-23da-48c1-bd6f-e0737588733b;file-id=996117;retry=0}"
    reply="ENC{250 OK}"

The FTP session identifier (`door:GFTP-prometheus-AAUYiTEsOCg` in above
example) allows discovery of all commands and corresponding responses
for this attempt to upload a file.

`The     inclusion of this information allows correlation of dCache activity     against FTS activity, which may prove useful when diagnosing     problems.`

##### New threading model for HTTP and xrootd movers

The threading model of HTTP and xrootd movers has changed due to an
upgrade to the Netty 4 I/O framework. Socket I/O and disk I/O are now
served by the same threads. The `pool.mover.http.disk-threads` property
is deprecated and has been replaced by `pool.mover.http.threads`.
`pool.mover.http.socket-threads` is marked as forbidden, while
`pool.mover.http.memory-per-connection` and `pool.mover.http.memory` are
marked as obsolete.

Xrootd doors are build on the same Netty I/O frame works as the HTTP and
xrootd movers. The upgrade to Netty 4 has caused the
`xrootd.max-total-memory-size` and `xrootd.max-channel-memory-size` to
be obsolete.

##### On the fly checksum calculation for NFS and xrootd

Both NFS and xrootd uploads now support calculating checksums during
upload when enabled in the pool’s checksum policy. This avoids the long
pause at the end of the upload that is usually caused by calculating the
checksum after the transfer.

Databases
=========

Schema Management
-----------------

Except for the `srm` and `replica` services, all databases in dCache are
managed through the *liquibase* schema management library. This now also
includes the `spacemanager` service. The SQL scripts for creating the
Chimera schema by hand are no longer shipped with dCache.

Schema changes are automatically applied the first time a service using
the schema is started. Such changes may also be applied manually before
starting dCache using the `dcache database update` command. It will
update the schemas of all databases used by services configured on the
current node.

For even more hands-on schema management, the
`dcache database showUpdateSQL` command was added. It outputs the
SQL that would be executed if `dcache` database update was executed. 
The emitted SQL may be applied manually to the database, thereby updating the schema.

Before downgrading dCache, it is essential that schema changes are rolled 
back to the earlier version. This needs to be done *before* installing the 
earlier version of dCache. Schema changes can be rolled back using the 
`dcache database rollbackToDate` command. It is important to remember 
the date the schema changes where applied in the first place so that 
it can be rolled back to the version prior to the upgrade.

*NOTE*  The `nfs` service does not automatically apply schema changes 
to Chimera - only the `pnfsmanager` service does that. `nfs` will however 
check whether the Chimera schema is current and refuse to start if not. 
One consequence of this is that if `nfs` and `pnfsmanager` run in the 
same domain, either `nfs` must be placed after `pnfsmanager` in  the 
layout file or the changes must be applied manually using `dcache database update`.

Schema Changes
--------------

### New defaults for database names and owners

The defaults for database names are now such that each logical database
is stored separately. Except for Chimera, this means that each RDBMS
enabled service uses its own database. The default name is that of the
service, e.g. `srm` stores its data in a database called `srm`, and the
`spacemanager` service stored its data in a database called
`spacemanager`.

The default owner has been changed to `dcache` for all databases,
including Chimera.

New common configuration properties control the defaults for all
databases, but per service overrides are of course possible:

    #  ---- Default host of RDBMS used by various services.
    #
    #   Various services need an RDBMS. Each service is recommended to use its
    #   own logical database, but these may or may not be hosted in the same RDBMS
    #   instance. This setting defines the host of the RDBMS used.
    #
    dcache.db.host = localhost

    #  ---- Default RDBMS user account by various services.
    #
    dcache.db.user = dcache

    #  ---- Default password of RDBMS used by various services.
    #
    dcache.db.password =

    #  ---- Default password file for RDBMS used by various services.
    #
    #   Follows the format of the PostgreSQL pgpass file format. Avoids putting
    #   the passwords in the dCache configuration files.
    #
    dcache.db.password.file =

Upon upgrade, existing sites should update the settings to match the
previous setup:

    srm.db.name = dcache
    srm.db.user = srmdcache

    spacemanager.db.name = dcache
    spacemanager.db.user = srmdcache

    pinmanager.db.name = dcache
    pinmanager.db.user = srmdcache

    transfermanagers.db.name = dcache
    transfermanagers.db.user = srmdcache

    replica.db.name = replicas
    replica.db.user = srmdcache

    chimera.db.name = chimerac
    chimera.db.user = chimera

    billing.db.name = billing
    billing.db.user = srmdcache

    alarms.db.name = alarms
    alarms.db.user = srmdcache

### chimera

Changes on Chimera can be found at [Chimera / NameSpace
changes](#Chimera%20/%20NameSpace%20changes|outline).

Also, refer to [New defaults for database names and
owners](#New%20defaults%20for%20database%20names%20and%20owners|outline)
for database changes related on Chimera.

Upgrade Procedure
=================

Steps
-----

1. If you use SRM, update to the latest SRM client version available.
2. Upgrade to the latest patch level release of the dCache release being in use.
3. Prior to upgrading, run `dcache check-config` and fix any warnings. Information about properties marked deprecated, obsolete or forbidden in version 2.10 or earlier has been dropped in 2.13. Failing to do this before upgrading will mean that you will not receive warnings or errors for using an old property.
4. If the node relies on any databases (you may check the output of `dcache database ls` to recognize the services that do), then tag the current schema version by running `dcache     database tag dcache-2.10`.
5. If you have installed any third party plugins that offer new services (that you have instantiated in the layout file), then remove these and check with the vendor for updated versions.
     * CMS-TFC Plugins can be downloaded from [XRootD CMS-TFC Releases](https://www.dcache.org/downloads/xrootd4j/index.shtml) in [dCache.org](https://www.dcache.org/).
          * Package [xrootd4j-cms-plugin-1.3.7-1.noarch.rpm](https://www.dcache.org/downloads/xrootd4j/1.3/xrootd4j-cms-plugin-1.3.7-1.noarch.rpm) is actually working with dCache 2.13
     * ATLAS N2N Plugins can be downloaded from the [WLCG Repository](http://linuxsoft.cern.ch/wlcg/sl6/x86_64/).
          * Package [dcache-xrootd-n2n-plugin-6.0.7-0.noarch.rpm](http://linuxsoft.cern.ch/wlcg/sl6/x86_64/dcache-xrootd-n2n-plugin-6.0.7-0.noarch.rpm)is actually working with dCache 2.13.
     * XRootD Monitoring plugins can be found in the [WLCG Repository](http://linuxsoft.cern.ch/wlcg/sl6/x86_64/).
          * Package [dcache-plugin-xrootd-monitor-7.0.0-0.noarch.rpm](http://linuxsoft.cern.ch/wlcg/sl6/x86_64/dcache-plugin-xrootd-monitor-7.0.0-0.noarch.rpm) is actually working with dCache 2.13
6. Run `dcache services` and compare the services with the [table listing changed services](#Table1|table). If any of those services are used, replace them with the suggested alternative after upgrading.
7. Ensure that `Java     8 `is installed as your *default* Java (or *unique*). Otherwise, please remove/substitute `Java     7` & install `Java     8`.
8. Install the dCache 2.13 package using your package manager.
9. Reset `/etc/dcache/logback.xml` to the default file. How this is done depends on your package manager: Debian will ask during package installation whether you want the new version from the package (say yes). RPM will leave the new version as `/etc/dcache/logback.xml.rpmnew` and expects you to copy it over.
10. If you used either `head`, `pool`, or `single` as the layout name, you should check that the package manager hasn't renamed your layout file.
11. Run `dcache check-config`. You will receive an error for any forbidden property and a warning for any deprecated or obsolete property. You will receive errors about unrecognized services defined in the layout file. You will also receive information about properties not recognized by dCache - if these are typos, fix them. Fix any errors and repeat.
     As a bare minimum, the following changes commonly have to be applied during this step:
          * Remove deprecated services.
          * Replace the `srmspacemanager` service for `spacemanager``.`
          * `dcache.enable.space-reservation`` ``is set to ``true`` ``by default. Check if this needs to be disabled for specific services.`
12. Verify that your HSM script can handle the `remove` command. If not, update the HSM script or disable the HSM cleaner.
     * Sites using [Enstore](http://www.fnal.gov/docs/products/enstore/) will have to set c`leaner.enable.hsm=false``     ``in     the ``cleaner``     ``service.`
13. If you have customized the look of the `webdav` door, you should reset the style to the default and reapply any customizations.
14. If you are using [pCells](https://www.dcache.org/downloads/gui/index.shtml), please update to the latest version. Check the [GUI Download Page](https://www.dcache.org/downloads/gui/index.shtml) in order to see which versions work with dCache 2.13.
15. If the node relies on any databases, then update the schemas by running `dcache database update`.
16. Start dCache and verify that it works.
17. Run `dcache check-config` and fix any warnings.

Note that some properties that were previously used by several services
have to be replaced with a configuration for each service type. Check
the documentation in `/usr/share/dcache/defaults` for any property you
need to change.

Also note that some properties have changed values or are inverted. The
deprecated properties retain their old interpretation, however when
replacing those with their new names, you have to update the value. In
particular boolean properties that used to accept values like *yes*,
*no*, *enabled*, *disabled*, now only accept *true* and *false*.

# Frequently Asked Questions

## \<Question 1\>

## \<Question 2\>

.

.

.

##`<Question     N>`
