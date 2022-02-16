# falco-sync

## Description
sample description

## Usage

### Fetch the package
`kpt pkg get REPO_URI[.git]/PKG_PATH[@VERSION] falco-sync`
Details: https://kpt.dev/reference/cli/pkg/get/

### View package content
`kpt pkg tree falco-sync`
Details: https://kpt.dev/reference/cli/pkg/tree/

### Apply the package
```
kpt live init falco-sync
kpt live apply falco-sync --reconcile-timeout=2m --output=table
```
Details: https://kpt.dev/reference/cli/live/
