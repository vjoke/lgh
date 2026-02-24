---
description: Automated migration tool for Go projects moving from gitlab.meitu.com to git.meitu.com
arguments: true
---

# GitLab to Git Migration Tool

Migrate Go projects from `gitlab.meitu.com` to `git.meitu.com`.

{{if eq (len $ARGUMENTS) 0}}

## Quick Start

Run the full migration:

```bash
# 1. Get current module
MODULE=$(grep "^module" go.mod | awk '{print $2}')
OLD_MODULE="$MODULE"
NEW_MODULE=$(echo "$MODULE" | sed 's|gitlab.meitu.com|git.meitu.com|')

echo "Migrating: $OLD_MODULE -> $NEW_MODULE"

# 2. Update go.mod module path
sed -i '' "s|^module gitlab.meitu.com/|module git.meitu.com/|" go.mod

# 3. Add replace directives for gitlab.meitu.com dependencies
echo "" >> go.mod
echo "replace (" >> go.mod
grep -E "^\s+gitlab\.meitu\.com" go.mod | sed -E 's/^[[:space:]]+(gitlab\.meitu\.com\/[^[:space:]]+)[[:space:]]+(v[0-9]+\.[0-9]+\.[0-9]+[^[:space:]]*).*/    \1 => git.meitu.com\/\1 \2/' >> go.mod
echo ")" >> go.mod

# 4. Remove go.sum
rm -f go.sum

# 5. Update .gitlab-ci.yml
sed -i '' 's|gitlab.meitu.com/|git.meitu.com/|g' .gitlab-ci.yml
sed -i '' 's|GOPRIVATE=.*|GOPRIVATE=git.meitu.com,gitlab.meitu.com|g' .gitlab-ci.yml

# 6. Update current module imports only
find . -name "*.go" -type f ! -path "./vendor/*" -exec \
    sed -i '' "s|\"$OLD_MODULE/|\"$NEW_MODULE/|g" {} \;

# 7. Verify
go mod tidy
go build ./...
```

## Step-by-Step Migration

Or run individual steps:

### Step 1: Analyze

```bash
grep "^module" go.mod
grep "gitlab.meitu.com" go.mod | grep -v "^module"
```

### Step 2: Update go.mod

```bash
sed -i '' 's|gitlab.meitu.com/|git.meitu.com/|g' go.mod

# Add replace directives
echo "" >> go.mod
echo "replace (" >> go.mod
grep -E "^\s+gitlab\.meitu\.com" go.mod | \
  sed -E 's/^[[:space:]]+(gitlab\.meitu\.com\/[^[:space:]]+)[[:space:]]+(v[0-9]+\.[0-9]+\.[0-9]+[^[:space:]]*).*/    \1 => git.meitu.com\/\1 \2/' >> go.mod
echo ")" >> go.mod
```

### Step 3: Remove go.sum

```bash
rm -f go.sum
```

### Step 4: Update CI

```bash
sed -i '' 's|gitlab.meitu.com/|git.meitu.com/|g' .gitlab-ci.yml
sed -i '' 's|GOPRIVATE=.*|GOPRIVATE=git.meitu.com,gitlab.meitu.com|g' .gitlab-ci.yml
```

### Step 5: Update Imports

**Important**: Only update imports for the current module, not external dependencies.

```bash
MODULE=$(grep "^module" go.mod | awk '{print $2}')
OLD_MODULE=$(echo "$MODULE" | sed 's|git.meitu.com|gitlab.meitu.com|')

find . -name "*.go" -type f ! -path "./vendor/*" -exec \
    sed -i '' "s|\"$OLD_MODULE/|\"$MODULE/|g" {} \;
```

### Step 6: Verify

```bash
go mod tidy
go build ./...
```

{{else if eq (index $ARGUMENTS 0) "analyze"}}

## Analyze Current Project

```bash
# Current module
echo "=== Current Module ==="
grep "^module" go.mod

# gitlab.meitu.com dependencies
echo ""
echo "=== Dependencies to Replace ==="
grep "gitlab.meitu.com" go.mod | grep -v "^module"

# Files with current module imports
echo ""
echo "=== Files with Current Module Imports ==="
MODULE=$(grep "^module" go.mod | awk '{print $2}')
grep -r "$MODULE" --include="*.go" . | grep -v "vendor/" | head -20
```

{{else if eq (index $ARGUMENTS 0) "gomod"}}

## Update go.mod

```bash
# Update module declaration
sed -i '' 's|gitlab.meitu.com/|git.meitu.com/|g' go.mod

# Add replace directives
echo "" >> go.mod
echo "replace (" >> go.mod
grep -E "^\s+gitlab\.meitu\.com" go.mod | \
  sed -E 's/^[[:space:]]+(gitlab\.meitu\.com\/[^[:space:]]+)[[:space:]]+(v[0-9]+\.[0-9]+\.[0-9]+[^[:space:]]*).*/    \1 => git.meitu.com\/\1 \2/' >> go.mod
echo ")" >> go.mod

echo "go.mod updated. Preview:"
head -30 go.mod
```

{{else if eq (index $ARGUMENTS 0) "imports"}}

## Update Import Paths

Update only current module imports (external dependencies use replace directives):

```bash
MODULE=$(grep "^module" go.mod | awk '{print $2}')
OLD_MODULE=$(echo "$MODULE" | sed 's|git.meitu.com|gitlab.meitu.com|')

echo "Updating imports: $OLD_MODULE -> $MODULE"

find . -name "*.go" -type f ! -path "./vendor/*" -exec \
    sed -i '' "s|\"$OLD_MODULE/|\"$MODULE/|g" {} \;

echo "Import paths updated."
```

{{else if eq (index $ARGUMENTS 0) "ci"}}

## Update .gitlab-ci.yml

```bash
# Update paths
sed -i '' 's|gitlab.meitu.com/|git.meitu.com/|g' .gitlab-ci.yml

# Update GOPRIVATE
sed -i '' 's|GOPRIVATE=.*|GOPRIVATE=git.meitu.com,gitlab.meitu.com|g' .gitlab-ci.yml

echo ".gitlab-ci.yml updated. Preview:"
head -30 .gitlab-ci.yml
```

{{else if eq (index $ARGUMENTS 0) "verify"}}

## Verify Migration

```bash
# Remove old go.sum
rm -f go.sum

# Download dependencies and regenerate go.sum
echo "Running go mod tidy..."
go mod tidy

# Verify build
echo ""
echo "Building project..."
go build ./...

# Run tests
echo ""
echo "Running tests..."
go test ./...

echo ""
echo "Migration verified successfully!"
```

{{end}}

## Examples

### Example go.mod Before/After

**Before:**
```go
module gitlab.meitu.com/platform/goboot

go 1.21

require (
    gitlab.meitu.com/pl_container/mlib v1.32.26
    gitlab.meitu.com/gocommons/cores v0.0.5
)
```

**After:**
```go
module git.meitu.com/platform/goboot

go 1.21

require (
    gitlab.meitu.com/pl_container/mlib v1.32.26
    gitlab.meitu.com/gocommons/cores v0.0.5
)

replace (
    gitlab.meitu.com/pl_container/mlib => git.meitu.com/pl_container/mlib v1.32.26
    gitlab.meitu.com/gocommons/cores => git.meitu.com/gocommons/cores v0.0.5
)
```

### Example Import Changes

**Before:**
```go
import (
    "gitlab.meitu.com/platform/goboot/config"      // Current module
    "gitlab.meitu.com/platform/goboot/utils"       // Current module
    "gitlab.meitu.com/pl_container/mlib/client"    // External dependency
)
```

**After:**
```go
import (
    "git.meitu.com/platform/goboot/config"         // Current module - updated
    "git.meitu.com/platform/goboot/utils"          // Current module - updated
    "gitlab.meitu.com/pl_container/mlib/client"    // External - unchanged (via replace)
)
```

## Usage

```
/lgh:migrate-gitlab              # Show full migration guide
/lgh:migrate-gitlab analyze      # Analyze current project
/lgh:migrate-gitlab gomod        # Update go.mod only
/lgh:migrate-gitlab imports      # Update import paths only
/lgh:migrate-gitlab ci           # Update .gitlab-ci.yml only
/lgh:migrate-gitlab verify       # Run go mod tidy and build
```

## Key Points

1. **Current module imports** are updated to `git.meitu.com/...`
2. **External dependencies** keep `gitlab.meitu.com/...` in source, resolved via `replace` directives
3. **go.sum is regenerated** - don't preserve the old one
4. **Both domains in GOPRIVATE** - ensures access during transition
