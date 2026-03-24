# Skill: MATLAB Package Manager

Use this skill when the user needs help installing, managing, packaging, or distributing MATLAB toolboxes and packages.

---

## Context: The Four Tools

There are four separate package management tools in the MATLAB ecosystem. Always identify which one applies before giving advice:

| Tool | What it does | When to use |
|------|-------------|-------------|
| **OS-level mpm** | Installs MATLAB itself + MathWorks toolboxes | Docker, CI/CD, server provisioning |
| **In-MATLAB mpm** (R2024b+) | Manages community packages via `mpackage.json` | New projects on R2024b+ wanting modern workflows |
| **Add-On Explorer / matlab.addons** | GUI and programmatic `.mltbx` installs | Interactive or scripted community toolbox installs |
| **tbxmanager** | Community registry (YALMIP, MPT3, SeDuMi, etc.) | Control systems, optimization, academic work |

---

## OS-Level mpm — Key Commands

```bash
# Download (Linux)
wget -q https://www.mathworks.com/mpm/glnxa64/mpm && chmod +x mpm

# Install MATLAB + toolboxes
./mpm install \
  --release=R2025b \
  --destination=/opt/matlab/R2025b \
  --products=MATLAB Signal_Processing_Toolbox Statistics_and_Machine_Learning_Toolbox

# Pin to specific update level
./mpm install --release=R2024bU3 --destination=/opt/matlab --products=MATLAB

# Download for offline use
./mpm download --release=R2025b --destination=/cache --products=MATLAB

# Install from local cache (offline)
./mpm install --source=/cache --destination=/opt/matlab --products=MATLAB
```

**Common product name tokens:** `MATLAB`, `Simulink`, `Deep_Learning_Toolbox`, `Statistics_and_Machine_Learning_Toolbox`, `Parallel_Computing_Toolbox`, `Signal_Processing_Toolbox`, `Control_System_Toolbox`, `Optimization_Toolbox`, `Image_Processing_Toolbox`, `Computer_Vision_Toolbox`

**GitHub Actions:**
```yaml
- uses: matlab-actions/setup-matlab@v2
  with:
    release: R2025b
    products: Simulink Deep_Learning_Toolbox
- uses: matlab-actions/run-tests@v2
```

**Docker pattern:**
```dockerfile
RUN wget -q https://www.mathworks.com/mpm/glnxa64/mpm && chmod +x mpm && \
    ./mpm install --release=R2025b --destination=/opt/matlab \
      --products=MATLAB Signal_Processing_Toolbox && rm -f mpm
ENV MLM_LICENSE_TOKEN=<token>
ENTRYPOINT ["matlab", "-batch"]
```

---

## In-MATLAB mpm (R2024b+) — Key Commands

```matlab
% Search
mpmsearch("signal processing")

% Install
mpminstall("package-name")
mpminstall("package-name", Version="1.2.3")          % specific version
mpminstall("/path/to/local/package")                  % from local folder
mpminstall("/path/to/package", Editable=true)         % editable/dev mode

% List / remove
mpmlist
mpmuninstall("package-name")

% Registry management
mpmListRepositories
mpmAddRepository("https://registry.example.com/matlab")
mpmRemoveRepository("repo-name")

% Create a new package
pkg = mpmcreate("myPackage", "/path/to/code", ...
    Version="1.0.0", ...
    Description="My utilities")
```

**Minimal mpackage.json** (must be at `resources/mpackage.json` in package root):
```json
{
  "schemaVersion": "1.1.0",
  "name": "my-package",
  "version": "1.0.0",
  "description": "My MATLAB utilities",
  "matlabVersion": ">=24.2.0"
}
```

**Version mapping:** R2024b = `24.2.0`, R2025a = `25.1.0`, R2025b = `25.2.0`

⚠️ **Known limitation:** Namespace folders (`+namespace`) are not supported as package roots.

---

## .mltbx Files — Programmatic Install

```matlab
% Install a local .mltbx
matlab.addons.install('/path/to/toolbox.mltbx')

% Install without dialog prompt
matlab.addons.toolbox.installToolbox('/path/to/toolbox.mltbx', true)

% List installed add-ons
matlab.addons.installedAddons()

% Uninstall by name
matlab.addons.uninstall('Toolbox Name')
```

---

## tbxmanager — Setup and Usage

```matlab
% Bootstrap (run once)
urlwrite('http://www.tbxmanager.com/tbxmanager.m', 'tbxmanager.m');
tbxmanager savepath

% Add to startup.m for persistence:
% tbxmanager restorepath

% Install
tbxmanager install yalmip
tbxmanager install mpt sedumi        % multiple at once

% Update
tbxmanager update                    % all packages
tbxmanager update yalmip             % specific

% Browse
tbxmanager show available
tbxmanager show installed
```

---

## Packaging a Toolbox

```matlab
% Programmatic packaging (CI-friendly)
opts = matlab.addons.toolbox.ToolboxOptions("/path/to/root", "guid-string");
opts.ToolboxName = "My Toolbox";
opts.ToolboxVersion = "1.0.0";
opts.OutputFile = "/output/MyToolbox.mltbx";
matlab.addons.toolbox.packageToolbox(opts)
```

**Recommended structure:**
```
my-toolbox/
├── toolbox/          ← source included in .mltbx
│   └── MyFunction.m
├── examples/
├── doc/
├── my-toolbox.prj    ← packaging project (contains GUID)
└── release/
    └── MyToolbox.mltbx   ← don't version control this
```

---

## Workaround: Vendor Dependencies in Git

When reproducibility matters and the package manager falls short:

```
project/
├── src/
├── vendor/           ← dependencies committed directly
│   ├── jsonlab/
│   └── export_fig/
└── setup.m
```

```matlab
% setup.m
addpath(genpath(fullfile(fileparts(mfilename('fullpath')), 'vendor')));
```

---

## Key Gotchas to Watch For

1. **No lockfile** — in-MATLAB mpm has no lockfile. Two installs at different times may get different versions.
2. **Licensing blocks CI** — headless MATLAB requires a paid Batch Licensing Token. Not a technical problem — an institutional one.
3. **Path pollution** — installed toolboxes go on the global MATLAB path. Name collisions between toolboxes cause silent incorrect behavior.
4. **Two mpm tools** — OS-level mpm (provisioning) and in-MATLAB mpm (packages) share a name but are completely different. Clarify which one the user means.
5. **GUID changes = duplicate installs** — if a `.prj` file's GUID changes, MATLAB treats it as a new toolbox instead of an upgrade.
6. **`+namespace` incompatibility** — the new in-MATLAB mpm doesn't support namespace folders as package roots.
