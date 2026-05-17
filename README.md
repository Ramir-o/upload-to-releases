# upload-to-releases

A GitHub Action that uploads files to GitHub Releases using the GitHub REST API.

[English](README.md) | [简体中文](README.cn.md)

## Usage

Reference this Action in a `.github/workflows/*.yml` workflow file, for example [upload.yml](https://github.com/ophub/amlogic-s9xxx-armbian/blob/main/.github/workflows/build-armbian-arm64-server-image.yml):

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Your pre-build steps here
        run: echo "Building..."

      - name: Upload files to Release
        id: upload_step
        uses: ophub/upload-to-releases@main
        with:
          tag: "Set your release tags name"
          artifacts: "<path>/*.txt"
          gh_token: ${{ secrets.GITHUB_TOKEN }}
          body: |
            ### Describe your release notes
            - More description...

      - name: Print release URL (optional)
        run: |
          echo "Release ID: ${{ steps.upload_step.outputs.release_id }}"
          echo "Release URL: ${{ steps.upload_step.outputs.html_url }}"
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `tag` | **Required** | — | Tag name of the release to create or update (e.g. `v1.0.0`). |
| `artifacts` | **Required** | — | File path(s) to upload. Supports glob patterns and comma-separated values (e.g. `dist/*.zip` or `dist/*.zip,out/*.tar.gz`). |
| `gh_token` | **Required** | — | [GITHUB_TOKEN](https://docs.github.com/en/actions/security-guides/automatic-token-authentication) used to authenticate API requests. Requires `contents: write` permission. |
| `repo` | Optional | Current repository | Target repository in `<owner>/<repo>` format. Defaults to the repository running the workflow. |
| `allow_updates` | Optional | `true` | Update the release metadata (name, body, flags) if a release for the given tag already exists. Set to `false` to skip metadata updates on existing releases. |
| `remove_artifacts` | Optional | `false` | Remove **all** existing assets from the release before uploading new ones. Takes priority over `replaces_artifacts`. |
| `replaces_artifacts` | Optional | `true` | Replace an existing asset that has the same filename. When `false`, uploading a duplicate filename will be skipped. |
| `upload_timeout` | Optional | `10` | Per-file upload timeout in **minutes**. If a single file upload exceeds this limit it is abandoned and the next file is attempted immediately. Set to `0` to disable the per-file max-time limit. Note: even when `upload_timeout=0`, the stall guard remains active — uploads that transfer less than 1 KB/s for 60 consecutive seconds are still automatically abandoned. |
| `make_latest` | Optional | `true` | Mark this release as the latest release. Options: `true` / `false` / `legacy` (determined by date and semantic version). |
| `prerelease` | Optional | `false` | Mark this release as a pre-release. |
| `draft` | Optional | `false` | Mark this release as a draft. |
| `name` | Optional | `""` | Display title name of the release. Falls back to the tag name when omitted. |
| `body` | Optional | `""` | Markdown body text of the release. Overridden by `body_file` when both are set. |
| `body_file` | Optional | `""` | Path to a Markdown file whose content is used as the release body. Takes precedence over `body`. |
| `out_log` | Optional | `false` | Output detailed JSON logs for each step. Useful for debugging. |

## Outputs(optional)

| Output | Description |
|--------|-------------|
| `release_id` | Numeric ID of the created or updated release. |
| `html_url` | HTML URL of the release page (e.g. `https://github.com/owner/repo/releases/tag/v1.0.0`). |
| `upload_url` | Asset upload URL for the release (useful for custom upload steps). |
| `assets` | JSON object mapping each uploaded filename to its download URL (e.g. `{"file.zip":"https://...","image.img.gz":"https://..."}`). |

## Notes

- If the specified tag does not yet exist in the repository, GitHub will automatically create it pointing to the default branch at the time of the release creation.
- `remove_artifacts: true` deletes **all** existing assets before uploading; use with care.
- When `replaces_artifacts` is `true` and a file with the same name already exists, the old asset is deleted first and then re-uploaded.
- `body_file` takes precedence over `body` when both are provided.
- If a single file upload is stuck (speed below 1 KB/s for 60 s, or the per-file timeout is reached), the upload is automatically abandoned and the script moves on to the next file in the queue.
- Setting `upload_timeout=0` disables only the per-file max-time limit. The stall guard (abort when speed < 1 KB/s for 60 s) stays active regardless.
- After all uploads complete, the action automatically verifies each file using the API-provided hash (`digest: sha256:<hex>`).

## Upload progress and logging

This action prints detailed real-time progress for every file:

```text
[ STEPS ] Expanding artifact patterns...
[ INFO  ] Total files to upload: [ 5 ]
[ INFO  ] ────────────────────────────────────────────────────────────────────────
[ INFO  ]    1/5     1.23 GiB   firmware-arm64.img.gz
[ INFO  ]    2/5    45.67 MiB   firmware-x86.img.gz
[ INFO  ]    3/5     2.10 MiB   checksums.sha256
[ INFO  ]    4/5    12.34 KiB   release-notes.md
[ INFO  ]    5/5      890 B     version.txt
[ INFO  ] ────────────────────────────────────────────────────────────────────────
[ INFO  ] Total: [ 5 ] files,  1.28 GiB
[ INFO  ] ────────────────────────────────────────────────────────────────────────

[ STEPS ] Starting upload of [ 5 ] file(s) to release [ 123456 ]...
[ FILE ] ┌─ (1/5) Uploading: [ firmware-arm64.img.gz ]
[ SIZE ] │  (1/5) Size: 1.23 GiB  MIME: application/gzip  timeout=10min
[ DONE ] │  (1/5) Upload completed in 87s: [ firmware-arm64.img.gz ]
[ DONE ] └─ (1/5) Download URL: [ https://github.com/owner/repo/releases/download/v1.0.0/firmware-arm64.img.gz ]

...
[ SUCCESS ] Upload summary: [ 5 ] total, [ 5 ] succeeded, [ 0 ] failed, [ 0 ] skipped.

[ STEPS ] Verifying upload integrity (SHA-256)...
[ INFO  ] ────────────────────────────────────────────────────────────────────────
[ SUCCESS ] OK   1/5 [ firmware-arm64.img.gz ]
[ SUCCESS ] OK   2/5 [ firmware-x86.img.gz ]
[ SUCCESS ] OK   3/5 [ checksums.sha256 ]
[ SUCCESS ] OK   4/5 [ release-notes.md ]
[ SUCCESS ] OK   5/5 [ version.txt ]
[ INFO  ] ────────────────────────────────────────────────────────────────────────
[ SUCCESS ] Integrity summary: [ 5 ] total, [ 5 ] passed, [ 0 ] failed, [ 0 ] skipped.
```

## Links

- [GitHub REST API – Releases](https://docs.github.com/en/rest/releases/releases)
- [GitHub REST API – Release Assets](https://docs.github.com/en/rest/releases/assets)
- [delete-releases-workflows](https://github.com/ophub/delete-releases-workflows)
- [amlogic-s9xxx-armbian](https://github.com/ophub/amlogic-s9xxx-armbian)
- [amlogic-s9xxx-openwrt](https://github.com/ophub/amlogic-s9xxx-openwrt)
- [luci-app-amlogic](https://github.com/ophub/luci-app-amlogic)
- [fnnas](https://github.com/ophub/fnnas)
- [kernel](https://github.com/ophub/kernel)
- [u-boot](https://github.com/ophub/u-boot)
- [firmware](https://github.com/ophub/firmware)

## License

upload-to-releases © OPHUB is licensed under [GPL-2.0](https://github.com/ophub/upload-to-releases/blob/main/LICENSE).
