# scoop-bucket

A personal [Scoop](https://scoop.sh) bucket for apps that aren't in `main`/`extras` —
things installed from loose JSON files, GitHub repos with no releases, or local
one-offs. Having them here makes a machine reproducible instead of depending on
whatever happens to be sitting in `~\Downloads`.

## Install

```powershell
scoop bucket add halr9000 https://github.com/halr9000/scoop-bucket
scoop install halr9000/sa3_tflite
```

## Apps

| App | Description |
|-----|-------------|
| [`sa3_tflite`](bucket/sa3_tflite.json) | [Stable Audio 3](https://github.com/Stability-AI/stable-audio-3) CPU inference via LiteRT/TFLite — text-to-audio, audio-to-audio, inpainting. No PyTorch at runtime. |

`sa3_tflite` puts upstream's own wrappers on `PATH` as `sa3` and `sa3-gradio`,
so they work from any directory:

```powershell
sa3 --prompt "lofi house loop" --dit sm-music --decoder same-s --play
sa3 --help
sa3-gradio
```

Model weights are not bundled; they lazy-download from HuggingFace on first use
(~2.3 GB small, ~9.5 GB medium) and persist across upgrades.

## Rebuilding a machine

The bucket covers *which* apps exist; `scoop export` covers *what you had installed*:

```powershell
scoop export > scoop-apps.json
```

Commit that file somewhere, then on a fresh machine:

```powershell
scoop import scoop-apps.json
```

`scoop import` restores buckets first, then apps — so anything from this bucket
comes back automatically, as long as the bucket is in the export.

## Adding an app

1. Drop a manifest in `bucket/<app>.json`.
2. Test it locally before committing — you can install straight from the file:

   ```powershell
   scoop install .\bucket\<app>.json
   ```

3. Check that `checkver` resolves. The `bin\` wrappers already point at `bucket\`,
   so don't pass `-Dir`:

   ```powershell
   .\bin\checkver.ps1 -App <app>
   ```

4. Let scoop compute the hash and rewrite the manifest for the newest version:

   ```powershell
   .\bin\checkver.ps1 -App <app> -Update
   ```

5. Format consistently:

   ```powershell
   .\bin\formatjson.ps1
   ```

6. Run the test suite. It needs two modules the CI installs for you, so install
   them once locally:

   ```powershell
   Install-Module BuildHelpers, Pester -Scope CurrentUser -Force
   ```

   ```powershell
   .\bin\test.ps1
   ```

> **The first CI run on a new bucket logs a scary error that you can ignore.**
> The changed-manifest linter diffs `HEAD^..HEAD`, which doesn't resolve when
> the repo has only one commit, so discovery reports
> `fatal: ambiguous argument 'HEAD^..HEAD'`. The job still passes — `test.ps1`
> exits on Pester's failed-test count, and a discovery error doesn't add to it.
> It stops appearing from the second commit onward.

### Notes from building `sa3_tflite`

Worth knowing if you package something similar:

- **No upstream release is needed.** Scoop installs from any URL. For a repo with
  no tags, pin a commit archive —
  `https://github.com/<owner>/<repo>/archive/<full-sha>.zip` — which is immutable
  and therefore hashable. Use `extract_dir` to pull out a subdirectory.
- **Phase order matters.** It runs
  `installer` → `create_shims` → `persist_data` → `post_install`.
  Generate any file that `bin` shims in `pre_install`/`installer`; do work that
  should reuse persisted state in `post_install`.
- **Version scheme for commit-pinned apps.** `<date>-<shortsha>` alone is unsafe:
  two commits on the same day fall back to comparing SHAs lexically, which is
  meaningless ordering. Include the time — `2026.08.02.0800-c7e8800`.
- **Don't depend on system Python.** `scoop install python` tracks the newest
  release, which routinely outpaces wheel availability for native packages, and
  `python` on `PATH` may belong to some other app's virtualenv. Depend on
  `main/uv` and let it provision a pinned interpreter instead.

## License

[Unlicense](LICENSE) — applies to this bucket's manifests and scripts, not to the
apps they install.
