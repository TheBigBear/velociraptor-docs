---
title: Server.Import.CuratedSigma
hidden: true
tags: [Server Artifact]
---

This artifact allows importing curated Sigma rules from
https://sigma.velocidex.com

Collect this artifact on the server to automatically import or
update these artifacts.


<pre><code class="language-yaml">
name: Server.Import.CuratedSigma
description: |
  This artifact allows importing curated Sigma rules from
  https://sigma.velocidex.com

  Collect this artifact on the server to automatically import or
  update these artifacts.

type: SERVER

required_permissions:
- SERVER_ADMIN

parameters:
  - name: PackageNames
    type: multichoice
    default: '["Velociraptor Hayabusa Ruleset"]'
    choices:
      - Velociraptor Hayabusa Ruleset
      - Velociraptor Hayabusa Live Detection
      - Velociraptor ChopChopGo Ruleset (Linux)

  - name: Prefix
    description: Add this prefix to imported artifacts
    validating_regex: '^[a-zA-Z0-9_.]*$'

sources:
  - query: |
      LET URLlookup = dict(
        `Velociraptor ChopChopGo Ruleset (Linux)`="https://sigma.velocidex.com/Velociraptor-ChopChopGo-Rules.zip",
        `Velociraptor Hayabusa Ruleset`="https://sigma.velocidex.com/Velociraptor-Hayabusa-Rules.zip",
        `Velociraptor Hayabusa Live Detection`="https://sigma.velocidex.com/Velociraptor-Hayabusa-Monitoring.zip")

      SELECT * FROM foreach(row=PackageNames,
                            query={SELECT * FROM
                                Artifact.Server.Import.ArtifactExchange(
                                Prefix=Prefix,
                                ArchiveGlob="*.yaml",
                                ExchangeURL=get(item= URLlookup, member= _value))})

</code></pre>

