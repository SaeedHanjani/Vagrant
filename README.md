# Vagrant
Vagrant best peractice
# Install 2 os ubuntu  with Vagrant on windows11 and we want to backup a text file form os web to os db automatic every day

## update vagrant file:
```
Vagrant.configure("2") do |config|
  # Define the web server VM
  config.vm.define "web" do |web|
    web.vm.box = "ubuntu/jammy64"
    web.vm.network "private_network", type: "dhcp"
  end

  # Define the database server VM
  config.vm.define "db" do |db|
    db.vm.box = "ubuntu/jammy64"
    db.vm.network "private_network", type: "dhcp"
  end
end
```

This assigns both VMs a private network, so they can communicate internally.

Start Both VMs

Run:
```
vagrant up
```
Check VM IPs:
```
vagrant ssh web -c "ip a | grep inet"
vagrant ssh db -c "ip a | grep inet"
```
You'll see an IP like 192.168.56.101 for web and 192.168.56.102 for db.
### 🔹 Step 3: SSH from web to db

By default, Vagrant sets up SSH keys under .vagrant/machines/{vmname}/virtualbox/private_key.

    Copy db's private key to web
    From Windows, copy the db private key:
```
cp .vagrant\machines\db\virtualbox\private_key .vagrant\machines\web\virtualbox\db_key
```
Then SSH into web:
```
vagrant ssh web
```
Inside web, move the key:
```
mv /vagrant/db_key ~/.ssh/db_key
chmod 600 ~/.ssh/db_key
```
Test SSH from web to db
Find db's IP (ip a in db), then run from web:
```
    ssh -i ~/.ssh/db_key vagrant@192.168.56.102
```
    If successful, you're connected!

### 🔹 Step 4: Configure SSH Authorized Keys (Persistent Setup)

To avoid copying private keys manually, let's configure SSH keys properly.
### 1️⃣ Generate an SSH Key on web

In web, run:
```
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N ""
```
This creates:

    Private key: ~/.ssh/id_rsa
    Public key: ~/.ssh/id_rsa.pub

2️⃣ Copy the Public Key to db

Run:

vagrant ssh db
mkdir -p ~/.ssh
touch ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

Now, from web, copy the key:
```
cat ~/.ssh/id_rsa.pub | ssh vagrant@192.168.56.102 "cat >> ~/.ssh/authorized_keys"
```
### 3️⃣ Test Passwordless SSH

Now SSH should work without specifying the key:
```
ssh vagrant@192.168.56.102
```
You're in! 🎉
### 🔹 Step 5: Automate Key Setup (Vagrantfile)

Instead of manually copying SSH keys, automate it in Vagrantfile:
```
Vagrant.configure("2") do |config|
  config.vm.define "web" do |web|
    web.vm.box = "ubuntu/jammy64"
    web.vm.network "private_network", type: "dhcp"
    web.vm.provision "shell", inline: <<-SHELL
      ssh-keygen -t rsa -b 2048 -f /home/vagrant/.ssh/id_rsa -N ""
      echo "Copying SSH key to db..."
      sshpass -p "vagrant" ssh-copy-id -i /home/vagrant/.ssh/id_rsa.pub -o StrictHostKeyChecking=no vagrant@192.168.56.102
    SHELL
  end

  config.vm.define "db" do |db|
    db.vm.box = "ubuntu/jammy64"
    db.vm.network "private_network", type: "dhcp"
    db.vm.provision "shell", inline: <<-SHELL
      mkdir -p /home/vagrant/.ssh
      touch /home/vagrant/.ssh/authorized_keys
      chmod 700 /home/vagrant/.ssh
      chmod 600 /home/vagrant/.ssh/authorized_keys
    SHELL
  end
end
```
Now, after vagrant up, web can SSH into db automatically.
🎯 Final Verification

Test the connection from web:
```
ssh vagrant@192.168.56.102
```
If successful, you've fully configured passwordless SSH between VMs! 🚀

You want to automate a daily backup of a text file from Ubuntu OS 1 (web) to Ubuntu OS 2 (db) using ssh and scp. We'll achieve this using cron jobs.

### 🔹 Step 1: Ensure SSH Access

Since you already have SSH access between the two VMs (web → db), test scp manually:
```
scp /path/to/textfile.txt vagrant@192.168.56.102:/backup/
```
If this works without a password, then SSH is properly set up.
### 🔹 Step 2: Create a Backup Directory on db

On db, run:
```
mkdir -p /backup
chmod 777 /backup  # Optional: Give full access
```
### 🔹 Step 3: Add a Cron Job on web

Now, let's set up an automated cron job on web to run every day at 3:00 PM.
### 1️⃣ Open the Crontab Editor

On web, type:
```
crontab -e
```
If it's your first time, choose a text editor (select nano if unsure).
### 2️⃣ Add This Cron Job

Scroll to the bottom and add:
```
0 15 * * * scp /path/to/textfile.txt vagrant@192.168.56.102:/backup/ >> /home/vagrant/backup.log 2>&1
```
    0 15 * * * → Runs every day at 3:00 PM.
    Redirects output to a log file (backup.log).

3️⃣ Save and Exit

In nano, press CTRL + X, then Y, then ENTER.
### 🔹 Step 4: Verify the Cron Job

    Check if the cron job is scheduled:
```
crontab -l
```
You should see:

0 15 * * * scp /path/to/textfile.txt vagrant@192.168.56.102:/backup/ >> /home/vagrant/backup.log 2>&1

Manually test by running:
```
scp /path/to/textfile.txt vagrant@192.168.56.102:/backup/
```
Check the backup directory in db:
```
    ls /backup/
```
    You should see textfile.txt.

### 🔹 Step 5: Monitor Backup Logs

Check logs on web:
```
cat /home/vagrant/backup.log
```
If there are errors, update your cron job to use absolute paths:
```
0 15 * * * /usr/bin/scp /path/to/textfile.txt vagrant@192.168.56.102:/backup/ >> /home/vagrant/backup.log 2>&1
```
🎯 Bonus: Add a Timestamp to the Backup

To keep multiple backups, modify the cron job:
```
0 15 * * * scp /path/to/textfile.txt vagrant@192.168.56.102:/backup/textfile_$(date +\%Y-\%m-\%d_\%H-\%M).txt
```
Now, backups will be stored as:
```
textfile_2025-02-15_15-00.txt
```
✅ Summary

✔ Configured SSH & SCP for file transfer
✔ Created a cron job to run at 3:00 PM daily
✔ Verified backups with logs
✔ Added timestamped backups
چالش ها: اول ارتباط ssh بین دو ماشین که پیچیده بود کمی ولی شد. دومی دادن دسترسی کامل به یوزر vagrant بای اینکه بتواند فایل تکست را کپی کند.
