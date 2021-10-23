---
title: "Fedora 32: Simple Local File-Sharing with Samba"
layout: post
categories: posting update
post_desc: "Sharing files with Fedora 32 using Samba is cross-platform, convenient, reliable, and performant."
---

Sharing files with Fedora 32 using Samba is cross-platform, convenient, reliable, and performant.

## What is ‘Samba’?

[Samba](https://www.samba.org/samba/) is a high-quality implementation of [Server Message Block protocol (SMB)](https://en.wikipedia.org/wiki/Server_Message_Block). Originally developed by Microsoft for connecting windows computers together via local-area-networks, it is now extensively used for internal network communications.

Apple used to maintain it’s own independent file sharing called [“Apple Filing Protocol (AFP)“](https://en.wikipedia.org/wiki/Apple_Filing_Protocol), however in [recent times](https://appleinsider.com/articles/13/06/11/apple-shifts-from-afp-file-sharing-to-smb2-in-os-x-109-mavericks), it also has also switched to SMB.

In this guide we provide the minimal instructions to enable:

- Public Folder Sharing (Read Only and Read Write)
- User Home Folder Access

`Note about this guide: The convention '~]$' for a local user command prompt, and '~]#' for a super user prompt will be used.`

## Public Sharing Folder

Having a shared public place where authenticated users on an internal network can access files, or even modify and change files if they are given permission, can be very convenient. This part of the guide walks through the process of setting up a shared folder, ready for sharing with Samba.

`Please Note: This guide assumes the public sharing folder is on a Modern Linux Filesystem; other filesystems such as NTFS or FAT32 will not work. Samba uses POSIX Access Control Lists (ACLs).`

`For those who wish to learn more about Access Control Lists, please consider reading the documentation: "Red Hat Enterprise Linux 7: System Administrator's Guide: Chapter 5. Access Control Lists", as it likewise applies to Fedora 32.`

`In General, this is only an issue for anyone who wishes to share a drive or filesystem that was created outside of the normal Fedora Installation process. (such as a external hard drive).`

`It is possible for Samba to share filesystem paths that do not support POSIX ACLs, however this is out of the scope of this guide.`

### Create Folder

For this guide the /srv/public/ folder for sharing will be used.

`The /srv/ directory contains site-specific data served by a Red Hat Enterprise Linux system. This directory gives users the location of data files for a particular service, such as FTP, WWW, or CVS. Data that only pertains to a specific user should go in the /home/ directory.`

[`- Red Hat Enterprise Linux 7, Storage Administration Guide: Chapter 2. File System Structure and Maintenance: 2.1.1.8. The /srv/ Directory`](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/storage_administration_guide/ch-filesystem#s3-filesystem-srv)


>	# mkdir --verbose /srv/public

### Set Filesystem Security Context

To have read and write access to the public folder the public_content_rw_t security context will be used for this guide. Those wanting read only may use: public_content_t.

`Label files and directories that have been created with the public_content_rw_t type to share them with read and write permissions through vsftpd. Other services, such as Apache HTTP Server, Samba, and NFS, also have access to files labeled with this type. Remember that booleans for each service must be enabled before they can write to files labeled with this type.`

[`- Red Hat Enterprise Linux 7, SELinux User’s and Administrator’s Guide: Chapter 16. File Transfer Protocol: 16.1. Types: public_content_rw_t`](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/chap-managing_confined_services-file_transfer_protocol#sect-Managing_Confined_Services-File_Transfer_Protocol-Types)

Add /srv/public as “public_content_rw_t” in the system’s local filesystem security context customization registry:

Add new security filesystem security context:
	
>	# semanage fcontext --add --type public_content_rw_t "/srv/public(/.*)?"

Verifiy new security filesystem security context:
	
>	# semanage fcontext --locallist --list

Expected Output: (should include)
	
>	/srv/public(/.*)? all files system_u:object_r:public_content_rw_t:s0

Now that the folder has been added to the local system’s filesystem security context registry; The restorecon command can be used to ‘restore’ the context to the folder:

Restore security context ke the /srv/public folder:
	
>	# restorecon -Rv /srv/public

Verify security context was correctly applied:
	
>	$ ls --directory --context /srv/public/

Expected Output:
	
>	unconfined_u:object_r:public_content_rw_t:s0 /srv/public/

### User Permissions

### Creating the Sharing Groups

To allow a user to either have read only, or read and write accesses to the public share folder create two new groups that govern these privileges: public_readonly and public_readwrite.

User accounts can be granted access to read only, or read and write by adding their account to the respective group (and allow login via Samba creating a smb password). This process is demonstrated in the section: “Test Public Sharing (localhost)”.

Create the public_readonly and public_readwrite groups:
	
>	# groupadd public_readonly
	# groupadd public_readwrite

Verify successful creation of groups:
	
>	$ getent group public_readonly public_readwrite

Expected Output: (Note: x:1...: number will probability differ on your System)
	
>	public_readonly:x:1009:
	public_readwrite:x:1010:

### Set permissions

Sekarang kita akan mengatur izin akses user di public share folder yang kita telah buat.

Set User and Group Permissions for Folder:
	
>	# chmod --verbose 2700 /srv/public
	# setfacl -m group:public_readonly:r-x /srv/public
	# setfacl -m default:group:public_readonly:r-x /srv/public
	# setfacl -m group:public_readwrite:rwx /srv/public
	# setfacl -m default:group:public_readwrite:rwx /srv/public

Melihat user permission yang telah dibuat:
	
>	$ getfacl --absolute-names /srv/public

Akan terihat hasilnya:
	
>	file: /srv/public
	owner: root
	group: root
	flags: -s-
	user::rwx
	group::---
	group:public_readonly:r-x
	group:public_readwrite:rwx
	mask::rwx
	other::---
	default:user::rwx
	default:group::---
	default:group:public_readonly:r-x
	default:group:public_readwrite:rwx
	default:mask::rwx
	default:other::---

## Samba

### Installation

>	# dnf install samba

### Hostname (systemwide)

Samba akan menggunakan nama komputer saat berbagi file, ada baiknya untuk mengatur nama host agar komputer dapat ditemukan dengan mudah di jaringan lokal.

>	Melihat hostname kita saat ini

>	$ hostnamectl status

Jika kita hendak merubah nama dari  hostname kita, berikut ini perintahnya:

>	# hostnamectl set-hostname "simple-samba-server"

### Firewall

Mengkonfigurasi firewall tidak lah mudah, berikut ini cara sederhana yang mungkin bisa dilakukan.

Mengizinkan akses samba melalui firewall:

>	# firewall-cmd --add-service=samba --permanent
	# firewall-cmd --reload

Melihat apakah service samba sudah aktif di firewall kita:

>	$ firewall-cmd --list-services

Dan outputnya adalah:

>	samba

### Configuration

#### Menghapus Default Configuration

Untuk configurasi sederhana ini kita tidak memerlukan default configurasi fedora system, maka kita akan coba membackup nya dan membuat configurasi baru 

>	cp --verbose --no-clobber /etc/samba/smb.conf /etc/samba/smb.conf.fedora0

Mengkosongkan configuration file:

>	# > /etc/samba/smb.conf

#### Samba Configuration

Mengedit file configurasi samba dengan Vim:

>	# vim /etc/samba/smb.conf

Tambahkan baris perintah berikut kedalam file /etc/samba/smb.conf

>	# smb.conf - Samba Configuration File

>	# The name of the share is in square brackets [],
	# this will be shared as //hostname/sharename
	# There are a three exceptions:
	#   the [global] section;
	#   the [homes] section, that is dynamically set to the username;
	#   the [printers] section, same as [homes], but for printers.

>	# path: the physical filesystem path (or device)
	# comment: a label on the share, seen on the network.
	# read only: disable writing, defaults to true.

>	# For a full list of configuration options,
	#   please read the manual: "man smb.conf".

>	[global]

>	[public]
	path = /srv/public
	comment = Public Folder
	read only = No

### Write Permission (Ijin akses Tulis pada Folder)

Secara default, Samba tidak diberikan izin untuk memodifikasi file apa pun dari sistem. Ubah konfigurasi keamanan sistem untuk memungkinkan Samba memodifikasi jalur sistem file apa pun yang memiliki security contex "public_content_rw_t."

Untuk kenyamanan, Fedora memiliki SELinux Boolean bawaan untuk tujuan ini yang disebut: smbd_anon_write, set nilai ini ke true akan memungkinkan Samba untuk menulis di jalur sistem file apa pun yang telah disetel ke security contex public_content_rw_t.

Bagi mereka yang menginginkan Samba hanya memiliki akses baca-saja di share folder publik mereka, mereka boleh mengabaikan setelan ini.

Set SELinux Boolean agar mengijinkan samba untuk menulis ke filesystem dengan security context public_content_rw_t:

>	# setsebool -P smbd_anon_write=1

Cek nilai yang baru saja kita set:

>	$ getsebool smbd_anon_write

seharusnya akan menampilkan hasil:

$ getsebool smbd_anon_write

## Samba Services

Samba Services dibagi menjadi dua bagian yang harus kita hidupkan.

### Samba ‘smb’ Service

Layanan "Server Message Block" (SMB) Samba adalah untuk berbagi file dan printer melalui jaringan lokal.

### Enable and Start Services

Enable and start smb and nmb services:

>	# systemctl enable smb.service
	# systemctl start smb.service

Cek smb service

>	# systemctl start smb.service

### Test Public Sharing (localhost)

Untuk mengizinkan dan menghapus akses ke dalam sharing folder publik, buat pengguna baru bernama samba_test_user, pengguna ini akan diberikan izin terlebih dahulu untuk membaca folder publik, lalu akses untuk membaca dan menulis folder publik.

Proses yang sama yang ditunjukkan di sini dapat digunakan untuk memberikan akses ke share folder publik Anda kepada pengguna lain dari komputer Anda.

Samba_test_user akan dibuat sebagai akun pengguna yang sudah di lock, tidak mengizinkan login ke komputer.

Create 'samba_test_user', and lock the account.

>	# useradd samba_test_user
	# passwd --lock samba_test_user

Set a Samba Password for this Test User (such as 'test'):

>	# smbpasswd -a samba_test_user

#### Test Read Only access to the Public Share:

Add samba_test_user to the public_readonly group:

>	# gpasswd --add samba_test_user public_readonly

Login to the local Samba Service (public folder):

>	$ smbclient --user=samba_test_user //localhost/public

First, the ls command should succeed,
Second, the mkdir command should not work,
and finally, exit:

>	smb: \> ls
	smb: \> mkdir error
	smb: \> mkdir error

Remove samba_test_user from the public_readonly group:

>	gpasswd --delete samba_test_user public_readonly

#### Test Read and Write access to the Public Share:

Add samba_test_user to the public_readwrite group:

>	# gpasswd --add samba_test_user public_readwrite

Login to the local Samba Service (public folder):

>	$ smbclient --user=samba_test_user //localhost/public

First, the ls command should succeed,
Second, the mkdir command should work,
Third, the rmdir command should work,
and finally, exit:

>	smb: \> ls
	smb: \> mkdir success
	smb: \> rmdir success
	smb: \> exit

Remove samba_test_user from the public_readwrite group:

>	# gpasswd --delete samba_test_user public_readwrite

Setelah pengujian selesai, untuk keamanan, nonaktifkan kemampuan samba_test_user untuk login melalui samba.

Disable samba_test_user login via samba:

>	# smbpasswd -d samba_test_user

## Home Folder Sharing

Di bagian terakhir panduan ini; Samba akan dikonfigurasi untuk berbagi User Home Folder.

Misalnya: Jika bob pengguna telah terdaftar pada smbpasswd, direktori home bob /home/bob, akan menjadi share //server-name/bob.

Share ini hanya akan tersedia untuk bob, dan tidak di pengguna lain.

### Setup Home Folder Sharing

Set SELinux Boolean allowing Samba to read and write to home folders:

>	# setsebool -P samba_enable_home_dirs=1

# setsebool -P samba_enable_home_dirs=1

>	# setsebool -P samba_enable_home_dirs=1

Expected Output:

>	samba_enable_home_dirs --> on

#### Add Home Sharing to the Samba Configuration

Append the following to the systems smb.conf file:

>	# The home folder dynamically links to the user home.
	# If 'bob' user uses Samba:
	# The homes section is used as the template for a new virtual share:

>	# [homes]
	# ...   (various options)

>	# A virtual section for 'bob' is made:
	# Share is modified: [homes] -> [bob]
	# Path is added: path = /home/bob
	# Any option within the [homes] section is appended.

>	# [bob]
	#       path = /home/bob
	# ...   (copy of various options)

>	# here is our share,
	# same as is included in the Fedora default configuration.

>	[homes]
        comment = Home Directories
        valid users = %S, %D%w%S
        browseable = No
        read only = No
        inherit acls = Yes
	

#### Reload Samba Configuration

Tell Samba to reload it's configuration:

>	# smbcontrol all reload-config

### Test Home Directory Sharing

Switch to samba_test_user and create a folder in it's home directory:

>	# su samba_test_user
	samba_test_user:~]$ cd ~
	samba_test_user:~]$ mkdir --verbose test_folder
	samba_test_user:~]$ exit

Enable samba_test_user to login via Samba:

>	# smbpasswd -e samba_test_user

Login to the local Samba Service (samba_test_user home folder):

>	$ smbclient --user=samba_test_user //localhost/samba_test_user

Test (all commands should complete without error):

>	smb: \> ls
	smb: \> ls test_folder
	smb: \> rmdir test_folder
	smb: \> mkdir home_success
	smb: \> rmdir home_success
	smb: \> exit

Disable samba_test_user from login in via Samba:

>	# smbpasswd -d samba_test_user




`source: http://fedoramagazine.org/fedora-32-simple-local-file-sharing-with-samba/`



