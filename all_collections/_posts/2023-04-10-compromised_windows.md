---
layout: post
title: Compromised Windows
date: 2023-04-10 14:20:00
categories: [writeup, windows]
---

# Deskripsi
Post ini berisi writeup challange tryhackme [Investigating Windows](https://tryhackme.com/room/investigatingwindows). Pada challange tersebut, diberikan akses ke suatu compromised Windows machine. Tujuan dari challange ini adalah untuk analisis proses yang terjadi saat machine tersebut ter-compromise.

# OS Version
Langkah pertama yang dilakukan adalah mendapatkan informasi tentang versi OS yang akan di analisis, hal ini dapat berguna untuk kebutuhan analisis nantinya. Untuk OS Windows, informasi ini terdapat pada `Start > Setting > System > About`
![alt](/blog/pic/investigating_windows/1.png)

# Malicious User
Dengan menggunakan command `Get-LocalUser` pada Powershell, kita bisa mendapatkan informasi mengenai user - user yang ada. Untuk memilih field yang ingin ditampilkan, tambahkan command `Select`.
![alt](/blog/pic/investigating_windows/1a.png)
Dapat dilihat bahwa terdapat 5 user dengan informasi terakhir login pada user **Administrator** dan **John**.
Karena kita masuk sebagai **Administrator**, maka user **John** patut dicurigai. Dapatkan informasi mengenai **John** dengan `net user`.
![alt](/blog/pic/investigating_windows/2.png)

Menampilkan user yang ada dalam group **Adminsitrator** dengan `net localgroup`.
![alt](/blog/pic/investigating_windows/4.png)

# Malicious Task
Saat menjalankan machine, terdapat popup berupa window terminal yang muncul secara terus-menerus. Setelah memeriksa Task Scheduler, ditemukan malicious task bernama "Clean file system" yang mana dijalankan setiap jam 4:55 PM.
Command tersebut terlihat seperti command tools [netcat](https://nmap.org/ncat/) dengan option `-l 1348` untuk melakukan listening pada port 1348.
![alt](/blog/pic/investigating_windows/9.png)
Ditemukan juga task "Game Over" yang mana akan berjalan setiap 5 menit sekali. Task tersebut mengeksekusi modul `sekurlsa` dari tools [Mimikatz](https://github.com/ParrotSec/mimikatz) yang berfungsi untuk dump password dari memory.
![alt](/blog/pic/investigating_windows/7.png)

# Malicious connection
Setelah melakukan pengecekan pada rules firewall, terlihat rule yang menerima traffic any menuju local port 1337 yang mana berhubungan dengan task "Clean file system" pada Task Scheduler.
![alt](/blog/pic/investigating_windows/16.png)

# Malicious File
Pada task scheduler terlihat banyak file yang dijalankan melalui folder `C:\TMP\` dan setelah diperiksa terlihat bahwa memang folder tersebut berisi banyak file yang malicious.
![alt](/blog/pic/investigating_windows/8.png)
Terindikasi penggunaan [Mimikatz](https://github.com/ParrotSec/mimikatz) pada salah satu file yang menunjukan adanya hasil dari credential dumping.
![alt](/blog/pic/investigating_windows/11.png)
Terdapat juga [DNS Spoofing](https://en.wikipedia.org/wiki/DNS_spoofing) dengan merubah IP dari DNS google.com dan virustotal.com pada file path `C:\Windows\System32\drivers\etc\hosts`
![alt](/blog/pic/investigating_windows/13.png)

# Inetpub Folder
Setelah melihat pada folder [C:\inetpub\wwwroot](https://stackify.com/what-is-inetpub/), terlihat bahwa machine yang dianalisa menjalankan web server. Web tersebut berfungsi untuk file editor.
![alt](/blog/pic/investigating_windows/wwwroot.png)
Terdapat kemungkinan bahwa Threat actor memanfaatkan fitur upload file pada web tersebut untuk upload file malicious karena terdapat file malicious pada folder web seperti file `shell.gif` dan `test.jsp`.
![alt](/blog/pic/investigating_windows/14.png)
![alt](/blog/pic/investigating_windows/15.png)

# Kesimpulan
Berikut adalah beberapa point yang dapat dijadikan kesimpulan mengenai apa yang dilakukan Threat Actor pada compromised Windows Machine.
- Initial Access dari Threat Actor adalah dengan Exploit Public-Facing Application melalui fitur upload file dari webserver. 
- Threat Actor berhasil mendapatkan akses dan melakukan privilege escalation sehingga mendapatkan privilege Administrator.
- Untuk mempertahankan presistence, Threat Actor melakukan beberapa teknik seperti DNS Spoofing, merubah settingan firewall, dan menggunakan Scheduled Task.
- Terdapat aktivitas credential dumping menggunakan Mimikatz yang menunjukan adanya aktivitas Lateral Movement.
