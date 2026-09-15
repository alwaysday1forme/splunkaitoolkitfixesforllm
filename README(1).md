# Splunk AI Toolkit Optimization Deployment Package

This package contains the exact source-level optimization set reconstructed from the tested Splunk environment and the changes applied during troubleshooting.

## What this package is for

Use it to reproduce the same optimized `| ai ... connection="foundationsec"` behavior in another **compatible** Splunk Machine Learning Toolkit environment.

The tested custom connection uses an OpenAI-compatible Ollama endpoint. The direct custom-provider path assumes `/v1/chat/completions` compatibility.

## Files installed

```text
Splunk_ML_Toolkit/bin/ai_commander/llm_base.py
Splunk_ML_Toolkit/bin/ai_commander/llm_factory.py
Splunk_ML_Toolkit/bin/processors/AiCommanderProcessor.py
Splunk_ML_Toolkit/bin/connection_config_manager/llm/llm_config_manager.py
Splunk_ML_Toolkit/bin/connection_config_manager/utils/common_utils.py
Splunk_ML_Toolkit/bin/chunked_controller.py
```

`chunked_controller.py` contains diagnostic timing instrumentation. The other five files contain the functional optimization changes.

`ai.py` was touched temporarily during troubleshooting but was restored, so it is not part of the replacement payload.

## Before installing

1. Confirm the destination uses the same or a compatible Splunk_ML_Toolkit version.
2. Test on a clone/non-production Splunk instance first.
3. Keep a rollback copy of the existing files.
4. Review `REPORT.md` and `CHANGES.patch`.

## Automated install

From the extracted package directory, run as an account that can modify the app files:

```bash
chmod +x install.sh
sudo ./install.sh
```

Default app root:

```text
/opt/splunk/etc/apps/Splunk_ML_Toolkit
```

To use a different root:

```bash
sudo APP_ROOT=/custom/path/Splunk_ML_Toolkit ./install.sh
```

The installer:

- verifies the expected target files exist;
- creates a timestamped backup directory;
- preserves each target file's owner, group, and mode;
- copies the optimized source files into place;
- runs `py_compile` when the expected Splunk Scientific Python interpreter exists.

## Manual install

Copy the contents below `deploy/Splunk_ML_Toolkit/` over the corresponding files in:

```text
/opt/splunk/etc/apps/Splunk_ML_Toolkit/
```

Preserve the original file ownership and permissions.

## Important behavior changes

- Expensive AI runtime setup is deferred until `process()` actually executes.
- Explicit named connections prefer the new connection store and only fall back to legacy config.
- The default-connection lookup is skipped for an explicit connection.
- Wildcard ACLs are evaluated before loading user roles.
- Runtime MLSPL retry settings are fixed to `max_retries=3`, `backoff_factor=2`.
- Custom (`is_custom=True`) connections use a direct OpenAI-compatible HTTPX path and bypass LiteLLM.
- Non-custom providers retain the existing LiteLLM/provider-specific routes.

### Compatibility warning for custom providers

The custom path was tested with Foundation-Sec on Ollama. If another custom provider is **not** OpenAI `/chat/completions` compatible, do not deploy `llm_base.py` / `llm_factory.py` unchanged without adapting the routing logic.

## Validation

Run:

```spl
| makeresults
| eval prompt="Reply with exactly one word: OK"
| fields prompt
| ai prompt="{prompt}" connection="foundationsec"
```

Then inspect the latest search log:

```bash
LATEST=$(ls -td /opt/splunk/var/run/splunk/dispatch/* | head -1)

grep "TIMING CUSTOM HTTPX\|TIMING LITELLM\|TIMING RUNTIME" \
  "$LATEST/search.log"
```

A successful optimized Foundation-Sec run should show `TIMING CUSTOM HTTPX completion` and should **not** show `TIMING LITELLM import`.

## Rollback

`install.sh` prints the backup directory it creates. Copy the backed-up files back to their original locations. If necessary, restart Splunk during a maintenance window after rollback.

## Package references

- `REPORT.md` - full investigation and rationale.
- `REPORT.docx` - formatted report.
- `CHANGES.patch` - diff versus the source snapshot used for packaging.
- `checksums.sha256` - integrity hashes.
- `reference/original/` - original source snapshot files.
