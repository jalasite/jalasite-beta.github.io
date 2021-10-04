---
title: "Simple Local File - Sharing with samba"
layout: post
categories: posting update
post_desc: "Langkah sharing file dengan samba pada fedora 34"
---

## Samba ?

Samba adalah sebuah protokol Server Message Block (SMB). Awalnya dikembangkan oleh Microsoft untuk menghubungkan komputer windows bersama-sama melalui jaringan area lokal, sekarang banyak digunakan untuk komunikasi jaringan internal.

Apple juga mempunyai protokol sharing file sendiri yang disebut "Apple Filing Protocol (AFP)", namun belakangan ini, juga telah beralih ke SMB.

Dalam panduan ini kita akan mencoba configurasi sederhana untuk mengaktifkan:
- Public Folder Sharing (Read Only and Read Write)
- User Home Folder Access

Catatan:

"$  untuk prompt perintah pengguna lokal"  
"\# untuk prompt pengguna super"

## Public Sharing Folder

### Create Folder

Untuk panduan ini  kita akan menggunakan folder "/srv/public/".

>	# mkdir --verbose /srv/public

### Set Filesystem Security Context

Untuk dapat melakukan proses baca dan tulis ke folder publik, kita akan membuat context security "public_content_rw_t". Mereka yang hanya ingin membaca saja dapat menggunakan: "public_content_t".

Tambahkan security filesystem security context:
	
>	# semanage fcontext --add --type public_content_rw_t "/srv/public(/.*)?"

Lihat security filesystem security context yang telah dibuat:
	
>	# semanage fcontext --locallist --list

Akan terlihat seperti dibawah ini
	
>	/srv/public(/.*)? all files system_u:object_r:public_content_rw_t:s0

Sekarang folder telah ditambahkan ke security context registry sistem file lokal. Perintah restorecon dapat digunakan untuk 'mengembalikan' context ke folder:

Restore security context ke the /srv/public folder:
	
>	# restorecon -Rv /srv/public

Lihat security context yang telah dibuat:
	
>	$ ls --directory --context /srv/public/

Akan terlihat seperti dibawah ini
	
>	unconfined_u:object_r:public_content_rw_t:s0 /srv/public/

### User Permissions

### Creating the Sharing Groups

Untuk mengizinkan pengguna memiliki hak akses baca saja, atau baca dan tulis ke sharing folder, buatlah dua grup baru yang mengatur akses nya: "public_readonly dan public_readwrite".

User account dapat diberikan akses untuk sekedar membaca saja, atau membaca dan menulis dengan menambahkan akun mereka ke grup masing-masing.

Membuat group public_readonly and public_readwrite:
	
>	# groupadd public_readonly
	# groupadd public_readwrite

Melihat group yang kita telah buat:
	
>	$ getent group public_readonly public_readwrite

Akan terlihat hasil nya sbb:
	
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

...tobe continued ...

`source: http://fedoramagazine.org/fedora-32-simple-local-file-sharing-with-samba/`



