# GithubAutoGreen

[![ci](https://github.com/AcideFluorhydrique/GithubAutoGreen/actions/workflows/ci.yml/badge.svg)](https://github.com/AcideFluorhydrique/GithubAutoGreen/actions/workflows/ci.yml)

A stochastic GitHub activity generator and **1D symmetric random walk** experiment powered by GitHub Actions.

Instead of producing a uniform, robotic grid of daily commits, this workflow simulates natural weekday developer activity while driving the line count of `notes.md` as a bounded random walk.

## How It Works

### 1. Stateless Weekly Quotas
At the start of each run, the workflow hashes the ISO week identifier (`YYYYWW`) with a fixed salt (`SHA-256`) to deterministically assign a workload profile for the current week without requiring external state or caches:

| Week Profile | Probability | Weekly Target (`TARGET`) |
| :--- | :---: | :--- |
| **Idle Week** | 15% | `0` or `1` commit |
| **Normal Week** | 70% | `3` to `5` commits |
| **Busy Week** | 15% | `5` commits (1 per weekday) |

### 2. Dynamic Pacing & Temporal Jitter
* **Weekdays Only:** Runs Monday through Friday (`1 1 * * 1-5` UTC). Weekends are explicitly skipped—even on manual `workflow_dispatch` triggers—leaving Saturday and Sunday reserved for organic commits.
* **Randomized Timing:** Scheduled runs sleep for a random interval of `0–14,400` seconds (up to 4 hours) before executing, spreading timestamps naturally across morning/midday working hours.
* **Adaptive Daily Probability:** Let `LEFT` be the remaining weekly quota and `REMAIN` be the remaining weekdays. If `LEFT >= REMAIN`, the script commits the required catch-up count (`N`); otherwise, it commits with probability `LEFT / REMAIN`. Multiple commits in a single day are spaced by `30–630` seconds.

### 3. 1D Symmetric Random Walk (`notes.md`)
Rather than growing a log file indefinitely, `notes.md` is initialized with `100` seed lines ($L_0 = 100$) and evolves as a discrete-time random walk with a reflecting barrier at $L = 1$:

$$L_{n+1} = \begin{cases} L_n + 1, & p = 0.5 \quad (\text{append a timestamped note}) \\ L_n - 1, & p = 0.5 \quad (\text{delete a random line, when } L_n > 1) \end{cases}$$

When $L_n = 1$, the workflow forces an insertion ($P(L_{n+1} = 2 \mid L_n = 1) = 1$) so the file is never emptied. Over a typical year (~188 commits), the file length drifts naturally around $100 \pm 14$ lines, turning the Git history of `notes.md` into a record of a Brownian-like trajectory.

## Usage

1. **Create Your Repository:** Click **Use this template** -> **Create a new repository** at the top right.
   > **Warning:** Do **not** fork this repository. GitHub does not count commits made to forked repositories toward your contribution graph.
2. **Configure Git Identity:** Edit `.github/workflows/ci.yml` and update `USER_EMAIL` and `USER_NAME` with your GitHub noreply email and username:
   ```yaml
   env:
     TOKEN: ${{ secrets.GITHUB_TOKEN }}
     REPO: ${{ github.repository }}
     USER_EMAIL: "your-id+username@users.noreply.github.com"
     USER_NAME: "username"
   ```
3. **Optional Customization:**
   * Change `SALT="plan-salt"` in `ci.yml` to generate your own unique sequence of idle/normal/busy weeks.
   * Adjust the `cron` schedule (`1 1 * * 1-5`) to align the 4-hour jitter window with your local timezone.
4. **Private Repository Visibility (Optional):** If you keep your repository private, enable **Private contributions** under **Profile -> Contribution settings** so the activity appears on your public profile.

## License
[MIT License](./LICENSE)
