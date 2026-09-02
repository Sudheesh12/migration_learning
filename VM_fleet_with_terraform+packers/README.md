## Part 1 — Set up the AWS "host" instance

1. **Launch the instance**
   - AMI: Windows Server 2022 (needed for Hyper-V role)
   - Instance type: `m8i.xlarge` (4 vCPU/16GB) or `c8i.xlarge` — must be one of the nested-virtualization-supported types (C8i, M8i, R8i, C8id, R8id, M8id, C8i-flex, R8i-flex, M8i-flex, X8i, C7i, R7i, M7i, C7id, R7id, M7id, C7i-flex, R7i-flex, M7i-flex, I7i)
   - In **Advanced details**, set **Nested virtualization → Enable**
   - Storage: bump to at least 150GB (you'll need room for 2-3 nested VM disks)

2. **Connect via RDP** and confirm nested virtualization is on:
   ```powershell
   Get-VM  # should run without error once Hyper-V is installed (next step)
   ```

3. **Install the Hyper-V role**
   ```powershell
   Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart
   ```
   Instance reboots automatically.

4. **Create a virtual switch** (so nested VMs get network access)
   ```powershell
   New-VMSwitch -Name "OnPremSwitch" -NetAdapterName "Ethernet" -AllowManagementOS $true
   ```

### Budget note
This instance type is not free-tier eligible and Windows licensing adds cost on top of compute. Estimate roughly $0.25–0.40/hr depending on region — so **stop the instance** (not just RDP disconnect) whenever you're not actively working. Don't leave it running overnight. Set an AWS Budget alert at $30–40 given your $200 total.

## Part 2 — Build the on-prem VMs inside Hyper-V

You'll create 2 VMs: an **app server** and a **DB server**, both Ubuntu Server (lighter than Windows guests, easier to script later).

1. **Download Ubuntu Server ISO** to the Windows instance (e.g., Ubuntu Server 24.04 LTS) — use a browser inside the RDP session.

2. **Create the App Server VM**
   ```powershell
   New-VM -Name "AppServer" -MemoryStartupBytes 4GB -Generation 2 -NewVHDPath "C:\VMs\AppServer.vhdx" -NewVHDSizeBytes 40GB -SwitchName "OnPremSwitch"
   Set-VMProcessor -VMName "AppServer" -Count 2
   Set-VMDvdDrive -VMName "AppServer" -Path "C:\ISOs\ubuntu-24.04-server.iso"
   Set-VMFirmware -VMName "AppServer" -EnableSecureBoot Off  # needed for Linux Gen2 VMs
   Start-VM -Name "AppServer"
   ```
   Connect via `vmconnect` in Hyper-V Manager and run through the Ubuntu installer normally.

3. **Create the DB Server VM** — repeat with `-Name "DBServer"`, similar specs.

4. **Install the app stack on AppServer** (once Ubuntu is installed):
   ```bash
   sudo apt update && sudo apt install -y apache2 php libapache2-mod-php php-mysql
   sudo systemctl enable apache2 --now
   ```
   Drop a simple PHP page in `/var/www/html/index.php` so you have something visibly "running" to migrate later.

5. **Install the DB stack on DBServer**:
   ```bash
   sudo apt update && sudo apt install -y mysql-server
   sudo mysql_secure_installation
   sudo mysql -e "CREATE DATABASE appdb; CREATE USER 'appuser'@'%' IDENTIFIED BY 'ChangeMe123!'; GRANT ALL ON appdb.* TO 'appuser'@'%'; FLUSH PRIVILEGES;"
   ```
   Load some sample data — even a simple `products` or `users` table with a few hundred rows via a `.sql` seed script — so your later migration has something real to verify (row counts, checksums).

6. **Connect the two**: point AppServer's PHP config at DBServer's internal IP, confirm the app can actually query the DB. This "working app" is your migration source of truth — post-cutover, you'll re-test the same functionality against AWS.

7. **Document the environment** (this is the discovery phase real migration engineers do first): OS versions, package versions, open ports (`sudo ss -tulpn`), resource usage baseline. Write it into a `discovery.md` — you'll reference it when sizing the AWS targets.

## Part 3 — Packer, briefly (for when you're ready)

Packer's job: take a base image and bake your app/DB configuration into it, producing a versioned AMI — so instead of manually installing Apache/MySQL on AWS, your target EC2 instances boot up already configured, identical to what you built by hand above.

Minimal shape of a Packer template (`app-server.pkr.hcl`):
```hcl
source "amazon-ebs" "app" {
  ami_name      = "app-server-{{timestamp}}"
  instance_type = "t3.micro"
  region        = "us-east-1"
  source_ami_filter {
    filters = { name = "ubuntu/images/*24.04*" }
    owners  = ["099720109477"]
    most_recent = true
  }
  ssh_username = "ubuntu"
}

build {
  sources = ["source.amazon-ebs.app"]
  provisioner "shell" {
    inline = [
      "sudo apt update",
      "sudo apt install -y apache2 php libapache2-mod-php php-mysql"
    ]
  }
}
```
Run with `packer init .` then `packer build app-server.pkr.hcl`. That's genuinely most of what you need to get a first image built — we'll go deeper (variables, Ansible provisioners, versioning) once your on-prem source is live and you're ready to bake it.

Want to start executing Part 1 (AWS instance launch) now, or do you want me to also write out the DB seed script and sample PHP app first so you have real content to migrate?