# Automated Backup Script (Bash + Cron)

A Bash script that finds files modified in the last 24 hours in a target directory, archives them into a compressed `.tar.gz`, and moves the archive to a destination directory. Scheduled to run automatically once a day via `cron`. Built as the final capstone project for IBM's Introduction to Linux and Bash Scripting course.

## The scenario

The project is framed around a real operational problem: a company where interns manually check encrypted password files on core servers every day and back up whatever changed in the last 24 hours. That manual process is slow and error-prone. The goal was to replace it with a script that does the whole thing automatically and reliably, on a schedule.

## Step 1: Handle arguments and set up variables

The script takes two arguments: a target directory (what to back up) and a destination directory (where to put the backup). After validating both are real directories, it:

- Assigns the two arguments to readable variable names
- Prints them out for confirmation
- Generates a timestamp and builds a backup filename like `backup-1789569479.tar.gz`
- Captures the absolute paths of both the original and destination directories, so the script can move freely between directories without losing track of where it started

## Step 2: Find files modified in the last 24 hours

The script calculates yesterday's timestamp (current time minus 24 hours), then loops over every file in the target directory using a wildcard (`*`). For each file, it checks the file's last-modified time against that 24-hour cutoff and, if it qualifies, adds it to a Bash array (`toBackup`).

**Bug I ran into:** my first version of this check used `(( ))` arithmetic brackets with a `-gt` comparison operator inside it. That's a bracket-type mismatch — `-gt`/`-lt`/`-eq` style operators only work inside `[[ ]]` test brackets, not `(( ))` arithmetic brackets, which expect symbols like `>`. It threw a genuinely confusing error ("syntax error in expression") that looked like a corrupted variable at first, since the printed error message split the number across two lines in a way that looked like a hidden character. Traced it by echoing the raw values at each step, ruling out line-ending issues, and finally testing the two bracket types side by side until the actual cause was clear. Switched to `[[ ]]` and it worked immediately.

## Step 3: Archive and move the backup

Once the loop finishes, the script passes the whole `toBackup` array to `tar` to create a single compressed archive, then moves that archive into the destination directory.

## Step 4: Schedule it with cron

Rather than running the script by hand, it's copied to `/usr/local/bin/` (so it's callable from anywhere) and scheduled with `crontab` to run once every 24 hours.

## Verification

Each of the following confirms one part of the pipeline actually works, not just that the code looks right:

**`outputs/backup-permissions`** — confirms the script has executable permissions after `chmod +x`.

**`outputs/backup-file-check`** — confirms a correctly-named backup archive (`backup-[TIMESTAMP].tar.gz`) was actually created after running the script against a real test directory.

**`outputs/backup-script-copy`** — confirms the script was successfully copied to `/usr/local/bin/`.

**`outputs/crontab-schedule`** — confirms the cron job is scheduled to run once every 24 hours (`0 0 * * *`).

Before setting the real 24-hour schedule, I also tested the cron trigger itself by temporarily scheduling the job every 1 minute, confirming new backup files actually appeared automatically, then swapping in the real daily schedule.

## Skills demonstrated

- Bash scripting fundamentals: variables, command-line arguments, command substitution, arrays, loops, conditionals
- File modification time comparisons and timestamp arithmetic
- Archiving and compressing files with `tar`
- Deploying a script system-wide (`/usr/local/bin/`)
- Scheduling and testing recurring jobs with `cron`/`crontab`
- Debugging a real, non-obvious shell syntax error by isolating variables and testing assumptions one at a time

## Files

```
backup.sh              # The completed script (all 13 required tasks implemented)
outputs/
  backup-permissions    # Confirms executable permissions
  backup-file-check     # Confirms backup archive was created correctly
  backup-script-copy    # Confirms script deployed to /usr/local/bin/
  crontab-schedule      # Confirms the 24-hour cron schedule
```
