# Current Version
Stable V3

V3.5 under testing

# Preception Results
Audit results for Preception, my AI tool.
I tried to name it perception but made a typo, thought it was cool so left it this way XD

|Contest|Submitted Issues|Valid Issues|Placement|Note|
|---|---|---|---|---|
|CurrentSui|5M|3M|-|Tested with Preception V1|
|Intuition|2M|1M|4th|Tested with Preception V1|
|Injective|7M|-|-|Tested with Preception V1|
|Base Azul|3H/5M|1C/3M/2L/1I|-|V3|
|K2|4H/21M|-|-|V3|

# Other Notable Results
- 8 submitted issues in the Base Azul contest, 7 valid. 1 Critical, 3 Medium, 2 Low and 1 Insight.
- 25 submitted issues in the K2 contest, and 1 QA report with 110 entries. The contest is still being judged.

# Past Contest Analysis
## Table

| Name                 | Size(nsloc) | Platform | Past/New | Total Preception Issues | Total Report Issues | Coverage | Note            |
| -------------------- | ----------- | -------- | -------- | ----------------------- | ------------------- | -------- | --------------- |
| DODO Cross Chain DEX | 1600        | Sherlock | Past     | 12H/6M/35I              | 5H/12M              | 14/17    | No human triage, tested with Preception V3, V2 had 10/14 coverage |
| Centrifuge v3.1      | 8900        | Sherlock | Past     | 2H/13M                  | 1H/10M              | 4/10     | No human triage, tested with Preception V3 |
| PeaPods | 9000+? | Sherlock | Past | 6H/55M/80I | 7H/34M | 26/41 | No human triage, tested with Preception V3 |
| Superfluid Locker System | 2000 | Sherlock | Past | 5H/6M/25I | 2H/6M | 6/8 | No human triage, tested with V3. Only did half of the entire process as the rest half will likely not cover new issues|
| Velocimeter | 4000 | Sherlock | Past | 15M/52I | 9H/9M | 11/18 | No human triage, tested with V3. |

## Note
### Duplication Rule
- Same root cause + similar impact/attack vector
- Only strict match

### False Positives/Triaging
Normally in a real audit/contest, issues will be triaged by me. For those past contest test runs, there are no human triaging. It could be possible that, after triaging, some issues will be removed/filtered out. Contest issues have weird validation/invalidation bars, and they are usually dependent on judges. So the coverage here should be used as a reference, not absolute truth.

Also since only High/Medium issues are compared, so uncovered issues don't necessarily mean they are false positives, they could be downgraded to Low/Info in reality. 

### What happened to Preception V2?
Did some contests with V2(Monetrix, XRP, etc), but the execution was lacking. I liked my idea, but it wasn't working out as I anticipated.
