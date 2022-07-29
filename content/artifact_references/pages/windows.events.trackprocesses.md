---
title: Windows.Events.TrackProcesses
hidden: true
tags: [Client Event Artifact]
---

This artifact uses sysmon and pslist to keep track of running
processes using the Velociraptor process tracker.

The Process Tracker keeps track of exited processes, and resolves
process callchains from it in memory cache.

This event artifact enables the global process tracker and makes it
possible to run many other artifacts that depend on the process
tracker.


```yaml
name: Windows.Events.TrackProcesses
description: |
  This artifact uses sysmon and pslist to keep track of running
  processes using the Velociraptor process tracker.

  The Process Tracker keeps track of exited processes, and resolves
  process callchains from it in memory cache.

  This event artifact enables the global process tracker and makes it
  possible to run many other artifacts that depend on the process
  tracker.

type: CLIENT_EVENT

parameters:
  - name: AlsoForwardUpdates
    type: bool
    description: |
      If set we also send process tracker state updates to
      the server.
  - name: MaxSize
    type: int64
    description: Maximum size of the in memory process cache (default 10k)

  - name: SysmonFileLocation
    description: If set, we check this location first for sysmon installed.
    default: C:/Windows/sysmon64.exe

tools:
  - name: SysmonBinary
    url: https://live.sysinternals.com/tools/sysmon64.exe
    serve_locally: true

  - name: SysmonConfig
    url: https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml
    serve_locally: true

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'

    query: |
      // Make sure sysmon is installed.
      LET _ <= SELECT * FROM Artifact.Windows.Sysinternals.SysmonInstall(
         SysmonFileLocation=SysmonFileLocation)

      LET UpdateQuery =
            SELECT * FROM foreach(row={
              SELECT *,
                     get(member='EventData') AS EventData
              FROM watch_etw(guid='{5770385f-c22a-43e0-bf4c-06f5698ffbd9}')
            }, query={
              SELECT * FROM switch(
              start={
                SELECT EventData.ProcessId AS id,
                       EventData.ParentProcessId AS parent_id,
                       "start" AS update_type,
                       EventData + dict(
                           CreateTime=EventData.UtcTime,
                           Hashes=parse_string_with_regex(regex=[
                             "SHA256=(?P<SHA256>[^,]+)",
                             "MD5=(?P<MD5>[^,]+)",
                             "IMPHASH=(?P<IMPHASH>[^,]+)"],
                           string=EventData.Hashes)
                       ) AS data,
                       EventData.UtcTime AS start_time,
                       NULL AS end_time
                FROM scope()
                WHERE System.ID = 1
              },
              end={
                SELECT EventData.ProcessId AS id,
                       NULL AS parent_id,
                       "exit" AS update_type,
                       dict() AS data,
                       NULL AS start_time,
                       EventData.UtcTime AS end_time
                FROM scope()
                WHERE System.ID = 5
              })
            })

      LET SyncQuery =
              SELECT Pid AS id,
                 Ppid AS parent_id,
                 CreateTime AS start_time,
                 dict(
                   Image=Exe,
                   CommandLine=CommandLine,
                   CreateTime=CreateTime) AS data
              FROM pslist()



      LET Tracker <= process_tracker(
         enrichments=[
           '''x=>if(
                condition=NOT x.Data.VersionInformation,
                then=dict(VersionInformation=parse_pe(file=x.Data.Image).VersionInformation))
           ''',
           '''x=>if(
                condition=NOT x.Data.OriginalFilename OR x.Data.OriginalFilename = '-',
                then=dict(OriginalFilename=x.Data.VersionInformation.OriginalFilename))
           '''],
        sync_query=SyncQuery, update_query=UpdateQuery, sync_period=60000)

      SELECT * FROM process_tracker_updates()
      WHERE update_type = "stats" OR AlsoForwardUpdates

```