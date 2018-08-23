# mastodon-vps-tutorial
Detailed step-by-step instruction for setting up Mastodon on cheap VPS (Hetzner behind Cloudflare) from scratch.

## Environment

This instruction assumes that you want to run Mastodon on Hetzner cloud VM, using Hetzner storage box for media storage, without elasticsearch, behind Cloudflare, using let's encrypt certificate locally and only allowing requests over cloudflare

Without the loss of generality, the domain name is assumed to be `social.penartur`.

It also assumes that Mastodon version is 2.4.4. You can probably install newer 2.4.x versions with this instruction; however, major upgrades may change docker layout, and you'll need to use updated docker-compose.yml.

## Domain prerequisites

You should have a registered domain with cloudflare enabled and email configured.

I've registered my domain with gandi.net, created two mailboxes in gandi control panel (admin@social.penartur and robot@social.penartur), added domain to cloudflare, and in gandi control panel updated domain nameservers to cloudflare-supplied ones.

You're better to do that in advance, since it takes time for changes to propagate through DNS, and it is easier to do the rest once everything is ready with domain and DNS.

## Preparation

1. Create new small (2EUR) Ubuntu-based Hetzner cloud VM with backups in Falkenstein (to get low latency connection to storage boxes, which are all located in Falkenstein).

2. In Cloudflare, on DNS tab, edit `@` (or `social.penartur`) A record so that it points to your VM IP, with cloudflare protection (cloud icon) enabled (cloud should be yellow).

2. Reset root password, open console (Chrome recommended; in FF and Edge, some key combinations such as Ctrl+Shift+F2 or Ctrl+K, do not work).

3. `$ systemctl disable sshd`

4. `$ adduser YOURNAME`, `$ adduser YOURNAME sudo`, `$ exit`, login as YOURNAME.

5. `$ sudo apt-get update`, `$ sudo apt-get dist-upgrade`. If you'll be prompted about grub config merge conflict, just "install the package maintainer's version".

6. `$ sudo reboot`, `$ sudo apt-get update`, `$ sudo apt autoremove`

7. Create new Hetzner storage box. Log in via SCP (e.g. with WinSCP), create directory `mastodon`, create sub-directory `media`

8. In storage box settings, create new sub-account for `mastodon` directory.

9. `$ sudo ssh-keygen`

10. `$ sudo cat /root/.ssh/id_rsa.pub > storagebox_authorized_keys`

11. `$ ssh-keygen -e -f /root/.ssh/id_rsa.pub | grep -v "Comment:" >> storagebox_authorized_keys`

12. `$ sftp uXXXXXX-subY@uXXXXXX.your-storagebox.de`, enter password, execute:

    1. `sftp> mkdir .ssh`

    2. `sftp> chmod 700 .ssh`

    3. `sftp> put storagebox_authorized_keys .ssh/authorized_keys`

    4. `sftp> chmod 600 .ssh/authorized_keys`

    5. `sftp> quit`

13. Add storagebox to the root's list of known hosts and verify that sftp works with key authorization by doing `$ sudo sftp uXXXXXX-subY@uXXXXXX.your-storagebox.de`; it should not ask for password.

14. `$ sudo adduser mastodon`

## Prerequisites

1. `$ sudo apt-get install nginx sshfs certbot`, `sudo systemctl stop nginx`, `sudo systemctl disable nginx`

2. `$ sudo apt-get remove docker docker-engine docker.io` (just in case these were already installed)

3. `$ sudo apt-get install apt-transport-https ca-certificates curl software-properties-common` (just in case there weren't already installed)

4. `$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`; verify with `$ sudo apt-key fingerprint 0EBFCD88`

5. `$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable`, `sudo apt-get update`

6. `$ sudo apt-get install docker-ce`; verify with `sudo docker run hello-world`

7. `$ sudo adduser mastodon docker`

8. `$ sudo curl -L https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose`; `sudo chmod +x /usr/local/bin/docker-compose`. **NOTE** that this command refers to the **specific version** of docker-compose; you should refer to the actual latest version.

9. `$ sudo mkdir /mastodon`

## Mastodon installation

1. Copy the required config files (see `samples` directory) to `/mastodon/conf` directory. For example, copy them to storage box, and then:

    1. `$ sudo sftp uXXXXXX-subY@uXXXXXX.your-storagebox.de`

    2. `sftp> get mastodonconf/* /mastodon/conf`

2. Open new TTY (e.g. with Ctrl+Alt+F2), log in as `mastodon` (or `$ sudo su - mastodon`)

3. `$ mkdir /home/mastodon/docker-scratchpad`, `cd /home/mastodon/docker-scratchpad`

4. `$ cp /mastodon/conf/docker-compose.yml .`, `$ cp /mastodon/conf/.env.production .`

5. `$ script -f -c "docker-compose run --rm web bundle exec rake mastodon:setup" setup.log`

6. Enter your domain name (e.g. `social.penartur`)

7. Choose single/multi-user mode

8. Confirm that you're on Docker

9. Leave default postgresql and redis settings

10. Confirm that you don't want to store uploaded files on the cloud

11. Confirm that you don't want to send e-mails from localhost

12. SMTP server is `mail.gandi.net`, SMTP port is default (587), username is `robot@social.penartur`, password is your robot mailbox password, authentication is default (plain), verify mode is default (none), e-mail address is `Your Mastodon Name <robot@social.penartur>`

13. Agree to sending a test email; check that it has arrived (note that it may end up in a junk folder)

14. Confirm that you want to save configuration

15. Open new TTY (e.g. Ctrl+Shift+F3), log in as `mastodon`

16. Make sure that `docker-scratchpad/setup.log` contains the configuration (e.g. by `$ less docker-scratchpad/setup.log`). If it does not, return to TTY2, press some letters to force `script` to flush its buffers, and switch to TTY3 again

17. `cp docker-scratchpad/setup.log docker-scratchpad/.env.production`, `nano docker-scratchpad/.env.production`, remove redundant garbage (e.g. by Ctrl+K), leaving only config data (starting with `# Generated with mastodon:setup` and ending before `It is also saved within this container`). Save it (Ctrl+O, Ctrl+X)

18. Exit TTY3, return to TTY2

19. Confirm that you want to prepare database

20. Once database is prepared, switch to TTY1, `$ sudo chown -R 991:991 /mastodon/public`, switch back to TTY2

21. Confirm that you want to compile assets. Wait while assets compile (it should take around five minutes).

22. Once assets are compiled, switch to TTY1

23. `$ sudo chmod 000 /mastodon/public/system`

24. `$ sudo nano /etc/fstab`, add new line: `uXXXXXX-subY@uXXXXXX.your-storagebox.de:/media	/mastodon/public/system	fuse.sshfs	delay_connect,_netdev,user,idmap=user,transform_symlinks,identityfile=/root/.ssh/id_rsa,allow_other,reconnect,uid=991,gid=991	0	0`, save and exit (Ctrl+O, Ctrl+X)

25. `$ sudo mount /mastodon/public/system`. Make sure everything is correct (including 991:991 ownership) by `$ ls -la /mastodon/public/system`

26. Return to TTY2

27. Confirm that you want to create an admin user, with username `admin` and email `admin@social.penartur`

28. Remember admin password

29. `docker-compose up -d`

30. `mkdir /home/mastodon/live`, `mkdir /home/mastodon/live/public`

31. Exit TTY2, switch to TTY1

## Going public

1. In Cloudflare, on Crypto tab, switch SSL to Flexible, disable "Always use HTTPS", disable "Authenticated origin pulls".

2. `$ sudo systemctl stop nginx`

3. `$ sudo rm /etc/nginx/sites-enabled/default`

4. Copy cloudflare client certificate to `/etc/nginx/certs/cloudflare.crt`

5. Edit `penartur.social.conf` for your domain, renaming as appropriate, and put it to `/etc/nginx/sites/available`

6. `$ cd /etc/nginx/sites-enabled`, `$ sudo ln -s ../sites-available/social.penartur.conf`

7. `$ certbot certonly --standalone -d social.penartur`, enter your admin email address (admin@social.penartur)

8. Check that everything is OK by `$ sudo nginx -t`

9. In Cloudflare, on Crypto tab, switch SSL to Full (strict), enable "Always use HTTPS", enable "Authenticated origin pulls"

10. `$ sudo systemctl start nginx`

11. `$ sudo certbot certonly --webroot -d social.penartur -w /home/mastodon/live/public`

12. Create new script: `$ sudo nano /etc/cron.daily/letsencrypt-renew`:

    ```
    #!/usr/bin/env bash
    certbot renew
    systemctl reload nginx
    ```

## Finishing steps

1. Check that your mastodon instance (social.penartur) opens in browser

2. `$ sudo reboot` (just to make sure it survives reboots correctly)

3. Go to your mastodon instance, log in as admin, change password

4. Upload some media and make sure it went to storage box (media/media_attachments/files/...)

5. Follow someone from other instance, make sure that federation works

6. Set your VM to "protected mode" in hetzner cloud control panel, to make sure you don't accidentally delete it

7. Create a snapshot of your VM

8. Have some rest at last

## Support

If you experience shortage of CPU or RAM, you can always rescale your VM in hetzner control panel. Make sure to leave disk at its original volume (20GB) so that you'll be able to downscale later
