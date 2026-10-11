| -Plataforma         | Docker Labs |
| ------------------- | ----------- |
| Dificultad          | Muy Fácil   |
| IP de la Máquina    | 172.17.0.2  |
| Fecha de resolución | 03/10/2026  |
# 📑 WRITEUP: BREAKMYSSH

> ℹ️ ****RESUMEN****
> - **IP Objetivo:** `172.17.0.2`
> - **Sistema Operativo:** Linux 🐧
> - **Descripción:**  Laboratorio para practicar ataques de fuerza bruta con Hydra contra el servicio SSH. 🔑
> - **Privilegios Obtenidos:** `root` 👑

---

## 🎯 FASE 1: RECONOCIMIENTO Y ENUMERACIÓN

### 💻 Conectividad Inicial
Determinamos el Sistema Operativo analizando el Time To Live (TTL) mediante un paquete ICMP:
```bash
ping -c 1 172.17.0.2
```

![PING a IP objetivo](Capturas/PING%20a%20IP%20objetivo.png)

* **Diagnóstico:** TTL = `64`. Confirmamos que la máquina víctima corresponde a un entorno **Linux**.

### 🔍 Mapeo de Puertos y Servicios
Lanzamos un análisis exhaustivo con **Nmap** para indexar todos los puertos abiertos en el host.

![Scan NMAP](Capturas/Scan%20NMAP.png)

| Puerto   | Servicio | Versión Detectada | Estado / Nota                                     |
| :------- | :------- | :---------------- | :------------------------------------------------ |
| `22/TCP` | SSH      | OpenSSH 7.7       | 🔴 Único Vector Abierto (Objetivo de Brute Force) |

#### Comandos de Nmap utilizados:
```bash
sudo nmap -p- -sS -sV --min-rate 5000 -n -Pn --open 172.17.0.2
```

> 📝 **Observación de Enumeración**
> El escaneo confirma que el servidor web (puerto 80) u otros servicios comunes están ausentes. Toda la superficie de ataque se reduce exclusivamente al servicio de acceso remoto **SSH**.

---

## ⚡ FASE 2: EXPLOTACIÓN (ACCESO INICIAL)

### 🔑 Ataque de Fuerza Bruta Dirigido (SSH)
Ante la ausencia de vectores web u otros servicios explotables, la vía de entrada pasa por realizar fuerza bruta de credenciales. Probamos con nombres de usuario habituales en este tipo de laboratorios introductorios (como `root`) contra el diccionario universal de contraseñas `rockyou.txt` empleando **Hydra**:

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

![Ataque Hydra a root](Capturas/Ataque%20Hydra%20a%20root.png)

> ✅ ****Credenciales Críticas Obtenidas****
> El ataque por diccionario arroja un resultado positivo inmediato debido al uso de claves sumamente débiles expuestas:
> - **Usuario:** `root`
> - **Contraseña:** `estrella`

Nos conectamos directamente usando el cliente de terminal oficial:
```bash
ssh root@172.17.0.2
```

---

## 👑 FASE 3: ESCALADA DE PRIVILEGIOS

Debido a que el ataque de fuerza bruta se enfocó y validó con éxito sobre el perfil del administrador máximo del sistema operativo (`root`), logramos evadir por completo cualquier restricción o entorno limitado.

Procedemos a evaluar la identidad actual en la shell establecida:
```bash
whoami
```

![ROOT](Capturas/ROOT.png)

```bash
# Output directo de la terminal comprometida:
root
```

> ✅ ****MÁQUINA COMPLETADA****
> No se requiere aplicar metodologías de escalada local (SUID, Sudoers o vulnerabilidades de Kernel). Al romper las credenciales del servicio SSH del superusuario, el contenedor queda 100% comprometido desde el acceso inicial.

---

## 🛠️ FASE 4: VULNERABILIDADES Y MITIGACIONES

>[!Danger] CWE-521

>[!Danger] Crítica
>La contraseña es débil y se encuentra en el listado de contraseñas y el acceso a `root` es directo, sin ninguna medida de protección o limite de intentos  

>[!Todo] Mitigación
>Configurar un software de prevención de intrusiones para bloquear temporal o permanentemente direcciones IP que fallen múltiples intentos de autenticación.
>Prohibir de forma estricta contraseñas cortas o que formen parte de diccionarios públicos masivos como `rockyou.txt`.

>[!Warning] CWE-250

>[!Warning] Alta
>Permitir el acceso directo de `root` por SSH expone la cuenta más crítica del sistema al ataque.

>[!Todo] Mitigación
>PermitRootLogin no y operar con un usuario sin privilegios.
