# Releasing et-upcloud

Every deployment to main must go through a versioned release with a GitHub release and tag.

## Release Checklist

### 1. Bump version in all files

Update the version in **all four locations** (they must match):

- `VERSION`
- `.claude-plugin/marketplace.json` → `metadata.version`
- `et-upcloud-plugin/.claude-plugin/plugin.json` → `version`
- `README.md` → version pin example

### 2. Update CHANGELOG.md

Add a new section at the top:

```markdown
## vX.Y.Z (YYYY-MM-DD)

- What changed (one bullet per change)
```

### 3. Validate plugin

```bash
claude plugin validate .
claude plugin validate ./et-upcloud-plugin
```

Both must pass.

### 4. Commit, push, and create PR

```bash
git checkout -b release/vX.Y.Z
git add VERSION .claude-plugin/marketplace.json et-upcloud-plugin/.claude-plugin/plugin.json README.md CHANGELOG.md
git commit -m "Release vX.Y.Z — <summary>"
git push -u origin release/vX.Y.Z
gh pr create --title "Release vX.Y.Z" --body "See CHANGELOG.md for details."
```

### 5. Merge PR

Merge via GitHub (main is protected — no direct pushes).

### 6. Create GitHub release with tag

After merge, create a release from main:

```bash
git checkout main && git pull
gh release create vX.Y.Z --title "vX.Y.Z" --notes-file CHANGELOG.md
```

This creates the `vX.Y.Z` git tag and a GitHub release. Users can then pin to this version:

```
/plugin marketplace add Emerging-Tech-Visma/et-upcloud@vX.Y.Z
```

## Versioning Rules

- **Patch** (1.0.x): Bug fixes, doc updates, reference corrections
- **Minor** (1.x.0): New commands, new skill features, new templates
- **Major** (x.0.0): Breaking changes to .deploy.json schema, skill interface, or permission model
