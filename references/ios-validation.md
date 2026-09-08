# iOS simulator validation

Read this only for native iOS testing with Maestro or CoreSimulator. Apply the
[shared validation procedure](validation.md) for source identity, process
cleanup, evidence, and resource ownership.

## Device ownership and build identity

Hold both `heavy` and `simulator-<device-id>` for the selected device. The same
resource pair applies to command jobs and interactive `--hold` sessions. Use
absolute skill paths in actual commands:

```bash
scripts/resource-run --resource heavy --resource simulator-<device-id> -- \
  scripts/validate-run --job /absolute/path/ios-job.json --output /absolute/path/new-evidence-dir
```

One QA owner drives the simulator. Other lanes submit affected flows and read
screenshots and accessibility trees. Prove one representative fixture flow before
expanding the suite. Prefer the project's stable fixture Release build over
repeated development-client prompt dismissal.

Declare all build configuration in the job's `environment` or
`inherit_environment`. For an Expo fixture build, that can include the project's
`EXPO_PUBLIC_*` switches and service origins. A changed bundled value requires a
matching build. `verify` must check the installed artifact's source and
configuration. The bundle identifier alone is insufficient.

Use repository-specific commands for build and installation verification.
Typical smoke and affected commands are `maestro test <smoke-flow.yaml>` and
`maestro test <affected-flow.yaml>`. Run the full suite after affected flows pass
on the fixed combined source.

## Crash evidence and recovery

`health` must check the target simulator and detect new simulator crash reports
since `VALIDATE_RUN_STARTED_AT`, including SpringBoard crashes that have already
restarted. A failed UI assertion alone is not evidence of an infrastructure crash.
`collect` preserves Maestro artifacts and matching device crash reports under
`VALIDATE_RUN_OUTPUT` before recovery. `recover` may restart only the leased
simulator. It must not erase devices or terminate other runs.

The shared helper attempts recovery once, verifies the installed artifact, and
re-enters through the smoke flow. Review a `passed-after-recovery` result before
accepting it. An app-triggered SpringBoard crash remains a defect even if a retry
passes. These checks require application-specific adapters; they are not
implemented by the generic helper itself.
