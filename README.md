# mariadb-backup

This repository contains a few scripts for automating backups with mariabackup. Originally based on <a href="https://github.com/do-community/ubuntu-1604-mysql-backup">do-community/ubuntu-1604-mysql-backup</a>

Please check and follow the instructions below. Original instructions are taken from <a href="https://www.digitalocean.com/community/tutorials/how-to-configure-mysql-backups-with-percona-xtrabackup-on-ubuntu-16-04">here</a>.

## Prerequisites
- Install pigz
    ```sudo apt install pigz```

## Create a MySQL User with Appropriate Privileges

The first thing we need to do is create a new MySQL user configured to handle backup tasks. We will only give this user the privileges it needs to copy the data safely while the system is running.

To be explicit about the account's purpose, we will call the new user backup. We will be placing the user's credentials in a secure file, so feel free to choose a complex password:

```
mysql> CREATE USER 'backup'@'localhost' IDENTIFIED BY 'password';
```

Next we need to grant the new **backup** user the permissions it needs to perform all backup actions on the database system. Grant the required privileges and apply them to the current session by typing:

```
mysql> GRANT SELECT, RELOAD, PROCESS, LOCK TABLES, REPLICATION CLIENT, SHOW VIEW  ON *.* TO 'backup'@'localhost';
mysql> FLUSH PRIVILEGES;
```

Our MySQL backup user is configured and has the access it requires.

## Display the value of the datadir variable 

```
mysql> SELECT @@datadir;

+-----------------+
| @@datadir       |
+-----------------+
| /var/lib/mysql/ |
+-----------------+
1 row in set (0.01 sec)
```

Take a note of the location you find.

## Configuring a Systems Backup User and Assigning Permissions

Now that we have a MySQL user to perform backups, we will ensure that a corresponding Linux user exists with similar limited privileges.

On Ubuntu 16.04 / Debian 8, a backup user and corresponding backup group is already available. Confirm this by checking the /etc/passwd and /etc/group files with the following command:

```
grep backup /etc/passwd /etc/group

/etc/passwd:backup:x:34:34:backup:/var/backup:/usr/sbin/nologin
/etc/group:backup:x:34:

```
The first line from the /etc/passwd file describes the backup user, while the second line from the /etc/group file defines the backup group.

The /var/lib/mysql directory where the MySQL data is kept is owned by the mysql user and group. We can add the backup user to the mysql group to safely allow access to the database files and directories. We should also add our sudo user to the backup group so that we can access the files we will back up.

Type the following commands to add the backup user to the mysql group and your sudo user to the backup group:

```
sudo usermod -aG mysql backup
sudo usermod -aG backup ${USER}
```

If we check the /etc/group files again, you will see that your current user is added to the backup group and that the backup user is added to the mysql group:

```
grep backup /etc/group

backup:x:34:sammy
mysql:x:116:backup
```

The new group isn't available in our current session automatically. To re-evaluate the groups available to our sudo user, either log out and log back in, or type:

```
exec su - ${USER}
```

You will be prompted for your sudo user's password to continue. Confirm that your current session now has access to the backup group by checking our user's groups again:

```
id -nG

sammy sudo backup
```

Our sudo user will now be able to take advantage of its membership in the backup group.

Next, we need to make the /var/lib/mysql directory and its subdirectories accessible to the mysql group by adding group execute permissions. Otherwise, the backup user will be unable to enter those directories, even though it is a member of the mysql group.

> **Note**: If the value of datadir was not **/var/lib/mysql** when you checked inside of MySQL earlier, substitute the directory you discovered in the commands that follow.

To give the mysql group access to the MySQL data directories, type:

```
sudo find /var/lib/mysql -type d -exec chmod 750 {} \;
```

> **Note**: MariaDB creates directories for new databases without read permission for group, so we need to create /etc/systemd/system/mariadb.service.d/umask.conf file with next contents:

```conf
[Service]

Environment="UMASK=0660"
Environment="UMASK_DIR=0750"
```

Our backup user now has the access it needs to the MySQL directory.

## Creating the Backup Assets

Now that MySQL and system backup users are available, we can begin to set up the configuration files, encryption keys, and other assets that we need to successfully create and secure our backups.

### Create a MySQL Configuration File with the Backup Parameters

Begin by creating a minimal MySQL configuration file that the backup script will use. This will contain the MySQL credentials for the MySQL user.

Open a file at **/var/backup/.my.cnf** (where **/var/backup** is home of backup user) in your text editor:

```
sudo mkdir -p /var/backup
sudo vim /var/backup/.my.cnf
```

Inside, start a ```[client]``` section and set the MySQL backup user and password user you defined within MySQL:

```conf
[client]
user=backup
password=password
```

Save and close the file when you are finished.

Give ownership of the file to the backup user and then restrict the permissions so that no other users can access the file:

```
sudo chown backup /var/backup/.my.cnf
sudo chmod 600 /var/backup/.my.cnf
```

The backup user will be able to access this file to get the proper credentials but other users will be restricted.

### Create a Backup Root Directory

Next, create a directory for the backup content. We will use **/var/backup/mysql** as the base directory for our backups:

```
sudo mkdir -p /var/backup/mysql
```

Next, assign ownership of the **/var/backup/mysql** directory to the backup user and group ownership to the mysql group:

```
sudo chown backup:mysql /var/backup/mysql
```

The ```backup``` user should now be able to write backup data to this location.

## Downloading the Backup and Restore Scripts

You can download the scripts directly from GitHub by typing:

```
cd /tmp
curl -LO https://raw.githubusercontent.com/capcom6/mariadb-backup/master/backup-mysql.sh
curl -LO https://raw.githubusercontent.com/capcom6/mariadb-backup/master/extract-mysql.sh
curl -LO https://raw.githubusercontent.com/capcom6/mariadb-backup/master/prepare-mysql.sh
curl -LO https://raw.githubusercontent.com/capcom6/mariadb-backup/master/common.inc
curl -LO https://raw.githubusercontent.com/capcom6/mariadb-backup/master/config.inc
```

Be sure to inspect the scripts after downloading to make sure they were retrieved successfully and that you approve of the actions they will perform. If you are satisfied, mark the scripts as executable and then move them into the **/usr/local/bin** directory by typing:

```
chmod +x /tmp/{backup,extract,prepare}-mysql.sh
sudo mv /tmp/{backup,extract,prepare}-mysql.sh /usr/local/bin
sudo mkdir -p /var/lib/backup-mysql/
sudo mkdir -p /etc/backup-mysql/
sudo mv /tmp/common.inc /etc/backup-mysql/
sudo mv /tmp/config.inc /var/lib/backup-mysql/
sudo chmod +x /var/lib/backup-mysql/config.inc
sudo chmod +x /etc/backup-mysql/common.inc
```

## Using the Backup and Restore Scripts

In order to make our backup and restore steps repeatable, we will script the entire process. We will use the following scripts:

* config.inc: common parameters for all scripts;
* common.inc: common functions for all scripts;
* backup-mysql.sh: This script backs up the MySQL databases, encrypting and compressing the files in the process. It creates full and incremental backups and automatically organizes content by day. By default, the script maintains 3 days worth of backups.
* extract-mysql.sh: This script decompresses and decrypts the backup files to create directories with the backed up content.
* prepare-mysql.sh: This script "prepares" the back up directories by processing the files and applying logs. Any incremental backups are applied to the full backup. Once the prepare script finishes, the files are ready to be moved back to the data directory.

### The backup-mysql.sh Script

The script has the following functionality:

* Creates a compressed full backup the first time it is run each day.
* Generates compressed incremental backups based on the daily full backup when called again on the same day.
* Maintains backups organized by day. By default, three days of backups are kept. This can be changed by adjusting the `days_of_backups` parameter within the **config.inc**.

When the script is run, a daily directory is created where timestamped files representing individual backups will be written. The first timestamped file will be a full backup, prefixed by full-. Subsequent backups for the day will be incremental backups, indicated by an incremental- prefix, representing the changes since the last full or incremental backup.

Backups will generate a file called **backup-progress.log** in the daily directory with the output from the most recent backup operation. A file called **xtrabackup_checkpoints** containing the most recent backup metadata will be created there as well. This file is needed to produce future incremental backups, so it is important not to remove it. A file called **xtrabackup_info**, which contains additional metadata, is also produced but the script does not reference this file.

### The extract-mysql.sh Script

Unlike the **backup-mysql.sh** script, which is designed to be automated, this script is designed to be used intentionally when you plan to restore from a backup. Because of this, the script expects you to pass in the .xbstream files that you wish to extract.

The script creates a **restore** directory within the current directory and then creates individual directories within for each of the backups passed in as arguments. It will process the provided .xbstream files by extracting directory structure from the archive, decrypting the individual files within, and then decompressing the decrypted files.

After this process has completed, the **restore** directory should contain directories for each of the provided backups. This allows you to inspect the directories, examine the contents of the backups, and decide which backups you wish to prepare and restore.

### The prepare-mysql.sh Script

This script will apply the logs to each backup to create a consistent database snapshot. It will apply any incremental backups to the full backup to incorporate the later changes.

The script expects single parameter - **restore** directory, created by **extract-mysql.sh**. The script looks for directories beginning with full- or incremental-. It uses the MySQL logs to apply the committed transactions to the full backup. Afterwards, it applies any incremental backups to the full backup to update the data with the more recent information, again applying the committed transactions.

Once all of the backups have been combined, the uncommitted transactions are rolled back. At this point, the full- backup will represent a consistent set of data that can be moved into MySQL's data directory.

In order to minimize chance of data loss, the script stops short of copying the files into the data directory. This way, the user can manually verify the backup contents and the log file created during this process, and decide what to do with the current contents of the MySQL data directory. The commands needed to restore the files completely are displayed when the command exits.

## Testing the MySQL Backup and Restore Scripts

### Perform a Full Backup

```
sudo -u backup backup-mysql.sh

Backup successful!
Backup created at /var/backup/mysql/2021-07-23/full-07-23-2021_14-32-45.xbstream.gz

```

If everything went as planned, the script will execute correctly, indicate success, and output the location of the new backup file. As the above output indicates, a daily directory ("2021-07-23" in this case) has been created to house the day's backups. The backup file itself begins with full- to express that this is a full backup.

Let's move into the daily backup directory and view the contents:

```
cd /var/backup/mysql/"$(date +%Y-%m-%d)"
ls

backup-progress.log  full-07-23-2021_14-32-45.xbstream.gz  xtrabackup_checkpoints  xtrabackup_info

```

Here, we see the actual backup file (**full-07-23-2021_14-32-45.xbstream.gz** in this case), the log of the backup event (backup-progress.log), the **xtrabackup_checkpoints** file, which includes metadata about the backed up content, and the xtrabackup_info file, which contains additional metadata.

If we tail the **backup-progress.log**, we can confirm that the backup completed successfully.

```
tail backup-progress.log

[00] 2021-07-23 14:32:47 Streaming ib_buffer_pool to <STDOUT>
[00] 2021-07-23 14:32:47         ...done
[00] 2021-07-23 14:32:47 Backup created in directory '/var/backup/mysql/2021-07-23/'
[00] 2021-07-23 14:32:47 MySQL binlog position: filename 'mariadb-bin.000002', position '5077', GTID of the last change '0-99-29'
[00] 2021-07-23 14:32:47 Streaming backup-my.cnf
[00] 2021-07-23 14:32:47         ...done
[00] 2021-07-23 14:32:47 Streaming xtrabackup_info
[00] 2021-07-23 14:32:47         ...done
[00] 2021-07-23 14:32:47 Redo log (from LSN 155365 to 155374) was copied.
[00] 2021-07-23 14:32:47 completed OK!

```

If we look at the **xtrabackup_checkpoints** file, we can view information about the backup. While this file provides some information that is useful for administrators, it's mainly used by subsequent backup jobs so that they know what data has already been processed.

This is a copy of a file that's included in each archive. Even though this copy is overwritten with each backup to represent the latest information, each original will still be available inside the backup archive.

```
cat xtrabackup_checkpoints

backup_type = full-backuped
from_lsn = 0
to_lsn = 155365
last_lsn = 155374

```

The example above tells us that a full backup was taken and that the backup covers log sequence number (LSN) 0 to log sequence number 155365. The last_lsn number indicates that some operations occurred during the backup process.

### Perform an Incremental Backup

Now that we have a full backup, we can take additional incremental backups. Incremental backups record the changes that have been made since the last backup was performed. The first incremental backup is based on a full backup and subsequent incremental backups are based on the previous incremental backup.

The **backup-mysql.sh** script will take an incremental backup if a full backup for the same day exists:

```
sudo -u backup backup-mysql.sh

Backup successful!
Backup created at /var/backup/mysql/2021-07-23/incremental-07-23-2021_14-36-25.xbstream.gz

```

Check the daily backup directory again to find the incremental backup archive:

```
cd /var/backup/mysql/"$(date +%Y-%m-%d)"
ls

backup-progress.log                   incremental-07-23-2021_14-36-25.xbstream.gz  xtrabackup_info
full-07-23-2021_14-32-45.xbstream.gz  xtrabackup_checkpoints

```

The contents of the **xtrabackup_checkpoints** file now refer to the most recent incremental backup:

```
cat xtrabackup_checkpoints

backup_type = incremental
from_lsn = 155365
to_lsn = 155365
last_lsn = 155374
```

The backup type is listed as "incremental" and instead of starting from LSN 0 like our full backup, it starts at the LSN where our last backup ended.

### Extract the Backups

Next, let's extract the backup files to create backup directories. Due to space and security considerations, this should normally only be done when you are ready to restore the data.

We can extract the backups by passing the .xbstream backup files to the **extract-mysql.sh** script. We can run from any user with access to backups. Based on current script configuration, you must execute this command from the directory of the backup (i.e. 2021-07-23):

```
cd /var/backup/mysql/2021-07-23
sudo -u backup extract-mysql.sh /var/backup/mysql/2021-07-23/*.xbstream.gz

Extraction complete! Backup directories have been extracted to the "restore" directory.

```

The above output indicates that the process was completed successfully. If we check the contents of the current directory, an **extract-progress.log** file and a restore directory have been created.

If we tail the extraction log, we can confirm that the latest backup was extracted successfully. The other backup success messages are displayed earlier in the file.

```
tail extract-progress.log

[00] 2021-07-23 14:40:17 xtrabackup_binlog_info
[00] 2021-07-23 14:40:17 ib_logfile0
[00] 2021-07-23 14:40:17 xtrabackup_checkpoints
[00] 2021-07-23 14:40:17 ib_buffer_pool
[00] 2021-07-23 14:40:17 backup-my.cnf
[00] 2021-07-23 14:40:17 xtrabackup_info


Finished work on /var/backup/mysql/2021-07-23/incremental-07-23-2021_14-36-25.xbstream.gz
```

If we move into the restore directory, directories corresponding with the backup files we extracted are now available:

```
cd restore
ls -F

full-07-23-2021_14-32-45.xbstream.gz/  incremental-07-23-2021_14-36-25.xbstream.gz/

```

The backup directories contains the raw backup files, but they are not yet in a state that MySQL can use though. To fix that, we need to prepare the files.

### Prepare the Final Backup

Next, we will prepare the backup files. To do so, you must run **prepare-mysql.sh** with restore directory that contains the full- and any incremental- backups as argument. The script will apply the changes from any incremental- directories onto the full- backup directory. Afterwards, it will apply the logs to create a consistent dataset that MySQL can use.

If for any reason you don't want to restore some of the changes, now is your last chance to remove those incremental backup directories from the restore directory (the incremental backup files will still be available in the parent directory). Any remaining incremental- directories within the current directory will be applied to the full- backup directory.

When you are ready, call the **prepare-mysql.sh** script (replacing your full path to the restore directory):

```
sudo -u backup prepare-mysql.sh /var/backup/mysql/2021-07-23/restore

    Backup looks to be fully prepared.  Please check the "prepare-progress.log" file
    to verify before continuing.

    If everything looks correct, you can apply the restored files.

    First, stop MySQL and move or remove the contents of the MySQL data directory:
        
            sudo systemctl stop mariadb
            sudo mv /var/lib/mysql/ /tmp/
        
    Then, recreate the data directory and  copy the backup files:
        
            sudo mkdir /var/lib/mysql
            sudo mariabackup --copy-back --target-dir=./restore/full-07-23-2021_14-32-45.xbstream.gz
        
    Afterward the files are copied, adjust the permissions and restart the service:
        
            sudo chown -R mysql:mysql /var/lib/mysql
            sudo find /var/lib/mysql -type d -exec chmod 755 {} \;
            sudo systemctl start mariadb

```

The output above indicates that the script thinks that the backup is fully prepared and that the full- backup now represents a fully consistent dataset. As the output states, you should check the **prepare-progress.log** file to confirm that no errors were reported during the process.

The script stops short of actually copying the files into MySQL's data directory so that you can verify that everything looks correct.

### Restore the Backup Data to the MySQL Data Directory

If you are satisfied that everything is in order after reviewing the logs, you can follow the instructions outlined in the prepare-mariadb.sh output.

First, stop the running MySQL process:

```
sudo systemctl stop mariadb
```

Since the backup data may conflict with the current contents of the MySQL data directory, we should remove or move the **/var/lib/mysql** directory. If you have space on your filesystem, the best option is to move the current contents to the **/tmp** directory or elsewhere in case something goes wrong:

```
sudo mv /var/lib/mysql/ /tmp
```

Recreate an empty **/var/lib/mysql** directory. We will need to fix permissions in a moment, so we do not need to worry about that yet:

```
sudo mkdir /var/lib/mysql
```

Now, we can copy the full backup to the MySQL data directory using the **mariabackup** utility. Substitute the path to your prepared full backup in the command below:

```
sudo mariabackup --copy-back --target-dir=./restore/full-07-23-2021_14-32-45.xbstream.gz
```

A running log of the files being copied will display throughout the process. Once the files are in place, we need to fix the ownership and permissions again so that the MySQL user and group own and can access the restored structure:

```
sudo chown -R mysql:mysql /var/lib/mysql
sudo find /var/lib/mysql -type d -exec chmod 750 {} \;
``` 

Our restored files are now in the MySQL data directory.

Start up MySQL again to complete the process:

```
sudo systemctl start mariadb
```

After restoring your data, it is important to go back and delete the restore directory. Future incremental backups cannot be applied to the full backup once it has been prepared, so we should remove it. Furthermore, the backup directories should not be left unencrypted on disk for security reasons:

```
sudo rm -rf ./restore
```

The next time we need a clean copies of the backup directories, we can extract them again from the backup files.

## Creating a Cron Job to Run Backups Hourly

Now that we've verified that the backup and restore process are working smoothly, we should set up a cron job to automatically take regular backups.

We will create a small script within the backup user crons to automatically run our backup script and log the results. The cron process will automatically run this every 15 minutes:

```
sudo crontab -u backup -e
```

Inside, we will call the backup script with the systemd-cat utility so that the output will be available in the journal. We'll mark them with a backup-mysql identifier so we can easily filter the logs:

```
*/15 * *   *   *     /usr/local/bin/backup-mysql.sh
```

Save and close the file when you are finished. Make the script executable by typing:


The backup script will now run every 15 minutes. The script itself will take care of cleaning up backups older than three days ago.

We can test the cron script by running it manually:

```
sudo /etc/cron.hourly/backup-mysql
```

After it completes, check the journal for the log messages by typing:

```
sudo journalctl -t backup-mysql

июл 23 00:00:04 mdb99.*****.tech backup-mysql[8915]: Backup successful!
июл 23 00:00:04 mdb99.*****.tech backup-mysql[8915]: Backup created at /var/backup/mysql/2021-07-24/full-07-24-2021_00-00-02.xbstream.gz
```

Check back in a few hours to make sure that additional backups are being taken.

## Hope this helps! Cheers!
