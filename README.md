# NodeGrab + TmuxShell

Interactive compute node allocation for Open OnDemand with persistent tmux
sessions. Users request a Slurm allocation through a web form and connect to it
via a one-click terminal that maintains session continuity across browser
disconnections, network interruptions, and client restarts.

Two apps work together:

- **NodeGrab** -- OOD Batch Connect app. Submits the Slurm job, starts a tmux
  session inside the allocation's batch step cgroup, and renders a "Connect"
  button in the dashboard.

- **TmuxShell** -- Modified OOD shell app. When called with `?jobid=X&ood_session_id=Y`
  query params, it runs `srun --overlap --pty tmux attach-session` into the
  allocation instead of a plain SSH login. Without those params, it behaves as
  a normal tmux-wrapped shell (or falls back to stock behavior with no params
  at all).


## How it works

```
[User clicks "Connect"]
    |
    v
TmuxShell/app.js  -- detects ?jobid= in URL
    |
    v
ssh login-node -t bash -l -c "srun --jobid=XXXX --overlap --pty tmux attach-session -t SESSION_ID"
    |
    v
Attaches to the tmux server started by script.sh.erb in the batch step cgroup
```

The tmux server runs in the job's main cgroup (the batch step), not inside an
srun step. This means it stays alive when srun connections come and go. Slurm
kills it when the job ends. The cleanup trap in script.sh.erb also explicitly
kills the tmux session on job exit as a belt-and-suspenders measure.

WebSocket timeout enforcement is skipped for Slurm-managed sessions
(`ws.slurmManaged = true`). The job's walltime controls the session lifetime,
not OOD's idle/max timers.


## Dependencies

- Open OnDemand >= 3.0 (tested with 3.1)
- Slurm (any reasonably recent version with `srun --overlap` support)
- tmux (must be available on both login and compute nodes)
- The stock OOD shell app's Node.js dependencies (no new packages added)


## TmuxShell: what changed from stock

TmuxShell is a copy of the upstream `apps/shell` from
[OSC/ondemand](https://github.com/OSC/ondemand) (master branch) with changes
to exactly two files:

**`app.js`** -- two modification blocks, clearly marked with
`BEGIN MODIFICATION` / `END MODIFICATION` comments:

1. Connection routing (around line 220): Replaces the single `args = [host, ...]`
   line with a three-way branch based on query params (`jobid`, `ood_session_id`,
   or neither).

2. Timeout bypass (around line 330): Skips idle/max timeout enforcement for
   connections where `ws.slurmManaged === true`.

**`manifest.yml`** -- name and description changed.

Everything else (views, utils, public assets, color themes, package.json) is
untouched. When OOD upstream updates the shell app, you only need to re-apply
these two blocks.


## Installation

### 1. Deploy TmuxShell

Copy the stock shell app and apply the modified `app.js`:

```
cd /var/www/ood/apps/sys
cp -r shell TmuxShell
```

Replace `TmuxShell/app.js` with the version from this repo. Replace
`TmuxShell/manifest.yml` likewise.

Run `bin/setup` inside the TmuxShell directory to install Node.js dependencies
(or just leave the existing `node_modules` from the copy).

### 2. Deploy NodeGrab

```
cd /var/www/ood/apps/sys
cp -r /path/to/this/repo/NodeGrab .
```

No `bin/setup` or `npm install` needed. NodeGrab is a pure Batch Connect app
with only ERB templates.

### 3. Restart the PUN

Users need to restart their PUN (Help -> Restart Web Server in the dashboard)
or you can do it globally:

```
sudo /opt/ood/nginx_stage/sbin/nginx_stage nginx_clean -f
```


## Adoption checklist

Things you MUST change before deploying:

- [ ] `NodeGrab/form.yml.erb` -- Edit the "SITE CONFIGURATION" block at the top:
  - `cluster_name` -- your cluster name from `/etc/ood/config/clusters.d/`
  - `slurm_bin` -- path to sacctmgr/sinfo/scontrol (or `/usr/bin` if in PATH)
  - `default_partition` -- your default partition name
  - `hidden_partitions` -- regex of partitions to exclude, or `nil`
  - `max_cores_per_node` -- matches your hardware
  - `default_mem` -- sensible default for your nodes
  - `gpu_enabled` / `gpu_description` -- set false if no GPUs

- [ ] `NodeGrab/submit.yml.erb` -- Match the `default_cores` / `default_mem` /
  `default_nodes` / `default_hours` to match `form.yml.erb`

- [ ] `NodeGrab/template/script.sh.erb` -- **Treat this as a reference
  implementation, not a drop-in.** The provided script is intentionally minimal.
  Before deploying, review and adapt it for your site: add module loads,
  environment initialization, pre-session configuration, and any resource
  validation your jobs require. The three structural responsibilities of the
  script (write `connection.yml`, start the tmux session, sleep for the
  walltime) must be preserved, but everything around them is yours to customize.

- [ ] `NodeGrab/view.html.erb` -- Edit the two variables at the top:
  - `cluster_name` -- same as form.yml.erb
  - `tmux_shell_app` -- directory name of TmuxShell under apps/sys

- [ ] `TmuxShell/manifest.yml` -- Adjust `category` / `subcategory` to fit
  your site's OOD navigation structure, and set `show_in_menu`:
  - **`show_in_menu: false`** hides TmuxShell from the nav menu entirely.
    Use this if you only want it as a NodeGrab backend and don't want users
    opening it directly.
  - **`show_in_menu: true`** exposes TmuxShell in the Clusters menu as a
    standalone persistent shell that users can open independently of NodeGrab.

- [ ] **Icons** -- Both apps ship with FontAwesome icon references
  (`fa://server` for NodeGrab, `fa://terminal` for TmuxShell). If you want a
  custom tile image instead, drop an `icon.png` into the respective app
  directory (`NodeGrab/icon.png`, `TmuxShell/icon.png`). OOD will use the
  PNG over the FontAwesome reference when both are present.

Things to verify:

- [ ] tmux is installed on compute nodes (not just login nodes)
- [ ] `srun --overlap` works on your Slurm version
- [ ] Compute node hostnames are in `OOD_SSHHOST_ALLOWLIST` or your cluster
  config, otherwise TmuxShell will reject the SSH connection
- [ ] Test in dev mode first (`~/ondemand/dev/`) before deploying to sys


## Multi-cluster / GPU-heavy deployments

The form.yml.erb in this repo is a general-purpose single-cluster config. If
you need per-partition GPU dropdowns, per-QOS hour limits, or multi-cluster
routing, you will need to extend the form logic. The ERB header is designed so
all site-specific knobs are in one place at the top of the file. The dynamic
Slurm queries (sacctmgr, sinfo, scontrol) below that block should work without
modification on any standard Slurm installation.

For a GPU-heavy cluster with multiple GPU types per partition, consider
replacing the checkbox with a select widget and adding `--gres=gpu:TYPE:COUNT`
logic in submit.yml.erb.


## Authors

- Mehmet Belgin <mehmet.belgin@moffitt.org>
- Shane Corder <shane.corder@moffitt.org>
- Moffitt Cancer Center


## License

MIT. Same as Open OnDemand.
