# preception-results
Audit results for Preception, my AI tool.
I tried to name it perception but made a typo, thought it was cool so left it this way XD

|Contest|Submitted Issues|Valid Issues|Placement|
|---|---|---|---|
|CurrentSui|5M|3M|-|
|Intuition|2M|1M|4th|
|Injective|7M|-|-|

# Past Contest Analysis
## Table

| Name                 | Size(nsloc) | Platform | Past/New | Total Preception Issues | Total Report Issues | Coverage | Note            |
| -------------------- | ----------- | -------- | -------- | ----------------------- | ------------------- | -------- | --------------- |
| DODO Cross Chain DEX | 1600        | Sherlock | Past     | 12H/6M/35I              | 5H/12M              | 14/17    | No human triage |
| Centrifuge v3.1      | 8900        | Sherlock | Past     | 2H/13M                  | 1H/10M              | 4/10     | No human triage |
| PeaPods | 9000+? | Sherlock | past | 6H/55M/80I | 7H/34M | 26/41 | No human triage |

## Note
### Duplication Rule
- Same root cause + similar impact/attack vector
- Only strict match

### False Positives/Triaging
Normally in a real audit/contest, issues will be triaged by me. For those past contest test runs, there are no human triaging. It could be possible that, after triaging, some issues will be removed/filtered out. Contest issues have weird validation/invalidation bars, and they are usually dependent on judges. So the coverage here should be used as a reference, not absolute truth.

Also since only High/Medium issues are compared, so uncovered issues doesn't necessarily mean they are false positives, they could be downgraded to Low/Info in reality. 

