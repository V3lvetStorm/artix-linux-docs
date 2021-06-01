# Installazione di Artix Linux

Guida non ufficiale redatta e tradotta da [Naomi Calabretta](https://github.com/AryToNeX).

Ultimo aggiornamento: 1 Giugno 2021

# Introduzione

Benvenuto, o benvenuta, in questo tutorial di installazione di Artix Linux!

Artix Linux è una distribuzione di Linux basata su Arch Linux, ma con una differenza molto importante: essa, infatti, non usa Systemd come sistema di init.

Questo è molto importante per chi crede che un sistema di init debba essere semplice, affidabile e debba fare **solo** da sistema di init; d'altro canto Systemd è complesso e le sue componenti interne svolgono svariati ruoli nella gestione di un sistema Linux, il che potrebbe aumentare la superficie d'attacco per eventuali malintenzionati che vogliono avere accesso al sistema.

# Preparazione

## Scaricamento della Live ISO e preparazione USB

Vai su artixlinux.org e scarica la ISO `artixlinux-base`.

Scrivi la ISO appena scaricata su un supporto (USB/CD/DVD):

Per scrivere su chiavette USB, questa guida consiglia l'uso di [balenaEtcher](https://www.balena.io/etcher/) o [ROSA ImageWriter](http://wiki.rosalab.ru/en/index.php/ROSA_ImageWriter).

## Boot della live ISO

Ricorda di disabilitare il Secure Boot per installare e usare la distro.

Scegli il layout tastiera e il fuso orario appropriati, poi boota nella live ISO con l'opzione corretta.

## Login

Sulla live ISO hai due scelte per quanto riguarda il login:
- User `root`, password `artix`
- User `artix`, password `artix`: tutti i comandi importanti andranno prefissati con `sudo`.

## Primi comandi

Controlla che internet vada con un banale `ping`:
```
$ ping 1.1.1.1
```

# Installazione

## Partizionamento del disco creazione del file system

Bisogna capire qual è il disco su cui vuoi installare Artix. `blkid` è una utility che lista i dischi disponibili, e usualmente una sua invocazione è abbastanza per capire su che disco devi lavorare.

Per semplicità questa guida assumerà che il disco sia `/dev/sda`.

- Se devi reinizializzare il disco in questione, il comando da dare è:
  ```
  # cfdisk -z /dev/sda
  ```
- se, invece, devi solo modificare la tabella partizioni di un disco esistente (ad es. dual boot), il comando da dare è:
  ```
  # cfdisk /dev/sda
  ```

Ti si aprirà il gestore partizioni `cfdisk`.
Se stai creando adesso la tabella partizioni e sei su UEFI o su un BIOS Legacy che supporta il boot da dischi GPT, scegli `gpt`; altrimenti scegli `msdos`.

Crea le seguenti partizioni:

- Se sei su BIOS Legacy e hai scelto una tabella partizioni GPT, la prima partizione deve essere di tipo `BIOS boot` e di dimensione `1M`; In caso tu sia su BIOS Legacy e tabella MBR (`msdos`) allora non c'è bisogno di creare questa partizione.
- Se non ce l'hai già, aggiungi la partizione di boot; essa deve essere di circa 500 MB e preferibilmente di tipo `EFI System`;
- Aggiungi una partizione per il file system di root (e lasciala flaggata come `Linux filesystem`);
- Se vuoi, puoi aggiungere una partizione di swap creandola e flaggandola come `Linux swap`.

> In una tabella partizioni GPT, flaggare preventivamente la partizione di boot come ESP anche se si è su BIOS Legacy è utile per rendere più semplice una eventuale conversione da BIOS Legacy a UEFI.

Conferma le modifiche impartendo il comando `Write`; conferma scrivendo `yes` e poi esci dal gestore partizioni impartendo il comando `Quit`.

Per semplicità di riferimento, il disco usato in questa guida sarà `/dev/sda` ed esso sarà partizionato come segue:
```
/dev/sda
- /dev/sda1 -> EFI System
- /dev/sda2 -> Linux filesystem
- /dev/sda3 -> Linux swap
```

Crea adesso i file system:

La partizione di boot sarà `/dev/sda1` quindi:
```
# mkfs.fat -F 32 /dev/sda1
```

La partizione root di Linux sarà `/dev/sda2` quindi:
```
# mkfs.ext4 /dev/sda2
```

> È strettamente consigliato crittare la partizione root di Linux. Di seguito le istruzioni per utilizzare LUKS in congiunzione con LVM2.
>
> Per prima cosa, crea uno spazio crittato con LUKS sulla partizione root Linux.
> Ricorda di conservare bene la password che sceglierai!
> 
> ```
> # cryptsetup luksFormat -v -s 512 -h sha512 /dev/sda2
> ```
> 
> Monta lo spazio crittato:
>
> ```
> # cryptsetup open /dev/sda2 luks_root
> ```
> 
> Crea il gruppo dei volumi logici e anche il volume logico del filesystem root:
> 
> ```
> # pvcreate /dev/mapper/luks_root
> # vgcreate lvmgroup /dev/mapper/luks_root
> # lvcreate -l 100%FREE lvmgroup -n root
> ```
> 
> Infine, formatta il volume logico:
> 
> ```
> # mkfs.ext4 /dev/lvmgroup/root
> ```

La partizione swap sarà su `/dev/sda3`, quindi:
```
# mkswap /dev/sda3
```

## Montaggio del file system

Se hai creato una partizione swap, attivala con
```
# swapon /dev/sda3
```

Monta il filesystem root su `/mnt`:
```
# mount /dev/sda2 /mnt
```

> Ricorda, se hai il disco crittato, il file system root sarà `/dev/lvmgroup/root`:
> ```
> # mount /dev/lvmgroup/root /mnt
> ```

Crea la directory dove poter montare il filesystem della partizione di boot:
```
# mkdir /mnt/boot
```

Monta il file system della partizione di boot:
```
# mount /dev/sda1 /mnt/boot
```

## Installazione della distro

Installa tutti i pacchetti base:
```
# basestrap /mnt linux linux-firmware linux-headers base base-devel elogind
```

Installa l'init system di tua scelta:
```
# basestrap /mnt {init} elogind-{init}
```

> Per esempio, per installare:
> - OpenRC:
> ```
> # basestrap /mnt openrc elogind-openrc
> ```
> 
> - Runit:
> ```
> # basestrap /mnt runit elogind-runit
> ```
> 
> - S6:
> ```
> # basestrap /mnt s6-base elogind-s6
> ```
> 
> - Suite66:
> ```
> # basestrap /mnt 66 elogind-66
> ```

> In caso tu abbia crittato il disco, installa anche le utilità per gestire la partizione LUKS:
> ```
> # basestrap /mnt cryptsetup cryptsetup-{init} lvm2 lvm2-{init}
> ```

Adesso, genera la tabella partizioni:
```
# fstabgen -U /mnt >> /mnt/etc/fstab
```

Infine, fai chroot sul nuovo root file system:
```
# artix-chroot /mnt
```

# Configurazione della distro

## Installazione di `nano` come editor di testo

Da qui in poi avrai bisogno di un editor di testo. Questa guida consiglia GNU nano; tuttavia la scelta sta all'utente finale.

Installa `nano`:
```
# pacman -S nano
```

Modifica `/etc/profile` in modo che successivi login usino `nano` di default: a fine file aggiungi una riga con il testo seguente:
```
export EDITOR=nano
```

Lo stesso comando può essere usato (ed è consigliato usarlo) nella shell corrente per essere effettivo da subito.

## Solo per utenti Suite66: Configurazione dell'init system

Se hai scelto Suite66 come sistema di init, devi inizializzarne le configurazioni:
```
# 66-tree -ncE default
# 66-tree -n boot
# 66-enable -t boot boot@system
```

Cambia la configurazione di boot di Suite66 con questo comando:
```
# 66-env -t boot -e $EDITOR boot@system
```
Dentro la configurazione, cambia i valori `!no` a `!yes` dove necessario.

Suite66 gestisce anche l'hostname internamente alle sue configurazioni; ricorda di impostarlo.

Per applicare la configurazione appena salvata:
```
# 66-enable -t boot -F boot@system
```

## Configurazione località, orario e lingua

Installa il servizio di sincronizzazione oraria basata sul fuso orario:
```
# pacman -S chrony chrony-{init}
```

Aggiungi il servizio `chrony` alla lista d'avvio.

> Per esempio
> - OpenRC: 
> ```
> # rc-update add chrony default
> ```
> 
> - Runit:
> ```
> # ln -s /etc/runit/sv/chrony /etc/runit/runsvdir/default
> ```
> 
> - S6:
> ```
> # s6-rc-bundle-update add default chrony
> ```
> 
> - Suite66:
> ```
> # 66-enable -t default chrony
> ```

Imposta il fuso orario e crea il file `/etc/adjtime`:

```
# ln -sf /usr/share/zoneinfo/Europe/Rome /etc/localtime
# hwclock --systohc
```

> In caso di dual boot con Windows, è probabile che quest'ultimo abbia problemi ad indicare l'ora corretta. Per risolvere basta indicare a quest'ultimo di usare l'ora in formato UTC. Leggi [l'articolo sulla ArchWiki](https://wiki.archlinux.org/index.php/System_time#UTC_in_Windows) a riguardo.

Modifica il file `/etc/locale.gen`; decommenta le lingue e localizzazioni che userai.

> È consigliabile decommentare `en_US.UTF-8` anche se non si userà il computer nella lingua inglese americana;
> questo perché molte applicazioni non internazionalizzate dipendono da questa lingua per comunicare con l'utente.

Genera le localizzazioni con:
```
# locale-gen
```

Crea un file `/etc/locale.conf` e scrivici dentro:

```
export LANG="it_IT.UTF-8" <-- la tua lingua
export LC_COLLATE="C"
```

Se hai scelto OpenRC come sistema di init, modifica il file `/etc/conf.d/keymaps`:

```
keymap="it"       <-- il tuo layout tastiera
windowkeys="YES"  <-- scrivi YES se stai usando una tastiera comunemente usata su Windows
```

> Se pianifichi l'uso di una sessione grafica X.Org, faresti bene a configurare adesso il layout tastiera.
> 
> Crea il file `/etc/X11/xorg.conf.d/00-keyboard.conf`:
> ```
> Section "InputClass"
>         Identifier "system-keyboard"
>         MatchIsKeyboard "on"
>         Option "XkbLayout" "it" <-- il tuo layout tastiera
> EndSection
> ```

## Configurazione dell'hostname e del file hosts

Scegli un hostname per la tua macchina.

Modifica i seguenti file:

Modifica `/etc/hostname`:

```
miohostname
```

Modifica `/etc/hosts`:

```
127.0.0.1 localhost
::1       localhost
127.0.1.1 miohostname
```

Se hai scelto OpenRC come sistema di init, modifica `/etc/conf.d/hostname`:
```
# Set to the hostname of this machine
hostname="miohostname"
```

## Installazione del microcode per CPU Intel e AMD

Intel e AMD rilasciano frequentemente aggiornamenti al [microcode](https://it.wikipedia.org/wiki/Microprogrammazione) dei loro processori. È quindi importante rimanere il più aggiornati possibile per evitare crash sporadici e problemi alla stabilità del sistema.

Per installare l'ultimo microcode disponibile:
```
# pacman -S amd-ucode    <-- CPU AMD
# pacman -S intel-ucode  <-- CPU Intel
```

Il boot loader è responsabile del caricamento del microcode aggiornato. Per maggiori informazioni consulta [questo articolo della ArchWiki](https://wiki.archlinux.org/title/Microcode).

## Configurazione del boot loader

Ci sono vari modi per caricare il kernel Linux all'avvio e tutti sono spiegati bene in [questo articolo della ArchWiki](https://wiki.archlinux.org/index.php/Arch_boot_process#Boot_loader). Di questi, questa guida spiegherà come installare GRUB, che è il più conosciuto.

Installa i pacchetti necessari per installare e configurare GRUB:
```
# pacman -S grub
# pacman -S efibootmgr         <-- se sei su UEFI
# pacman -S ntfs-3g os-prober  <-- se stai facendo dual boot
```

Installa GRUB:
- Se sei su UEFI:
  ```
  # grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
  ```
- Se sei su BIOS:
  ```
  # grub-install --recheck /dev/sda
  ```

> Se stai crittando il disco, modifica i file di configurazione in modo da supportare la decrittazione all'avvio.
> - Modifica `/etc/mkinitcpio.conf`:
>   Aggiungi `encrypt` ed `lvm2` sul vettore `HOOKS` tra le entry `block` e `filesystem`.
> - Rendi effettive le nuove configurazioni con:
>   ```
>   # mkinitcpio -P
>   ```
> - Modifica `/etc/default/grub`:
>   ```
>   GRUB_CMDLINE_LINUX=”cryptdevice=/dev/sda2:luks_root”
>   ```

Genera la configurazione di GRUB:
```
# grub-mkconfig -o /boot/grub/grub.cfg
```

Per maggiori informazioni consulta [l'articolo sulla ArchWiki](https://wiki.archlinux.org/index.php/GRUB).

## Installazione di un gestore di reti

Questa guida consiglia `NetworkManager` in quanto è supportato da molti ambienti desktop tra cui GNOME, KDE, MATE e XFCE.

Per installarlo:
```
# pacman -S networkmanager networkmanager-{init}
# pacman -S dnsmasq      <-- per il supporto alla creazione di Hotspots
```

Aggiungi `NetworkManager` alla lista servizi all'avvio.

### Alternative

#### `connman`

Installa `connman` insieme ad un client DHCP:
```
# pacman -S connman connman-{init} dhclient
```

Aggiungi `connmand` alla lista servizi all'avvio.

Installa anche `connman-gtk` (se userai GNOME, XFCE, MATE) o `cmst` (se userai KDE, LXQt)
```
# pacman -S connman-gtk  <-- GTK-based DEs
# pacman -S cmst         <-- Qt-based DEs
```

Se hai intenzione di usare una scheda di rete wireless, avrai anche bisogno di `wpa_supplicant`:
```
# pacman -S wpa_supplicant wpa_supplicant-{init}
```

## Opzionale: Installazione di un logger di sistema

Installa Syslog-ng:
```
# pacman -S syslog-ng syslog-ng-{init}
```

Aggiungi `syslog-ng` alla lista servizi all'avvio.

## Installazione di Filesystem in Userspace

Filesystem in Userspace consente di montare supporti esterni senza dover invocare alcun comando via root.

Per installarlo:
```
# pacman -S fuse2 fuse3 fuse-{init}
```

Aggiungi `fuse` alla lista servizi all'avvio.

## Configurazione utente

Cambia la password di root:
```
# passwd
```

Dopodiché, crea un utente che userai per le tue normali operazioni:
```
# useradd -m nomeutente
```

Imposta una password per questo nuovo utente:
```
# passwd nomeutente
```

Aggiungi questo utente al gruppo `wheel` (così può effettuare alcune operazioni di amministrazione di sistema):
```
# usermod -a -G wheel nomeutente
```

> Altri gruppi utili sono `lp`, `storage`, `input`, `games`, `rfkill`, `optical`, `disk`.
> 
> I gruppi `audio` e `video` sono utili per comunicare direttamente con schede video e audio.

## Configurazione di `sudo`

Bisogna configurare `sudo` in modo da permettere agli utenti del gruppo `wheel` di eseguire comandi come root.
```
# visudo
```

A tre quarti del file che si è aperto esiste una riga da decommentare:
```
# %wheel ALL=(ALL) ALL <-- decommenta
```

## Installazione di X.Org

X.Org è il server grafico più usato su Linux. Installalo con:
```
# pacman -S xorg
```

Riferisci all'[articolo sulla ArchWiki](https://wiki.archlinux.org/index.php/Xorg) per più informazioni su questo server grafico.

## Opzionale: Installazione dei driver NVIDIA (per computer con schede grafiche NVIDIA)

Da terminale installa il pacchetto `nvidia` (o `nvidia-lts` se stai usando una versione LTS del kernel Linux).

```
# pacman -S nvidia
```

Se hai un portatile con due schede grafiche (ad es. integrata Intel e discreta NVIDIA) avrai anche bisogno del pacchetto `nvidia-prime`.

```
# pacman -S nvidia-prime
```

## Installazione di un ambiente grafico

### Installazione di KDE Plasma

Per installare KDE Plasma basta installare il gruppo di applicazioni `plasma` e, successivamente, un subset del gruppo di applicazioni `kde-applications`.

Installa Plasma: 
```
# pacman -S plasma
```

Installa il gruppo di applicazioni KDE:
```
# pacman -S kde-applications
```

> Sarebbe consigliabile non installare tutto il gruppo di applicazioni KDE, ma solo un subset scelto dall'utente.
> Di seguito una lista di applicazioni consigliabili per una installazione considerata decente.
> ```
> ark
> dolphin
> dolphin-plugins
> ffmpegthumbs
> gwenview
> kate
> kcalc
> kdeconnect
> kdegraphics-thumbnailers
> kdesdk-kioslaves
> kdesdk-thumbnailers
> kdialog
> kfind
> kio-extras
> kio-gdrive
> konsole
> kwalletmanager
> okular
> spectacle
> zeroconf-ioslave
> ```

#### Installazione di SDDM (display manager)

```
# pacman -S sddm sddm-{init}
```

Aggiungi `sddm` alla lista servizi all'avvio.

## Uscita dal chroot

```
# exit
```

## Smontaggio delle partizioni

```
# umount -R /mnt
```

## Riavvio

`reboot` (ricordati di scollegare la chiavetta USB)

# Post-installazione

## Disattivare il beeper

### Nella shell

Modifica `/etc/inputrc`:

```
# do not bell on tab-completion
set bell-style none             <-- decommenta
```

### Oppure totalmente

Crea il file `/etc/modprobe.d/blacklist.conf` e scrivi dentro:
```
blacklist pcspkr
```

## Installare il demone Bluetooth

Installa BlueZ:
```
$ sudo pacman -S bluez bluez-{init}
```

Aggiungi `bluetoothd` alla lista servizi all'avvio.

## Installare il demone per le stampanti

Installa CUPS:
```
$ sudo pacman -S cups libcups cups-filters cups-{init}
```

Aggiungi `cupsd` alla lista servizi all'avvio.

Aggiungi il tuo utente al gruppo di CUPS: 
```
$ sudo usermod -a -G cups $USER
```

Installa il pacchetto `system-config-printer`:
```
$ sudo pacman -S system-config-printer
```
> Non installare questo pacchetto potrebbe causare il seguente errore:
> `The name org.fedoraproject.Config.Printing was not provided by any .service files`

> Su KDE Plasma è consigliato installare il pacchetto `print-manager`:
> ```
> $ sudo pacman -S print-manager
> ```

## Installare il demone per gli scanner

Installa SANE:
```
pacman -S sane sane-frontends sane-{init}
```

Aggiungi `saned` alla lista servizi all'avvio.

## Installare Samba per il file sharing su rete locale

Installa Samba:
```
$ sudo pacman -S samba samba-{init}
```

Scarica il file di configurazione di esempio di Samba:
```
$ sudo curl -o /etc/samba/smb.conf "https://git.samba.org/samba.git/?p=samba.git;a=blob_plain;f=examples/smb.conf.default;hb=HEAD" 
```

Aggiungi `smb` alla lista servizi all'avvio.

Aggiungi un gruppo chiamato `sambashare`:
```
$ sudo groupadd sambashare
```

Aggiungi il tuo utente al gruppo `sambashare`:
```
$ sudo usermod -a -G sambashare $USER
```

> Su KDE Plasma è consigliato installare il pacchetto `kdenetwork-filesharing`:
> ```
> $ sudo pacman -S kdenetwork-filesharing
> ```

## Configurare Pacman

È possibile modificare `/etc/pacman.conf` per personalizzarne il comportamento:

- Decommentando l'opzione `Color`, l'output di Pacman sarà colorato e più leggibile;
- Puoi decommentare la sezione `[lib32]` (e relativo `Include`) per avere il supporto multilib;
- Puoi aggiungere la repository Artix Universe per avere qualche pacchetto precompilato in più:
  ```
  [universe]
  Server = https://universe.artixlinux.org/$arch
  ```
  
Nel caso ci fossero state modifiche alle repo di Pacman, è obbligatorio aggiornare il database con:
```
# pacman -Sy
```

## Opzionale: Aggiungere le repo di Arch Linux

> NOTA: Artix ha scelto di NON includere le repo di Arch come fallback; questo serve per evitare che una installazione di Artix si possa rompere con l'installazione di pacchetti erroneamente scaricati dalle repo Arch, quindi dipendenti da Systemd.
> Prima di abilitare queste repository è altamente consigliato provare ad usare il sistema installato senza di esse.

Nel caso volessi includere le repo Arch Linux sul tuo sistema Artix, devi installare il pacchetto `artix-archlinux-support` ed aggiungere le repo di Arch a `/etc/pacman.conf`:

```
#
# ARCHLINUX
#

#[testing]
#Include = /etc/pacman.d/mirrorlist-arch

[extra]
Include = /etc/pacman.d/mirrorlist-arch

#[community-testing]
#Include = /etc/pacman.d/mirrorlist-arch

[community]
Include = /etc/pacman.d/mirrorlist-arch

#[multilib-testing]
#Include = /etc/pacman.d/mirrorlist-arch

#[multilib]
#Include = /etc/pacman.d/mirrorlist-arch
```

Se stai facendo uso delle repo `lib32` puoi decommentare `multilib` e relativo `Include`.

Ricorda di popolare il keyring di Pacman:
```
# pacman-key --populate archlinux`
```

Infine, aggiorna il database:
```
# pacman -Sy
```

## Velocizzare la compilazione di pacchetti da AUR

È possibile velocizzare notevolmente la compilazione di pacchetti dalla AUR; per ciò basta aumentare il numero di jobs paralleli che `make` può usare.

Ciò è possibile grazie alla modifica del file di configurazione `/etc/makepkg.conf`:

Qui, basta cercare la variabile `MAKEFLAGS` e decommentarla;
poi basta cambiare la flag `-j2` a `-j` + il numero di thread disponibili sul processore.

Per capire il numero di thread disponibili sul sistema:
```
$ nproc --all
```

### Opzionale: Configurare il dominio di regole della scheda di rete Wireless

Installa `crda`:
```
$ sudo pacman -S crda
```

Modifica il file `/etc/conf.d/wireless-regdom`: qui bisogna decommentare la linea `WIRELESS_REGDOM` coincidente con il Paese in cui si abita.

NOTA: Alcune schede (soprattutto quelle Intel) potrebbero decidere arbitrariamente che set di regole utilizzare.

## Risolvere problemi comuni

### Steam non rimappa i controller a Xbox

Per risolvere, crea il file `/etc/modules-load.d/uinput` con dentro:
```
uinput
```

### Il computer non trova alcun servizio mDNS

Installa il servizio Avahi:
```
$ sudo pacman -S avahi avahi-{init}
```

Aggiungi `avahi-daemon` alla lista servizi all'avvio.

### KDE Plasma non aggiorna la lista applicazioni

Triggera il refresh manuale:
```
$ kbuildsycoca5
```

