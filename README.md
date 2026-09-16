# Automated Backup Script (Bash + Cron)

A Bash script that finds files modified in the last 24 hours in a target directory, archives them into a compressed `.tar.gz`, and moves the archive to a destination directory. Scheduled to run automatically once a day via `cron`. Built as the final capstone project for IBM's Introduction to Linux and Bash Scripting course.

## The scenario

The project is framed around a real operational problem: a company where interns manually check encrypted password files on core servers every day and back up whatever changed in the last 24 hours. That manual process is slow and error-prone. The goal was to replace it with a script that does the whole thing automatically and reliably, on a schedule.

## Step 1: Validate arguments

The script takes two arguments: a target directory (what to back up) and a destination directory (where to put the backup). Before anything else, it checks that exactly two arguments were passed and that both are real, valid directories.

```bash
if [[ $# != 2 ]]
then
  echo "backup.sh target_directory_name destination_directory_name"
  exit
fi

if [[ ! -d $1 ]] || [[ ! -d $2 ]]
then
  echo "Invalid directory path provided"
  exit
fi
```

## Step 2: Assign readable variable names and print them

```bash
targetDirectory=$1
destinationDirectory=$2

echo "Target Directory: $targetDirectory"
echo "Destination Directory: $destinationDirectory"
```

## Step 3: Build a timestamped backup filename

```bash
currentTS=$(date +%s)
backupFileName="backup-$currentTS.tar.gz"
```

`date +%s` gives the current time as a Unix timestamp (seconds since epoch), so each backup file gets a unique name like `backup-1789569479.tar.gz`.

## Step 4: Track absolute paths so the script can move freely

The script needs to `cd` into different directories later, so it captures the full absolute paths up front. That way it can always find its way back regardless of where it currently is.

```bash
origAbsPath=$(pwd)

cd "$destinationDirectory" || exit
destDirAbsPath=$(pwd)

cd "$origAbsPath" || exit
cd "$targetDirectory" || exit
```

## Step 5: Find files modified in the last 24 hours

Calculate yesterday's timestamp, then loop over every file in the target directory. For each one, compare its last-modified time against the 24-hour cutoff, and if it qualifies, add it to a Bash array.

```bash
yesterdayTS=$(($currentTS - 24 * 60 * 60))

declare -a toBackup

for file in *
do
  if [[ `date -r $file +%s` -gt $yesterdayTS ]]
  then
    toBackup+=($file)
  fi
done
```

**Bug I ran into here:** my first version used `(( ))` arithmetic brackets with a `-gt` comparison operator inside it:

```bash
# This is broken - don't use this
if (( `date -r $file +%s` -gt $yesterdayTS ))
```

That's a bracket-type mismatch. `-gt`/`-lt`/`-eq` style operators only work inside `[[ ]]` test brackets, not `(( ))` arithmetic brackets, which expect symbols like `>` instead. It threw a genuinely confusing error ("syntax error in expression") where the printed error message split the number across the line in a way that looked like a hidden/corrupted character at first glance. I traced it by echoing the raw values at each step, ruling out line-ending issues, and testing the two bracket types side by side until the actual cause was clear. Switching to `[[ ]]` fixed it immediately.

## Step 6: Archive and move the backup

Pass the whole `toBackup` array to `tar` to create one compressed archive, then move it into the destination directory.

```bash
tar -czvf $backupFileName ${toBackup[@]}

mv $backupFileName "$destDirAbsPath"
```

## Step 7: Deploy and schedule with cron

Copy the script somewhere callable from anywhere on the system, then schedule it with `crontab` to run once every 24 hours.

```bash
chmod +x backup.sh
sudo cp backup.sh /usr/local/bin/
```

Before setting the real schedule, I tested that cron actually triggers the script by scheduling it every 1 minute first and confirming new backup files appeared automatically:

```bash
crontab -e
# */1 * * * * /usr/local/bin/backup.sh /home/project/important-documents /home/project

sudo service cron start
# ...wait, confirm new .tar.gz files appear...
sudo service cron stop
```

Once confirmed working, swapped in the real daily schedule:

```bash
crontab -e
# 0 0 * * * /usr/local/bin/backup.sh /home/project/important-documents /home/project
```

## Verification

Each output file confirms one part of the pipeline actually works, not just that the code looks right:

**`outputs/backup-permissions`** — confirms the script has executable permissions after `chmod +x`.

**`outputs/backup-file-check`** — confirms a correctly-named backup archive (`backup-[TIMESTAMP].tar.gz`) was actually created after running the script against a real test directory.

**`outputs/backup-script-copy`** — confirms the script was successfully copied to `/usr/local/bin/`.

**`outputs/crontab-schedule`** — confirms the cron job is scheduled to run once every 24 hours (`0 0 * * *`).

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
