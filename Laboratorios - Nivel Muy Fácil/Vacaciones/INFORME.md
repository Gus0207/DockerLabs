
| Plataforma          | Docker Labs |
| ------------------- | ----------- |
| Dificultad          | Muy Fácil   |
| IP de la Máquina    | 172.17.0.2  |
| Fecha de resolución | 03/10/2026  |
# 📑 WRITEUP: VACACIONES

> [!INFO] **RESUMEN**
> - **IP Objetivo:** `172.17.0.2`
> - **Sistema Operativo:** Linux 🐧
> - **Descripción** Laboratorio para practicar fuerza bruta contra SSH y escalada de privilegios en Linux. ✉️
> - **Privilegios Obtenidos:** `root` 👑

---

## 🎯 FASE 1: RECONOCIMIENTO Y ENUMERACIÓN

### 💻 Conectividad Inicial
Determinamos el Sistema Operativo analizando el Time To Live (TTL) mediante un paquete ICMP:
```bash
ping -c 1 172.17.0.2
```

![[Laboratorios - Nivel Muy Fácil/Vacaciones/Capturas/PING a IP objetivo.png]]

* **Diagnóstico:** TTL = `64`. Confirmamos que la máquina víctima corre bajo un entorno **Linux**.

### 🔍 Mapeo de Puertos y Servicios
Lanzamos un escaneo con **Nmap** para identificar las vías de acceso expuestas.

![[Laboratorios - Nivel Muy Fácil/Vacaciones/Capturas/Scan NMAP.png]]

| Puerto   | Servicio | Versión Detectada   | Estado / Nota                              |
| :------- | :------- | :------------------ | :----------------------------------------- |
| `22/TCP` | SSH      | OpenSSH 7.6p1       | 🔐 Vector de acceso inicial (Fuerza Bruta) |
| `80/TCP` | HTTP     | Apache httpd 2.4.29 | 🌐 Servidor Web Activo (Fuga de nombres)   |

#### Comando de Nmap utilizado:
```bash
sudo nmap -p- -sS -sV --min-rate 5000 -n -Pn --open 172.17.0.2
```

---

## ⚡ FASE 2: EXPLOTACIÓN (ACCESO INICIAL)

### 🌐 Inspección Web y Descubrimiento de Usuarios
Al ingresar a la aplicación web (`http://172.17.0.2`) no tiene contenido visible, pero en su código fuente hay un comentario interesante:

```html
<!-- De : Juan Para: Camilo , te he dejado un correo es importante... -->
```

> [!TIP] Posibles usuarios Identificados
> El código fuente revela la existencia de 2 posibles usuarios identificados en el sistema operativo, `Juan` y `Camilo`

### 🔑 Ataque de Fuerza Bruta (SSH)
Con un usuario válido confirmado, utilizamos **Hydra** contra el servicio SSH (puerto 22) empleando el diccionario estándar `rockyou.txt`:

```bash
hydra -l camilo -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

![[Ataque Hydra sobre camilo.png]]

> [!SUCCESS] **Credenciales Iniciales Halladas**
> - **Usuario:** `camilo`
> - **Contraseña:** `password1`

Nos conectamos mediante SSH con las credenciales obtenidas:
```bash
ssh camilo@172.17.0.2
```

![[Conexion ssh a Camilo.png]]

---

## 🔄 FASE 3: MOVIMIENTO LATERAL (CAMILO -> JUAN)

Una vez dentro como el usuario `camilo`, realizamos tareas de recolección de información local y nos percatamos de que no tiene permisos directos para elevar a root. Sin embargo, auditando el directorio de correos locales del sistema descubrimos un archivo crítico:

![[Revision de permisos Camilo.png]]

```bash
ls -la /var/mail/
# Detectamos un archivo de correo para el usuario camilo
cat /var/mail/camilo/correo.txt
```

![[Correo para Camilo.png]]

> [!Warning] Fuga de Credenciales Interna
> Al leer el correo, interceptamos un mensaje del usuario `juan` donde comparte su contraseña en texto claro:
> - **Usuario:** `juan`
> - **Contraseña:** `2k84dicb`

Pivotamos de cuenta mediante la terminal interactiva:
```bash
su juan
```

![[Pivote de cuenta a Juan.png]]

---

## 👑 FASE 4: ESCALADA DE PRIVILEGIOS (JUAN -> ROOT)

Estando bajo el contexto del usuario `juan`, ejecutamos el comando de control de privilegios delegados:

```bash
sudo -l
```

![[Permisos para usuario Juan.png]]

> [!Warning] Permiso de Ejecución Inseguro
> El output nos indica que el usuario `juan` puede ejecutar **Ruby** como `root` sin necesidad de contraseña (`NOPASSWD: /usr/bin/ruby`).

### ⚔️ Explotación mediante Ruby (GTFOBins)
Abusamos de la capacidad de Ruby para ejecutar comandos del sistema operativo desde su propio entorno, solicitando una shell bash privilegiada:

```bash
sudo ruby -e 'exec "/bin/sh"'
```

Validamos el éxito de la intrusión midiendo nuestra identidad actual:
```bash
whoami
```

![[Laboratorios - Nivel Muy Fácil/Vacaciones/Capturas/ROOT.png]]

```bash
# Output de la consola comprometida:
root
```

> [!SUCCESS] **MÁQUINA COMPLETADA**
> Acceso administrativo total garantizado. Hemos escalado con éxito explotando los permisos sobre Ruby tras un movimiento lateral por lectura de correspondencia interna.

---

## 🛠️ FASE 5: VULNERABILIDADES Y MITIGACIONES

>[!Danger] CWE-250

>[!Danger] Crítica
>El usuario `juan` puede ejecutar ruby como root sin contraseña.

>[!Todo] Mitigación
>No delegar permisos completos de `sudo` a lenguajes de programación o intérpretes de comandos (`Ruby`, `Python`, `Perl`) de forma libre, ya que rompen de manera las restricciones del sistema.

>[!Warning] CWE-521

>[!Warning] Alta
> El usuario utiliza una contraseña simple (`password1`) y fácilmente accesible por SSH mediante fuerza bruta

>[!Todo] Mitigación
>Utilizar contraseñas de gran longitud, bloqueo de intentos y sistema de autenticación para las contraseñas. 

>[!Warning] CWE-312
>

>[!Warning] Alta
>Un correo en /var/mail/camilo guarda en claro la contraseña del usuario `juan`, permitiendo el movimiento lateral entre cuentas.

>[!Todo] Mitigación
> Implementar restricciones estrictas en los directorios de mensajería (`/var/mail/`) para que únicamente el destinatario legítimo pueda leer el contenido.

>[!Info] CWE-200

>[!Info] Baja
>El mensaje en el código fuente HTML nombra a `juan` y `camilo` los cuales se pueden suponer son usuarios, siendo posibles objetivos de ataques mediante fuerza bruta.

>[!Todo] Mitigación
>Evitar dejar nombres de usuario, autores o pistas de configuración en los comentarios del código fuente de producción (`HTML/JS`).


> [!NOTE] Desconfiguración Identificada
> El listado de resultados expone una anomalía crítica: el binario **`/usr/bin/env`** tiene asignados permisos SUID.

