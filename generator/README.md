# Record/Replay Generator for cisco.ios Molecule Tests

## Links

- **Generator script**: https://github.com/rohitthakur2590/cisco.ios/blob/ios_mock_test_br/generate_molecule_from_recording.py
  (PR: https://github.com/ansible-collections/cisco.ios/pull/1316)
- **Patched ansible.netcommon** (recording support): https://github.com/rohitthakur2590/ansible.netcommon/tree/record_poc_br
  (PR: https://github.com/ansible-collections/ansible.netcommon/pull/772)

---

## How It Works

1. A patched `network_cli` connection plugin records every CLI command/response as JSONL
2. You run the module's integration test flow against a real device with recording enabled
3. The generator script reads the JSONL and produces Molecule test files (converge.yml, molecule.yml, vars, inventories, CISSHGO transcripts)

---

## Step-by-Step: ios_hostname

### Prerequisites

- Python 3.10+
- Access to a Cisco IOS/IOS-XE device (your 2084 router works)
- Inventory file at `ansible-dev/inventory.ini`

### Step 1: Set up the environment

Run all commands from `ansible-dev/generator/`:

```bash
cd /Users/piyushmalik/dev-workspace/ansible-dev/generator

# Create a venv (in generator/ or the parent ansible-dev/ — either works)
python3 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install ansible-core molecule paramiko ansible-pylibssh pyyaml

# Install the patched netcommon (with recording support) + utils
ansible-galaxy collection install \
  git+https://github.com/rohitthakur2590/ansible.netcommon.git,record_poc_br \
  git+https://github.com/ansible-collections/ansible.utils.git \
  -p ./collections --force

# Install cisco.ios from your local checkout (MUST use absolute path)
ansible-galaxy collection install \
  /Users/piyushmalik/dev-workspace/ansible-dev/ansible_collections/cisco/ios \
  -p ./collections --force
```

If you already have a venv in `ansible-dev/.venv`, just activate it:
```bash
source ../.venv/bin/activate
```

### Step 2: Take the recording

Run from `ansible-dev/generator/`:

```bash
# Set environment variables
export ANSIBLE_COLLECTIONS_PATH=$(pwd)/collections
export ANSIBLE_HOST_KEY_CHECKING=False
export ANSIBLE_NETWORK_CLI_RECORD=1
export ANSIBLE_NETWORK_CLI_RECORD_PATH=$(pwd)/recordings

# Create the recordings directory
mkdir -p recordings

# Run the recording playbook
ansible-playbook record_ios_hostname.yml -i ../inventory.ini
```

This produces a JSONL file in `./recordings/` named like `<host>_<port>.jsonl`
(e.g. `54.190.208.146_2084.jsonl`).

Rename it for clarity:
```bash
mv recordings/*_2084.jsonl recordings/ios_hostname.jsonl
```

### Step 3: Download the generator script

```bash
curl -o generate_molecule_from_recording.py \
  https://raw.githubusercontent.com/rohitthakur2590/cisco.ios/ios_mock_test_br/generate_molecule_from_recording.py
```

### Step 4: Run the generator

The generator needs to run from the **collection root** (so it can find `tests/integration/targets/ios_hostname/vars/main.yaml`):

```bash
# From generator/ directory:
cd ../ansible_collections/cisco/ios

python3 /Users/piyushmalik/dev-workspace/ansible-dev/generator/generate_molecule_from_recording.py \
  /Users/piyushmalik/dev-workspace/ansible-dev/generator/recordings/ios_hostname.jsonl \
  --module ios_hostname \
  --show-command "show running-config | section ^hostname" \
  --molecule-dir /Users/piyushmalik/dev-workspace/ansible-dev/generator/generated-output/extensions/molecule

# Go back to generator dir
cd /Users/piyushmalik/dev-workspace/ansible-dev/generator
```

The generator will:
- Read the JSONL recording
- Match recording phases to states in `tests/integration/targets/ios_hostname/vars/main.yaml`
- Output molecule scenario files under `generated-output/extensions/molecule/`

### Step 5: Inspect the output

The generated output will be in `generated-output/extensions/molecule/` and includes:
- `hostname/converge.yml` — the molecule test playbook
- `hostname/molecule.yml` — molecule config with CISSHGO platforms
- `hostname/vars.yml` — port assignments
- `hostname/inventories/` — host inventory for CISSHGO
- CISSHGO transcript fixtures

---

## What the recording playbook does

`record_ios_hostname.yml` exercises these states in order:

1. **Setup**: Deletes any hostname, then sets `hostname box1`
2. **Gathered**: Runs `state: gathered` to capture current config
3. **Merged**: Sets `hostname boxTest`, then runs idempotent check
4. **Setup for deleted**: Resets to `hostname box1`
5. **Deleted**: Runs `state: deleted`, then runs idempotent check
6. **Cleanup**: Final delete

This matches the flow expected by `vars/main.yaml`:
- `gathered.config.hostname: "box1"`
- `merged.commands: ["hostname boxTest"]`
- `deleted.before.hostname: "box1"`, `deleted.commands: ["no hostname box1"]`

---

## Running Molecule Tests (Hand-Authored Scenarios)

Molecule tests use CISSHGO (a Go-based IOS CLI simulator) to replay transcripts
offline — no real device needed.

### Prerequisites

From `ansible-dev/generator/` (with venv activated):

```bash
# molecule-plugins is needed for the default driver
pip install molecule-plugins
```

### Running from the collection root

Molecule must run from the collection's `extensions/` directory. The hand-authored
scenarios live in the cisco.ios collection itself:

```bash
cd /Users/piyushmalik/dev-workspace/ansible-dev/ansible_collections/cisco/ios/extensions

# Run the hostname scenario
molecule test -s hostname

# Run interfaces scenario
molecule test -s interfaces

# Run l2_interfaces scenario
molecule test -s l2_interfaces
```

### What happens under the hood

1. `_shared/create.yml` downloads (or locates) the CISSHGO binary
2. CISSHGO starts, listening on localhost ports, replaying transcript fixtures
3. `converge.yml` runs the module against CISSHGO (simulated device)
4. `_shared/verify.yml` runs any post-checks
5. `_shared/destroy.yml` kills the CISSHGO process

### If CISSHGO download fails (macOS tar issue)

On macOS, the default BSD tar may fail to extract. Install GNU tar:

```bash
brew install gnu-tar
export PATH="/opt/homebrew/opt/gnu-tar/libexec/gnubin:$PATH"
```

Then re-run `molecule test`.

### Running generated scenarios

The generated output doesn't have its own `_shared/` directory. To test it,
copy the generated files into the collection's molecule structure:

```bash
# Copy generated hostname scenario (overwrites hand-authored)
cp -r generated-output/extensions/molecule/hostname/* \
  ../ansible_collections/cisco/ios/extensions/molecule/hostname/

# Copy generated transcript fixtures
cp -r generated-output/extensions/molecule/cisshgo_fixtures/transcripts/ios_hostname/* \
  ../ansible_collections/cisco/ios/extensions/molecule/cisshgo_fixtures/transcripts/ios_hostname/

# Copy generated inventories
cp generated-output/extensions/molecule/cisshgo_fixtures/inventories/ansible/hostname.yaml \
  ../ansible_collections/cisco/ios/extensions/molecule/cisshgo_fixtures/inventories/ansible/hostname.yaml
cp generated-output/extensions/molecule/cisshgo_fixtures/inventories/cisshgo/hostname.yaml \
  ../ansible_collections/cisco/ios/extensions/molecule/cisshgo_fixtures/inventories/cisshgo/hostname.yaml

# Then run molecule from extensions/
cd ../ansible_collections/cisco/ios/extensions
molecule test -s hostname
```

Note: The generated scenario's `converge.yml` includes an idempotent check for `deleted`
state which will fail (see Known Issues below).

---

## Known Issues

- **Deleted idempotent**: After `no hostname box1`, IOS reverts to `hostname Router` (default).
  Re-running `state: deleted` sees "Router" and reports `changed: true`.
  The generator adds idempotent checks for all states including deleted — this will fail.
  Hand-authored scenarios intentionally skip the idempotent check for deleted.

---

## Directory Structure (after completing all steps)

```
generator/
├── README.md                          # This file
├── record_ios_hostname.yml            # Recording playbook
├── recordings/
│   └── ios_hostname.jsonl             # Raw JSONL recording
├── generate_molecule_from_recording.py # Generator script
├── generated-output/
│   └── extensions/molecule/hostname/  # Generated molecule scenario
│       ├── converge.yml
│       ├── molecule.yml
│       ├── vars.yml
│       └── inventories/
├── collections/                       # Installed collections (gitignored)
└── .venv/                             # Virtual env (gitignored)
```
