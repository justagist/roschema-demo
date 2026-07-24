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
.github/workflows/roschema.yml   # CI gate using the roschema Action
```

## Branches

| Branch | Change vs `main` | roschema verdict |
|---|---|---|
| `main` | baseline | - |
| `feature/safe-change` | adds a new `Diagnostic.msg`, tweaks a comment | **passes** (adding an interface is not breaking; comments are ignored) |
| `feature/breaking-change` | removes a field, retypes another, reorders | **fails** (RS001 + RS003 + RS004) |

## Try it locally

```bash
pip install roschema        # or: uvx roschema ...

# compare a branch against main
git switch feature/safe-change
roschema check --against main          # exit 0, clean

git switch feature/breaking-change
roschema check --against main          # exit 1, reports the breaks
roschema check --against main --format json   # same, machine-readable

# compare against an upstream you have not merged yet
roschema check --against origin/main

# other capabilities
roschema lint                          # naming/style rules
roschema fmt --check                   # formatting gate
roschema freeze -o roschema.lock       # pin a baseline, then: check --against roschema.lock
```

Expected output on `feature/breaking-change`:

```
RS001 error  robot_msgs/msg/RobotStatus.message: field 'message' removed  (wire-breaking)
RS003 error  robot_msgs/msg/RobotStatus.battery_percent: type changed float32 -> float64  (wire-breaking)
RS004 error  robot_msgs/msg/RobotStatus: fields reordered ...  (wire-breaking)
  robot_msgs: suggests MAJOR version bump
```

## CI

`.github/workflows/roschema.yml` runs the roschema GitHub Action on every push
and PR, comparing the branch against `origin/main`. Open a PR from each feature
branch to `main` to see the check pass on one and fail on the other. No local
setup is needed - the action provisions roschema on the runner.

> This demo pins the action to `@main` and installs roschema from git because it
> predates the first PyPI release. Once roschema is published, use
> `justagist/roschema/action@v1` and drop the `version:` input.
