# Fetch latest Deno release version
> format `vX.Y.Z`

## GitHub
> Works on: macOS, Linux, and Windows.
```sh
gh release view --repo denoland/deno --json tagName --jq '.tagName'
# fallback raw api if gh is not available
curl -fsSL https://api.github.com/repos/denoland/deno/releases/latest | jq -r '.tag_name'
```

## Deno hosted
> Works on: macOS, Linux, and Windows.
> Requires: curl.
```sh
curl -fsSL https://dl.deno.land/release-latest.txt
```

### PowerShell
```powershell
(Invoke-WebRequest -Uri https://dl.deno.land/release-latest.txt -UseBasicParsing).Content
Invoke-RestMethod -Uri https://dl.deno.land/release-latest.txt
```
