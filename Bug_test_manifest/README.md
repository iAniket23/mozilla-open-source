# bug 1816767
Switch `test-manifest-alpha` linter to use an exclude list of manifests

# description
The `test-manifest-alpha` linter has a yaml file of manifests, where if they fail, the issue will become an error rather than a warning. This allows us to do a gradual migration of manifests rather than needing to fix them all at once. However, it would be slightly better if instead of listing the manifests that were good, it listed the manifests that were bad. This way, new manifests will automatically get linted. To solve this bug, I'd recommend: 1. Adjust the logic in the linter so that manifests listed in the `tools/lint/test-manifest-alpha/error-level-manifests.yml` turn issues into a warning (and the default is error). Also rename the file to `warning-level-manifests.yml`. 2. Run `./mach lint -l test-manifest-alpha -f unix` and use scripts or coreutils to parse out the list of manifests that have failures. Then add them to the `warning-level-manifests.yml` file.
