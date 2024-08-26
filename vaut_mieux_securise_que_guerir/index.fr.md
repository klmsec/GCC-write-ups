---
author: K.L.M
title: 404CTF - Vaut mieux sécuriser que guérir
description: Write-Up du challenge de forensic du 404 CTF 2024
slug: vaut-mieux-securise-que-guerir-forensic                   
date: 2023-12-31 00:00:00+0000
image: assets/cover.png
categories:
    - Forensics
tags:
    - 404CTF
    - Forensics
    - K.L.M
---

# 404CTF - Vaut mieux sécuriser que guérir

---
## Description
Lors d'une compétition de lancer de poids, un athlète laisse son ordinateur allumé avec sa session ouverte. Cependant, une personne a utilisé son ordinateur et, a vraisemblablement fait des cachotteries. Nous vous mettons à disposition le dump de la RAM de l'ordinateur après l'incident.
Investiguez ce dump mémoire pour comprendre ce qu'il s'est passé.

La deuxième partie du flag est le nom d'une certaine tâche.
Les deux parties sont séparées d'un tiret "-". Par exemple si le flag de la première partie est "flag1" et celui de la deuxième partie est "flag2". Le réel flag du challenge sera 404CTF{flag1-flag2}

---
## Premiers pas 
Après avoir dézippé le dump, j'ai utilisé `volatility` afin de commencer une reconnaissance basique.

#### Les Process
Rien de très fameux, on note que l'explorer est ouvert et PowerShell a été lancé :
```Volatility Foundation Volatility Framework 2.6.1
Name                                                  Pid   PPid   Thds   Hnds Time
-------------------------------------------------- ------ ------ ------ ------ ----
0xffffd50ebabff080:winlogon.exe                      516    444      3      0 2024-03-12 09:57:36 UTC+0000
. 0xffffd50ebb026580:fontdrvhost.ex                   680    516      5      0 2024-03-12 09:57:37 UTC+0000
. 0xffffd50ebb10c080:dwm.exe                          860    516     13      0 2024-03-12 09:57:37 UTC+0000
. 0xffffd50ebb93e580:userinit.exe                    2968    516      0 ------ 2024-03-12 08:58:11 UTC+0000
.. 0xffffd50ebb940580:explorer.exe                   2528   2968     61      0 2024-03-12 08:58:11 UTC+0000
... 0xffffd50ebba91580:unregmp2.exe                  3668   2528      0 ------ 2024-03-12 08:58:15 UTC+0000
... 0xffffd50eb9b7a580:OneDriveSetup.                4736   2528      0 ------ 2024-03-12 09:00:13 UTC+0000
... 0xffffd50ebb9f4580:ie4uinit.exe                  3452   2528      0 ------ 2024-03-12 08:58:14 UTC+0000
... 0xffffd50eb9707080:powershell.exe                4852   2528     13      0 2024-03-12 09:07:46 UTC+0000
.... 0xffffd50ebbd88580:conhost.exe                  4544   4852      4      0 2024-03-12 09:07:46 UTC+0000
... 0xffffd50eba11d580:fsquirt.exe                   4612   2528      0 ------ 2024-03-12 09:00:02 UTC+0000
... 0xffffd50ebb9f1580:unregmp2.exe                  3432   2528      0 ------ 2024-03-12 08:58:14 UTC+0000
... 0xffffd50eb9aef580:MSASCuiL.exe                  5512   2528      1      0 2024-03-12 09:00:12 UTC+0000
... 0xffffd50ebb972300:FirstLogonAnim                1012   2528      0 ------ 2024-03-12 08:58:11 UTC+0000
```

Je m'en vais donc chercher le fichier suivant : "ConsoleHost_History.txt"
```
klm@KLM:~/CTFs/40424$ python3 /home/klm/Utils/Forensic/volatility3/vol.py -f memory.dmp windows.filescan | grep ConsoleHost_history
0xd50ebb98a080.0\Users\Maison\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt   216

klm@KLM:~/CTFs/40424$ python3 /home/klm/Utils/Forensic/volatility3/vol.py -f memory.dmp windows.dumpfiles --virtaddr 0xd50ebb98a080
Volatility 3 Framework 2.5.0
Progress:  100.00               PDB scanning finished
Cache   FileObject      FileName        Result

DataSectionObject       0xd50ebb98a080  ConsoleHost_history.txt file.0xd50ebb98a080.0xd50eb945d010.DataSectionObject.ConsoleHost_history.txt.dat

klm@KLM:~/CTFs/40424$ cat file.0xd50ebb98a080.0xd50eb945d010.DataSectionObject.ConsoleHost_history.txt.dat
rm hacked.ps1
```
#### Tiens tiens tiens.
Petit strings des familles :
```
klm@KLM:~/CTFs/40424$ strings memory.dmp | grep -Fi "hacked.ps1"
"C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" .\hacked.ps1
\hacked.ps1
powershell .\hacked.ps1
rm hacked.ps1
wsPowerShell\v1.0\powershell.exe" .\hacked.ps1
GET /hacked.ps1 HTTP/1.1
"C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" .\hacked.ps1
"C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" .\hacked.ps1
```
Ce qui nous intéresse ici c'est le `GET /hacked.ps1 HTTP/1.1`

#### Récupérer un fichier supprimé ?
J'ai passé une petite heure à essayer de récupérer le script sans succès. En passant par la table MFT, scrollant à l'infini dans HxD sans rien trouver jusqu'à ce que je me demande si c'était possible de récupérer les paquets HTTP du dump.

J'ai donc trouvé un super plugin : https://github.com/Memoryforensics/carve_packets

```
klm@KLM:~/CTFs/40424$ python2.7 /home/klm/Utils/Forensic/volatility/vol.py  --plugins=/home/klm/CTFs/40424 -f memory.dmp --profile=Win10x64_17134 networkpackets --dump-dir .
Volatility Foundation Volatility Framework 2.6.1
*** Failed to import volatility.plugins.crypto.solvecrypto1 (ImportError: No module named crypto.solvecrypto1)

Network analysis of carved packets:

Number of carved packets
------------------------
514


Potential local interfaces used by analyzed system
--------------------------------------------------
10.0.2.15 with mac No ARP resolves seen


Potential gateways used by analyzed system
------------------------------------------
No ARP resolves <-> 52:54:00:12:35:02


ARP resolves                           Number of packets
-------------------------------------- -----------------
10.43.0.1 is at 26:ea:19:79:96:00                     18
10.42.13.116 is at f4:7b:09:0b:a4:00                   2
10.42.25.179 is at c8:cb:9e:7a:5a:49                   2
10.42.39.245 is at 54:14:f3:c6:89:f0                   2
10.42.34.244 is at 8c:55:4a:1f:73:ef                   1
10.42.18.210 is at c4:bd:e5:a2:49:28                   1
10.43.0.13 is at 08:00:27:a1:e0:61                     1


Public IP src or dst Number of packets Source ports < 1024 Destination ports < 1024  Total bytes Minimum packet size Maximum packet size Average packet size
-------------------- ----------------- ------------------- ------------------------ ------------ ------------------- ------------------- -------------------
2.21.35.225                        210                 443                                 27878                  40                 674                 132
2.21.35.217                        100                 443                                121301                  40                1500                1213
20.111.58.202                       32                 443                                  8269                  40                1500                 258
88.221.83.184                       28                 443                                  6315                  40                1500                 225
20.103.156.88                       18                 443                                  7391                  40                1500                 410
20.234.120.54                       16                 443                                  6945                  40                1500                 434
2.21.35.241                          9                 443                                   453                  40                  71                  50
8.8.8.8                              3                  53                                   569                 174                 203                 189
172.64.149.23                        2                  80                                    80                  40                  40                  40
152.199.19.161                       2                 443                                    80                  40                  40                  40
192.229.221.95                       2                  80                                    80                  40                  40                  40
104.18.38.233                        2                  80                                    80                  40                  40                  40
2.21.225.223                         2                  80                                    80                  40                  40                  40
52.113.194.132                       1                 443                                    40                  40                  40                  40
```
On obtient alors un fichier appelé `packets.pcap`, on sort les objets HTTP directement sans même regarder et on tombe sur :
```############################################################################################################################################################                      
#                                  |  ___                           _           _              _             #              ,d88b.d88b                     #                                 
# Title        : Wallpaper-Troll   | |_ _|   __ _   _ __ ___       | |   __ _  | | __   ___   | |__    _   _ #              88888888888                    #           
# Author       : I am Jakoby       |  | |   / _` | | '_ ` _ \   _  | |  / _` | | |/ /  / _ \  | '_ \  | | | |#              `Y8888888Y'                    #           
# Version      : 1.0               |  | |  | (_| | | | | | | | | |_| | | (_| | |   <  | (_) | | |_) | | |_| |#               `Y888Y'                       #
# Category     : Prank             | |___|  \__,_| |_| |_| |_|  \___/   \__,_| |_|\_\  \___/  |_.__/   \__, |#                 `Y'                         #
# Target       : Windows 10,11     |                                                                   |___/ #           /\/|_      __/\\                  #     
# Mode         : HID               |                                                           |\__/,|   (`\ #          /    -\    /-   ~\                 #             
#                                  |  My crime is that of curiosity                            |_ _  |.--.) )#          \    = Y =T_ =   /                 #      
#                                  |   and yea curiosity killed the cat                        ( T   )     / #   Luther  )==*(`     `) ~ \   Hobo          #                                                                                              
#                                  |    but satisfaction brought him back                     (((^_(((/(((_/ #          /     \     /     \                #    
#__________________________________|_________________________________________________________________________#          |     |     ) ~   (                #
#                                                                                                            #         /       \   /     ~ \               #
#  github.com/I-Am-Jakoby                                                                                    #         \       /   \~     ~/               #         
#  twitter.com/I_Am_Jakoby                                                                                   #   /\_/\_/\__  _/_/\_/\__~__/_/\_/\_/\_/\_/\_#                     
#  instagram.com/i_am_jakoby                                                                                 #  |  |  |  | ) ) |  |  | ((  |  |  |  |  |  |#              
#  youtube.com/c/IamJakoby                                                                                   #  |  |  |  |( (  |  |  |  \\ |  |  |  |  |  |#
############################################################################################################################################################
```
On cherche un peu et on retrouver le nom de la tâche (A.K.A Flag2) :
`Register-ScheduledTask -Action $Action -Trigger $Trigger -TaskName "XXX"
`
Puis je lis le code jusqu'à identifier du base 64 dans un champ, je décode :
```
klm@KLM:~/CTFs/40424$ echo "e1ByQG5rM2Qt" | base64 -d
{XXXXXX
```

On a donc désormais le flag complet : `404CTF{XXXXXXXXX}`

Super chall, j'ai découvert pas mal de nouvelles choses, j'avoue que je suis plutôt de niveau moyen en forensique.

~K.L.M

