# Proyecto Integrador: Servidor para el Centro de Terapias Integrales Bienestar

**Materia:** Sistemas Operativos  
**Integrante / Administrador:** Jhonatan Perez, Katherine Galvis , Karina Garcia (`user`)  
**Sistema Operativo:** Ubuntu Server 24.04.5 LTS  
**Conexión remota:** SSH (`ssh -p 2222 user@127.0.0.1`)

---

## 1. De qué trata el caso

Para este proyecto decidimos montar un servidor para un consultorio llamado **Centro de Terapias Integrales Bienestar**, donde atienden dos profesionales (psicología y fisioterapia), una recepcionista y un director médico. 

La necesidad principal del consultorio es tener sus agendas de citas y las historias clínicas de los pacientes organizadas en un servidor Linux, pero cumpliendo tres cosas clave:
1. **Privacidad de los pacientes:** Que la recepcionista pueda manejar las citas, pero nunca pueda abrir historias clínicas; y que cada terapeuta vea solo las historias de sus propios pacientes.
2. **Disponibilidad:** Tener un servicio propio que vigile el estado del servidor y que, si se llega a caer, vuelva a arrancar solo.
3. **Respaldos seguros:** Hacer copias de seguridad automáticas de toda la información y comprobar que de verdad se pueden restaurar si algo se borra.

*(Nota: Todos los nombres de pacientes y archivos que creamos en el servidor son inventados para no usar datos personales reales).*

---

## 2. Usuarios, grupos y permisos

Antes de tirar comandos en la consola, organizamos quiénes iban a usar el servidor y a qué carpetas debían entrar:

### Grupos que creamos
* **`equipo_clinico`:** Aquí metimos a todos los trabajadores del consultorio (`dr`, `jhonatan`, `karina` y `kathe`). Sirve para que puedan entrar a la carpeta principal `/srv/consultorio` y compartir la carpeta de `citas`.
* **`auditoria`:** Aquí solo está el director (`dr`). Lo creamos para darle permiso de solo lectura en las carpetas de historias clínicas cuando necesite revisarlas, sin que pueda borrar ni alterar nada.

### Usuarios del sistema
* **`user`:** Mi usuario como administrador del servidor (es el único que tiene `sudo` y acceso por SSH).
* **`dr`:** El director del consultorio (está en `equipo_clinico` y `auditoria`).
* **`jhonatan`:** Psicólogo del centro (está en `equipo_clinico`).
* **`karina`:** Fisioterapeuta del centro (está en `equipo_clinico`).
* **`kathe`:** Recepcionista (está en `equipo_clinico`).

### Tabla de carpetas y permisos

| Carpeta | Dueño : Grupo | Permisos | Quién entra y qué puede hacer | Por qué lo configuramos así |
| :--- | :--- | :--- | :--- | :--- |
| `/srv/consultorio` | `dr:equipo_clinico` | `750` (`rwxr-x---`) | Todo el equipo clínico puede entrar; cualquier otro usuario del sistema queda por fuera. | Para aislar los archivos del consultorio del resto del servidor. |
| `/srv/consultorio/citas` | `kathe:equipo_clinico` | `770` (`rwxrwx---`) | `kathe`, `jhonatan`, `karina` y `dr` pueden leer y modificar las citas. | Porque tanto recepción como los terapeutas necesitan revisar y anotar horarios en la agenda. |
| `/srv/consultorio/historias_jhonatan` | `jhonatan:auditoria` | `750` (`rwxr-x---`) | Solo `jhonatan` puede leer y escribir; `dr` (por el grupo `auditoria`) solo puede leer. | Para cuidar el secreto profesional de psicología; si `kathe` o `karina` intentan entrar, les sale *Permission denied*. |
| `/srv/consultorio/historias_karina` | `karina:auditoria` | `750` (`rwxr-x---`) | Solo `karina` puede leer y escribir; `dr` (por el grupo `auditoria`) solo puede leer. | Para proteger las historias de fisioterapia frente a otros empleados que no deben verlas. |
| `/srv/consultorio/backups` | `root:dr` | `750` (`rwxr-x---`) | Solo `root` (mediante `cron`) guarda los respaldos; el director `dr` puede entrar a verlos. | Para evitar que algún empleado borre o dañe por error las copias de seguridad. |

### Pruebas que hicimos
* Cuando intentamos entrar con la recepcionista (`kathe`) a las historias de psicología (`su - kathe -c "ls /srv/consultorio/historias_jhonatan"`), el sistema nos bloquea con **`Permission denied`**.
* Cuando entramos con `jhonatan` (`su - jhonatan -c "cat /srv/consultorio/historias_jhonatan/hc_juan_perez.txt"`), sí deja leer el archivo normalmente.

---

## 3. Servicio propio en `systemd`

Creamos el servicio **`monitoreo-consultorio.service`**, que ejecuta un script cada 30 segundos para revisar cuánto espacio ocupado tiene el disco y cuántas historias clínicas hay guardadas, dejando el registro en `/var/log/consultorio_estado.log`.

* **Script:** `/usr/local/bin/monitoreo_consultorio.sh`
* **Archivo del servicio:** `/etc/systemd/system/monitoreo-consultorio.service`
* **Recuperación ante fallos:** Lo dejamos habilitado con `systemctl enable` para que arranque con el servidor, y le configuramos `Restart=always` y `RestartSec=3`. Probamos matarlo a la fuerza con `sudo systemctl kill -s SIGKILL monitoreo-consultorio.service` y comprobamos que a los 3 segundos vuelve a quedar `active (running)` solo.

---

## 4. Respaldos automáticos (`tar` + `cron`) y restauración

Para no perder las citas ni las historias clínicas, armamos un script que empaqueta y comprime las tres carpetas principales dentro de `/srv/consultorio/backups/`:

* **Script de respaldo:** `/usr/local/bin/respaldo_consultorio.sh`
* **Formato del archivo:** `respaldo_consultorio_FECHA_HORA.tar.gz` (con permisos `640` para que solo `root` y `dr` puedan verlo).
* **Automatización:** Lo programamos en el `crontab` de `root` para que se ejecute automáticamente sin intervención manual.
* **Prueba de restauración:** Creamos la carpeta `/tmp/restauracion_prueba` y descomprimimos ahí el respaldo más reciente con `tar -xzf`. Verificamos con `ls -R` y `cat` que todas las carpetas y las historias clínicas se recuperaron completas.

---

## 5. Seguridad básica que aplicamos

1. **Cero permisos `777`:** Revisamos todo `/srv/consultorio` con `find /srv/consultorio -perm 0777` para asegurarnos de que ninguna carpeta quedara abierta a todo el mundo, evitando que cualquiera pueda borrar o cambiar historias clínicas.
2. **`sudo` solo para el administrador:** Ningún usuario del consultorio (`dr`, `jhonatan`, `karina`, `kathe`) tiene permisos de `sudo`; solo mi usuario `user` puede administrar el sistema.
3. **Servidor actualizado:** Corrimos `sudo apt update && sudo apt upgrade` para tener los parches de seguridad al día.
4. **Administración por SSH:** Nos conectamos de forma remota desde Windows usando SSH con nuestro usuario autenticado, tal como se administra un servidor real.

---

## 6. Qué hacer si algo falla (Diagnóstico rápido)

| Qué pasó | Cómo revisarlo | Cómo solucionarlo |
| :--- | :--- | :--- |
| **El servicio de monitoreo aparece caído** | `sudo systemctl status monitoreo-consultorio.service` | Revisar que el script tenga permiso de ejecución (`chmod +x /usr/local/bin/monitoreo_consultorio.sh`) y reiniciarlo con `sudo systemctl restart monitoreo-consultorio.service`. |
| **No se están creando copias de seguridad** | `sudo crontab -l` y `sudo systemctl status cron` | Verificar que el demonio `cron` esté corriendo y que la ruta del script en `crontab -e` esté bien escrita. |
| **Un doctor no puede entrar a su carpeta** | `ls -ld /srv/consultorio/historias_*` | Volver a asignar el dueño y permisos correctos con `sudo chown <usuario>:auditoria <carpeta>` y `sudo chmod 750 <carpeta>`. |
| **Se borró una historia clínica por error** | `sudo ls -lht /srv/consultorio/backups/` | Extraer el `.tar.gz` más reciente en `/tmp/restauracion_prueba` con `tar -xzf` y volver a copiar el archivo perdido a su carpeta original. |

---

## 7. Evidencias

*(Capturas guardadas en la carpeta `evidencias/` del repositorio):*
* `evidencias/01_permisos.png` — Prueba de acceso permitido y *Permission denied*.
* `evidencias/02_servicio_systemd.png` — Servicio activo y reinicio automático después de `SIGKILL`.
* `evidencias/03_respaldo_restauracion.png` — Respaldos creados por `cron` y archivos restaurados en `/tmp`.
