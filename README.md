---

# Guía de Instalación de Arch Linux en Sistemas UEFI con Cifrado de Disco

## 1. Preparativos

1. **Descargar la ISO de Arch Linux**
   - Visita la [página oficial de Arch Linux](https://archlinux.org/download/) para descargar la ISO más reciente.

2. **Crear un USB Booteable con Rufus**
   - Usa Rufus para crear un USB booteable con la ISO de Arch Linux.
   - En Rufus, selecciona el **Esquema de partición** como **GPT** y el **Sistema de destino** como **UEFI (no CSM)**.

3. **Preparar el Disco**
   - Se recomienda tener un disco libre para la instalación para evitar complicaciones. Asegúrate de que el disco esté particionado.

## 2. Iniciar el Entorno de Instalación

1. **Cargar la Distribución del Teclado (opcional)**
   - Si deseas usar un teclado en español:
     ```bash
     root@archiso ~# loadkeys la-latin1
     ```

2. **Verificar el Modo UEFI**
   - Ejecuta el siguiente comando para asegurarte de que el sistema esté arrancado en modo UEFI:
     ```bash
     root@archiso ~# ls /sys/firmware/efi/efivars
     ```

3. **Verificar la Conexión a Internet**
   - Comprueba la conectividad con:
     ```bash
     root@archiso ~# ping archlinux.org
     ```
   - Si no tienes acceso a internet, puedes conectar un cable Ethernet o usar `iwctl` para configurar Wi-Fi:
     ```bash
     root@archiso ~# iwctl
     ```

     - Lista los dispositivos Wi-Fi:
       ```bash
       root@archiso ~# device list
       ```
     - Enciende el dispositivo (reemplaza `name` con el nombre del adaptador):
       ```bash
       root@archiso ~# device name set-property Powered on
       root@archiso ~# adapter adapter set-property Powered on
       ```
     - Escanea redes disponibles:
       ```bash
       root@archiso ~# station name scan
       ```
     - Enumera las redes disponibles:
       ```bash
       root@archiso ~# station name get-networks
       ```
     - Conéctate a una red (reemplaza `SSID` con el nombre de la red):
       ```bash
       root@archiso ~# station name connect SSID
       ```
     - Verifica la conexión:
       ```bash
       root@archiso ~# ping google.com
       ```

4. **Sincronizar la Hora del Sistema**
   - Asegúrate de que el reloj del sistema esté sincronizado:
     ```bash
     root@archiso ~# timedatectl
     ```

## 3. Particionamiento del Disco

1. **Identificar los Discos**
   - Lista los discos y verifica que se muestre "GPT Present":
     ```bash
     root@archiso ~# lsblk
     ```

2. **Crear Particiones con `gdisk`**
   - **Crear la partición EFI:**
     ```bash
     root@archiso ~# gdisk /dev/nombre_de_tu_disco
     n
     ENTER
     ENTER
     +2GB
     EF00
     W
     Y
     ```
   - **Crear la partición raíz `/`:**
     ```bash
     root@archiso ~# gdisk /dev/nombre_de_tu_disco
     n
     ENTER
     ENTER
     +100G  # Ajusta el tamaño según tu preferencia
     8300
     W
     Y
     ```
   - **Crear la partición `/home`:**
     ```bash
     root@archiso ~# gdisk /dev/nombre_de_tu_disco
     n
     ENTER
     ENTER
     ENTER  # Usa el espacio restante
     8300
     W
     Y
     ```

## 4. Notas Adicionales

- Asegúrate de revisar la documentación de Arch Linux para ajustes específicos y detalles adicionales.

Claro, aquí está la guía ajustada para iniciar en la sección 5, con las instrucciones para formatear, cifrar y montar particiones, instalar el sistema base, y configurar GRUB. También te explico cómo incluir imágenes en un archivo `README.md`.


## 5. Formatear y Encriptar Particiones

Asegúrate de adaptar los comandos a tus particiones y nombres de disco específicos. Puedes verificar los nombres con `fdisk -l`.

1. **Formatear y cifrar la partición raíz (`/`):**
   ```bash
   root@archiso ~# cryptsetup luksFormat /dev/nvme0n1p2
   root@archiso ~# cryptsetup open /dev/nvme0n1p2 root
   root@archiso ~# mkfs.ext4 /dev/mapper/cryptroot
   ```

2. **Formatear y cifrar la partición `/home`:**
   ```bash
   root@archiso ~# cryptsetup luksFormat /dev/nvme0n1p3
   root@archiso ~# cryptsetup open /dev/nvme0n1p3 home
   root@archiso ~# mkfs.ext4 /dev/mapper/home
   ```

3. **Formatear la partición EFI (`/boot`):**
   ```bash
   root@archiso ~# mkfs.vfat -F32 /dev/nvme0n1p1
   ```

4. **Formatear las particiones cifradas (de nuevo para asegurar):**
   ```bash
   root@archiso ~# mkfs.ext4 /dev/mapper/root
   root@archiso ~# mkfs.ext4 /dev/mapper/home
   ```

## 6. Montar Particiones en `/mnt`

1. **Montar la partición raíz (`/`):**
   ```bash
   root@archiso ~# mount /dev/mapper/root /mnt
   ```

2. **Crear y montar la partición `/home`:**
   ```bash
   root@archiso ~# mkdir /mnt/home
   root@archiso ~# mount /dev/mapper/home /mnt/home
   ```

3. **Crear y montar la partición EFI (`/boot`):**
   ```bash
   root@archiso ~# mkdir /mnt/boot
   root@archiso ~# mount /dev/nvme0n1p1 /mnt/boot
   ```

## 7. Instalar el Sistema Base

1. **Instalar el sistema base y paquetes esenciales:**
   ```bash
   root@archiso ~# pacstrap /mnt base base-devel nano neovim vim linux linux-firmware grub networkmanager dhcpcd efibootmgr dialog wpa_supplicant netctl
   ```

## 8. Generar `fstab`

1. **Generar el archivo `fstab`:**
   ```bash
   root@archiso ~# genfstab -U /mnt >> /mnt/etc/fstab
   ```

2. **Verificar el archivo `fstab`:**
   ```bash
   root@archiso ~# cat /mnt/etc/fstab
   ```

## 9. Configuración de Cifrado

1. **Modificar `crypttab`:**
   ```bash
   root@archiso ~# nvim /mnt/etc/crypttab
   ```
   Agrega las siguientes líneas al final del archivo:
   ```plaintext
   root UUID=1234-5678-90AB-CDEF none luks
   home UUID=<uuid> <keyfile> <options>
   ```
   ```markdown 
   ![Ejemplo de Configuracion](Images\crypttab.jpg)
   ```

## 10. Chroot y Configuración del Sistema

1. **Entrar en el entorno chroot:**
   ```bash
   root@archiso ~# arch-chroot /mnt /bin/bash
   ```

2. **Generar locales:**
   ```bash
   root@archiso ~# nano /etc/locale.gen
   ```
   Descomenta la línea:
   ```plaintext
   es_MX.UTF-8 UTF-8
   ```

   Construir el soporte de idioma:
   ```bash
   root@archiso ~# locale-gen
   ```

3. **Crear el archivo de configuración correspondiente:**
   ```bash
   root@archiso ~# echo Nombre_Del_PC >> /etc/hostname
   root@archiso ~# echo LANG=es_MX.UTF-8 >> /etc/locale.conf
   root@archiso ~# echo KEYMAP=es >> /etc/vconsole.conf
   ```

4. **Ajustar la zona horaria:**
   ```bash
   root@archiso ~# tzselect
   ```
   Selecciona:
   - 2 (Zona Horaria)
   - 32 (Número correspondiente a la zona)
   - 1 (Número correspondiente a la subzona)

   Borra el archivo de configuración anterior y crea el link simbólico:
   ```bash
   root@archiso ~# rm /etc/localtime
   root@archiso ~# ln -s /usr/share/zoneinfo/<ZONA>/<SUB_ZONA> /etc/localtime
   ```

## 11. Configuración de GRUB

1. **Editar el archivo de configuración de GRUB:**

   Abre el archivo de configuración de GRUB en un editor de texto con privilegios de superusuario:
   ```bash
   root@archiso ~# nvim /etc/default/grub
   ```

   ```markdown 
   ![Modificacion del GRUB_CMDLINE?LINUX](Images\grub.jpg)
   ```
   Asegúrate de agregar o ajustar las siguientes líneas:
   ```plaintext
   GRUB_ENABLE_CRYPTODISK=enable
   GRUB_CMDLINE_LINUX="cryptdevice=/dev/nvme0n1p2:root root=/dev/mapper/root"
   ```

   Si `GRUB_ENABLE_CRYPTODISK` no está presente, añádela como una nueva línea.

2. **Modificar `mkinitcpio.conf`:**

   Abre el archivo `mkinitcpio.conf` para añadir el hook `encrypt`:
   ```bash
   root@archiso ~# nvim /etc/mkinitcpio.conf
   ```

   ```markdown 
   ![Modificacion del GRUB_CMDLINE?LINUX](Images\mkinitcpio.jpg)
   ```

   Dentro del archivo, localiza la línea que empieza con `HOOKS` y asegúrate de que `encrypt` esté incluido después de `block`, como se muestra a continuación:
   ```plaintext
   HOOKS=(base udev block encrypt filesystems keyboard fsck)
   ```

3. **Regenerar el archivo `initramfs`:**

   Después de modificar `mkinitcpio.conf`, debes regenerar el archivo `initramfs`:
   ```bash
   root@archiso ~# mkinitcpio -P
   ```

4. **Actualizar la configuración de GRUB:**

   Instala GRUB en el disco y genera el archivo de configuración de GRUB:
   ```bash
   root@archiso ~# grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
   root@archiso ~# grub-mkconfig -o /boot/grub/grub.cfg
   ```

5. **Salir del entorno chroot y reiniciar:**

   Sal del entorno chroot y desmonta las particiones:
   ```bash
   root@archiso ~# exit
   root@archiso ~# umount -R /mnt
   root@archiso ~# reboot
   ```

---
