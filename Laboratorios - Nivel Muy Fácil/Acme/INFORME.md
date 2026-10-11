
| Plataforma          | Docker Labs |
| ------------------- | ----------- |
| Dificultad          | Muy Fácil   |
| IP de la Máquina    | 172.17.0.2  |
| Fecha de resolución | 03/10/2026  |

# 📑 WRITEUP: ACME

> ℹ️ ****RESUMEN****
> - **IP Objetivo:** `172.17.0.2`
> - **Sistema Operativo:** Linux 🐧
> - **Descripción:** Maquina con acceso fácil al ssh donde después deberás explorar los servicios web interno e intentar explotarlo 🌐
> - **Privilegios Obtenidos:** `root` 👑

---

## 🎯 FASE 1: RECONOCIMIENTO Y ENUMERACIÓN

### 💻 Conectividad Inicial
Determinamos el Sistema Operativo analizando el Time To Live (TTL) mediante un paquete ICMP:
```bash
ping -c 1 172.17.0.2
```

![PING a IP objetivo](Capturas/PING%20a%20IP%20objetivo.png)

* **Diagnóstico:** TTL = `64`. Confirmamos que la máquina víctima corre sobre un sistema **Linux**.

### 🔍 Mapeo de Puertos y Servicios
Lanzamos un escaneo por **Nmap** para indexar de manera precisa las vías de acceso expuestas.

![Scan NMAP](Capturas/Scan%20NMAP.png)

| Puerto   | Servicio | Versión Detectada   | Estado / Nota                              |
| :------- | :------- | :------------------ | :----------------------------------------- |
| `80/TCP` | HTTP     | Apache httpd 2.4.52 | 🔴 Entrada Principal (Aplicación Web ACME) |
| `22/TCP` | SSH      | OpenSSH 8.9p1       | 🟢 Activo (Para posible persistencia)      |

#### Comando de Nmap utilizado:
```bash
sudo nmap -p- -sS -sV --min-rate 5000 -n -Pn --open 172.17.0.2
```

---

## ⚡ FASE 2: EXPLOTACIÓN (ACCESO INICIAL)

### 🌐 InspecciónWeb y Descubrimiento
Al interactuar con el servidor HTTP (`http://172.17.0.2`), nos encontramos con un portal corporativo de la empresa "ACME" el cual nos informa estar fuera de linea temporalmente.

![Pagina WEB ACME](Capturas/Pagina%20WEB%20ACME.png)

Menciona que la administración se hace por SSH. Esa es la pista: el servidor SSH muestra un banner, y conviene leerlo.

> ℹ️ **Credenciales en el banner SSH**
> Un banner SSH es el mensaje que el servidor envía a cualquier cliente antes de pedir la contraseña. Aquí está mal configurado: filtra credenciales. Nos conectamos con cualquier nombre de usuario ya que el banner sale antes de autenticar

![Credenciales filtradas](Capturas/Credenciales%20filtradas.png)

### 💥 Acceso por SSH

Al colocar la contraseña filtrada, tenemos acceso como `usuario`

![Acceso como Usuario](Capturas/Acceso%20como%20Usuario.png)

---

## 👑 FASE 3: ESCALADA DE PRIVILEGIOS SUID

`usuario` no tiene permisos de sudo.

![Sin permisos de sudo](Capturas/Sin%20permisos%20de%20sudo.png)

Entonces, probaremos un comando para ver si tenemos algún binario que podamos ejecutar con permisos SUID de la siguiente manera:

```bash
find / -perm -4000 -ls 2>/dev/null
```

![SUID](Capturas/SUID.png)

> ✅ **Permiso Encontrado**
> Efectivamente existe /usr/bin/bash y /usr/bin/dash, ambos binarios explotables para lanzarnos una shell privilegiada. Procedemos a ejecutar cualquiera de los dos comandos  para la escalada de privilegios (bash -p o dash -p)

Validamos de inmediato los cambios de rol en la consola activa:
```bash
whoami
```

![ROOT](Capturas/ROOT.png)

```bash
# Output de la terminal comprometida:
root
```

> ✅ ****MÁQUINA COMPLETADA****
> Hemos vulnerado exitosamente la máquina **ACME** obteniendo credenciales mediante el banner SSH y posterior escalda mediante SUID a un usuario con privilegios de **`root`**.

---

## 🛠️ FASE 4: VULNERABILIDADES Y MITIGACIONES

>[!Danger] CWE-250

>[!Danger] CRITICO
>Escalada a root por bit SUID en /usr/bin/bash

>[!Todo] Mitigación
>Quitar el SUID y auditar periódicamente los binarios con SUID.

>[!Warning] CWE-200

>[!Warning] ALTA
>Al ingresar al servidor, el banner SSH brinda un usuario y una contraseña previo a la autenticación para cualquier persona que se conecte. 

>[!Todo] Mitigación
>Enviar cualquier tipo de credencial por un canal seguro y no incluir estos en los banners