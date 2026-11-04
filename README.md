# roschema-demo

A tiny ROS 2 interface package (`robot_msgs`) used to demonstrate
[roschema](https://github.com/justagist/roschema): breaking-change detection,
linting and formatting for `.msg`/`.srv`/`.action` files.

## Layout

```
robot_msgs/
  msg/JointState.msg
  msg/RobotStatus.msg
  srv/SetMode.srv
roschema.toml                    # check configuration
.pre-commit-config.yaml          # roschema's pre-commit hooks
.github/workflows/roschema.yml   # CI gate using the roschema Action
```

## Branches

| Branch | Change vs `main` | roschema verdict |
|---|---|---|
| `main` | baseline | - |
| `feature/safe-change` | adds a new `Diagnostic.msg`, tweaks a comment | **passes** and suggests a MINOR bump (a new interface breaks nothing; comments are ignored) |
| `feature/breaking-change` | removes a field, retypes another, reorders, adds a constant | **fails** (RS001 + RS003 + RS004) |

## Try it locally

```bash
uv tool install roschema    # or run it without installing: uvx roschema ...

# compare a branch against main
git switch feature/safe-change
roschema check --against main          # exit 0; names the new interface, suggests MINOR

git switch feature/breaking-change
roschema check --against main          # exit 1, reports the breaks
roschema check --against main --format json   # same, machine-readable
roschema check --against main --verbose        # also lists what was not reported, and why
roschema explain RS004                 # what a rule reports, and why it matters

# compare against an upstream you have not merged yet
roschema check --against origin/main

# only this branch's changes, even after main has moved on (merge base)
roschema check --against main...

# other capabilities
roschema lint                          # naming/style rules
roschema fmt --check                   # formatting gate
roschema freeze -o roschema.lock       # pin a baseline, then: check --against roschema.lock
```

Expected output on `feature/breaking-change`:

```
checking workspace:/path/to/roschema-demo  against  git:main
compared 3 interface(s) in 1 package(s)

robot_msgs
  RS001 error  robot_msgs/msg/RobotStatus.message: field 'message' removed  (wire-breaking)
      robot_msgs/msg/RobotStatus.msg:7
  RS003 error  robot_msgs/msg/RobotStatus.battery_percent: type changed float32 -> float64  (wire-breaking)
      robot_msgs/msg/RobotStatus.msg:8 -> robot_msgs/msg/RobotStatus.msg:7
  RS004 error  robot_msgs/msg/RobotStatus: fields reordered: mode, battery_percent -> battery_percent, mode  (wire-breaking)

3 finding(s): 3 error(s), 0 warning(s)
  robot_msgs: suggests MAJOR version bump (3 wire/source-breaking change(s))
```

## pre-commit

`.pre-commit-config.yaml` enables roschema's hooks. After `pre-commit install`
(with [pre-commit](https://pre-commit.com/) installed, e.g.
`uv tool install pre-commit`), every commit is checked against the last one,
linted and formatted, so a breaking change like `feature/breaking-change`'s is
stopped before it is committed. `pre-commit run --all-files` runs the hooks
without committing.

## CI

`.github/workflows/roschema.yml` runs the roschema GitHub Action on every push
and PR, comparing the branch against `origin/main`. Open a PR from each feature
branch to `main` to see the check pass on one and fail on the other: findings
appear as annotations on the diff, and the run's job summary lists every finding
with the suggested version bump. No local setup is needed - the action
provisions roschema on the runner.

The workflow uses `justagist/roschema/action@main`, which installs the latest
roschema release from PyPI on every run. To pin a release instead, set the
action's `version` input (for example `version: roschema==0.6.0`).
