# SAD · Temas 1-6 — Fundamentos de seguridad y alta disponibilidad

Manual de consulta, resumido del temario. Orden: propiedades de seguridad → disponibilidad → vulnerabilidades → amenazas → seguridad física → SAI.

---

## 1. Propiedades de seguridad (Tema 1)

| Propiedad | Qué garantiza | Se rompe cuando... |
|---|---|---|
| **Confidencialidad** | La información solo la ve quien está autorizado | Alguien accede o la roba sin permiso |
| **Integridad** | La información no se altera por causas involuntarias (o sin autorización) | Se modifica o destruye |
| **Disponibilidad** | El sistema responde con las mínimas pausas | Un servicio deja de funcionar (DoS, corte eléctrico...) |
| **Fiabilidad** | El sistema ofrece la información sin pausas entre peticiones; funciona como se espera | Funcionamiento incorrecto o inesperado |

Las tres primeras forman la **tríada CIA** (*Confidentiality, Integrity, Availability*).

### Disponibilidad en «nueves»

| Nueves | % de disponibilidad | Parada máxima aprox. al año |
|---|---|---|
| 2 nueves | 99 % | ~3,65 días |
| 3 nueves | 99,9 % | ~8,76 horas |
| 4 nueves | 99,99 % | ~52 minutos |
| 5 nueves | 99,999 % | ~5 minutos |

(Las paradas anuales son cálculo propio, orientativo.)

### Niveles TIER de un centro de datos

| Nivel | Idea | Frase para recordarlo |
|---|---|---|
| **TIER I** — infraestructura básica | Una sola vía de alimentación y refrigeración; casi nada redundante; el mantenimiento puede exigir parar | «Tengo lo necesario, pero si algo importante falla, puedo tener que parar.» |
| **TIER II** — componentes redundantes | Hay SAI, generadores y refrigeración de reserva, pero **una única vía de distribución** | «Tengo componentes de reserva, pero no todo está duplicado.» |
| **TIER III** — mantenimiento simultáneo | **Varias vías** de distribución; se puede hacer mantenimiento sin apagar los sistemas | «Puedo mantener parte de la infraestructura sin apagar el CPD.» |
| **TIER IV** — tolerancia a fallos | Sistemas independientes y redundantes; un fallo de un componente crítico no interrumpe el servicio | «Si falla un componente crítico, el sistema debe poder continuar.» |

```
DATA CENTER TIER III
     SERVIDORES
         │
   ┌─────┴─────┐
 RED A       RED B
   │           │
 UPS A       UPS B       → si se hace mantenimiento en uno, el otro sigue dando servicio
```

---

## 2. Elementos vulnerables (Tema 2)

**Vulnerabilidad:** debilidad o defecto de un sistema que puede ser aprovechado por una amenaza y comprometer su confidencialidad, integridad o disponibilidad. **Una vulnerabilidad no es un fallo**: es el punto débil; el fallo es lo que pasa si se aprovecha.

| Concepto | Qué es | Ejemplo |
|---|---|---|
| **Activo** | Lo que hay que proteger | Servidor y datos de clientes |
| **Vulnerabilidad** | Debilidad del sistema | Software sin actualizar |
| **Amenaza** | Circunstancia o agente que puede causar daño | Atacante, malware, incendio |
| **Ataque** | Acción que aprovecha una vulnerabilidad | Explotar el fallo para instalar malware |
| **Incidente** | Cuando el problema de seguridad se materializa | Servidor comprometido |
| **Impacto** | Consecuencia del incidente | Pérdida de datos o de dinero |
| **Riesgo** | Posibilidad de que una amenaza aproveche una vulnerabilidad y cause daño | Pérdida de información |

```
ACTIVO → VULNERABILIDAD → (la aprovecha) AMENAZA → ATAQUE → INCIDENTE → IMPACTO → RIESGO
```

Una vulnerabilidad por sí sola no implica que haya incidente, pero **aumenta el riesgo**.

### Hardware

| Elemento | Qué puede fallar | Medidas |
|---|---|---|
| Alimentación eléctrica | Apagados, pérdida de datos, corrupción del sistema de archivos | SAI, fuentes redundantes, grupo electrógeno, protección contra sobretensiones |
| Placa base | Avería total, fallos intermitentes | Mantenimiento preventivo, monitorización, repuestos |
| Memoria RAM | Bloqueos, reinicios, corrupción; errores intermitentes | **ECC** en servidores (detecta y corrige errores) |
| Discos | Sectores defectuosos, fallo mecánico, desgaste del SSD | **RAID**, monitorización, copias |
| Tarjetas de expansión | Fallo de red, controladoras... | NIC y controladoras redundantes, *hot swap* |
| Interconexiones | Cables y conectores | Varios caminos de red |

```
SERVIDOR ───── CABLE ───── SWITCH     (un solo cable = posible SPOF)
```

**SPOF** (*Single Point of Failure*): componente único cuyo fallo para todo el servicio. Se elimina con redundancia.

```
RAID 1:   DATOS
            │
         ┌──┴──┐
       DISCO  DISCO     si falla uno, el otro conserva una copia
         A      B
```

> **RAID no sustituye a las copias de seguridad.** RAID protege contra el fallo de un disco; la copia permite recuperar lo borrado, modificado o cifrado por ransomware, y sobrevive a un incendio si está fuera.

### Software, firmware y configuración

- **Sistema operativo:** parchear, pero **probando antes**: entorno de pruebas → validación → producción.
- **Aplicaciones:** errores de programación, validación incorrecta de datos, contraseñas mal guardadas, desbordamientos de memoria. Medidas: actualizar, desarrollo seguro, auditorías, mínimo privilegio.
- **Firmware** (routers, switches, tarjetas de red, discos, IoT): también tiene vulnerabilidades. No basta con actualizar el sistema operativo.
- **Configuración:** un sistema actualizado puede ser vulnerable por mala configuración (servicios o puertos innecesarios, contraseñas débiles, permisos excesivos, cuentas sin uso, valores por defecto, falta de cifrado). Principio: **reducir la superficie de ataque**.

### Datos

Se protegen con defensa en profundidad: permisos, control de acceso, autenticación y autorización, cifrado, copias de seguridad, RAID y redundancia, antivirus/EDR, firewall, segmentación, monitorización, actualizaciones y plan de recuperación ante desastres.

| Amenaza a los datos | Ejemplo |
|---|---|
| Fallo hardware | Un disco se avería |
| Error humano | Borrar una base de datos |
| Malware / ransomware | Cifrar los datos y pedir rescate |
| Robo | Obtener información confidencial |
| Catástrofe | Incendio, inundación |
| Fallo de software | Una aplicación corrompe la base de datos |

---

## 3. Análisis de vulnerabilidades (Tema 3)

### Consecuencias de una vulnerabilidad

| Consecuencia | Qué ocurre | Propiedad que rompe |
|---|---|---|
| Escalada de privilegios | Un usuario normal consigue permisos de administrador/root | Todas (control total) |
| Ejecución de código | El atacante ejecuta código en el equipo (malware, usuarios nuevos, ataques a otros) | Todas |
| Robo de información | Acceso a contraseñas, datos personales, bases de datos, claves | Confidencialidad |
| Modificación o destrucción de información | Se altera o borra información | Integridad |
| Denegación de servicio | Un servicio deja de funcionar | Disponibilidad |

### Ciclo de vida de una vulnerabilidad

```
DESCUBRIMIENTO → ANÁLISIS → PUBLICACIÓN → IDENTIFICACIÓN (CVE) → EVALUACIÓN DEL RIESGO
              → CORRECCIÓN / MITIGACIÓN → ACTUALIZACIÓN → VERIFICACIÓN
```

### Los cuatro estándares

| Siglas | Nombre | Qué hace | Pregunta que responde |
|---|---|---|---|
| **CVE** | Common Vulnerabilities and Exposures | Da un **identificador único** a una vulnerabilidad concreta (`CVE-AÑO-NÚMERO`) | ¿Cuál es? |
| **NVD** | National Vulnerability Database (NIST) | Añade información, productos afectados y métricas a los CVE | ¿Qué más se sabe? |
| **CVSS** | Common Vulnerability Scoring System | Puntuación numérica de **gravedad** (baja, media, alta, crítica); sirve para priorizar | ¿Qué gravedad tiene? |
| **CWE** | Common Weakness Enumeration | Clasifica **tipos de debilidad** que originan vulnerabilidades | ¿Qué tipo de debilidad es? |
| **CPE** | Common Platform Enumeration | Identifica de forma estructurada **productos y versiones** | ¿Qué producto está afectado? |

- CVE ≠ NVD: son programas distintos; la lista CVE alimenta a NVD. Los registros CVE los publican las **CNA** (*CVE Numbering Authorities*).
- **CVE identifica; CVSS valora y prioriza. CWE describe el tipo de debilidad; CVE, una vulnerabilidad concreta.**
- Ejemplo de CVSS: 2,1 → baja; 6,5 → media; 9,8 → crítica.
- Dónde consultar: Tenable (`tenable.com/cve`), INCIBE (`incibe.es/incibe-cert/alerta-temprana/avisos`), lista Top CWE de MITRE.

### Gestión de vulnerabilidades

1. **Identificar activos:** servidores, PCs, routers, switches, SO, aplicaciones, bases de datos, IoT.
2. **Descubrir vulnerabilidades:** avisos de seguridad, bases de datos, escáneres.
3. **Evaluar:** qué vulnerabilidades hay, qué sistemas afectan, gravedad e impacto.
4. **Priorizar:** gravedad, exposición, criticidad del activo, existencia de *exploits*, impacto, disponibilidad de parche.
5. **Remediar:** parches, actualizaciones, cambios de configuración, desactivar servicios, sustituir componentes, mitigaciones.
6. **Verificar:** que ya no está, que no ha generado problemas nuevos y que el sistema funciona.

Un **escáner de vulnerabilidades** analiza servidores, PCs y routers y genera un informe con la vulnerabilidad, el CVE, el producto y la versión, la gravedad y recomendaciones.

**Ver la versión de un servidor web** (para buscar su CVE): pestaña *Red* (F12) del navegador → primera petición → cabecera `Server`; o `curl -I https://dominio`. Se oculta con `ServerSignature Off` (Apache).

---

## 4. Amenazas (Tema 4)

**Amenaza:** circunstancia, agente o acontecimiento que puede dañar un sistema o comprometer su seguridad. **No tiene por qué ser intencionada ni una persona** (un incendio o un error humano también lo son).

### Clasificaciones

| Criterio | Tipos |
|---|---|
| Naturaleza | **Físicas** (afectan a equipos, instalaciones, entorno) · **Lógicas** (software, datos, comunicaciones, acceso) |
| Origen | **Naturales** (inundación, terremoto, tormenta, incendio, temperaturas extremas) · **Accidentales** (borrado accidental, error de configuración, fallo eléctrico, cable desconectado) · **Intencionadas** (robo, malware, phishing, DDoS, sabotaje, acceso no autorizado) |

Una misma amenaza puede estar en varias: un empleado que borra una base de datos sin querer es una amenaza **interna y accidental**.

### Amenazas físicas

| Amenaza | Medidas |
|---|---|
| Robo de equipos | Control de acceso físico, cámaras, racks cerrados, antirrobo, **cifrado**, inventario |
| Incendios e inundaciones | Detección y extinción, sensores de agua, climatización, centros de respaldo, plan de recuperación |
| Fallos eléctricos | SAI, generadores, fuentes redundantes |
| Acceso físico no autorizado | Cerraduras, tarjetas, biometría, cámaras, registro de accesos y visitantes, racks cerrados |
| Temperatura y ambiente | Climatización, sensores de temperatura, humedad, humo y agua |
| Averías de hardware | Componentes redundantes, RAID, *hot swap*, monitorización, mantenimiento preventivo, repuestos |

### Amenazas lógicas

| Amenaza | Qué es | Medidas principales |
|---|---|---|
| **Malware** | Software malicioso: virus (se adhiere a archivos), gusano (se propaga solo por la red), troyano (aparenta ser legítimo), ransomware (cifra y pide rescate), spyware (recopila información), keylogger (registra pulsaciones), bot (control remoto, botnet) | Antivirus/EDR, actualizaciones, copias |
| **Phishing** | Engaño (correo, web falsa, SMS, mensajería, redes) para obtener credenciales o datos. **El usuario forma parte de la superficie de seguridad** | Formación, MFA |
| **Robo de credenciales** | Phishing, fuerza bruta, malware, ingeniería social, reutilización de contraseñas, filtraciones | Contraseñas robustas, MFA, gestor de contraseñas, bloqueo por intentos, mínimo privilegio |
| **Explotación de vulnerabilidades** | Sistema sin actualizar → atacante → acceso | Parches, análisis de vulnerabilidades, quitar servicios innecesarios |
| **Acceso no autorizado** | Acceder sin permiso (credenciales, contraseñas débiles, vulnerabilidades, configuración, escalada) | Autenticación, autorización, auditoría |
| **DoS / DDoS** | Agotar ancho de banda, CPU, memoria, conexiones o recursos de una aplicación. En DDoS el tráfico viene de muchos equipos. Afecta sobre todo a la **disponibilidad** | Filtrado, redundancia, balanceo |
| **Modificación o borrado de datos** | Eliminar, modificar, cifrar o destruir copias accesibles (integridad, disponibilidad, a veces confidencialidad) | Permisos, copias, cifrado, monitorización |
| **Errores de configuración** | Puertos o servicios innecesarios, contraseñas por defecto, permisos excesivos, recursos compartidos sin proteger, reglas de firewall incorrectas, sin cifrado | Configuración segura |

```
Autenticación → ¿Quién eres?      Autorización → ¿Qué puedes hacer?      Auditoría → ¿Qué has hecho?
```

### El factor humano

Los usuarios y administradores provocan incidentes de forma accidental, negligente o intencionada (contraseñas débiles, compartir credenciales, abrir archivos maliciosos, software no autorizado, configurar mal un servidor, dar información confidencial). La seguridad combina **medidas técnicas, organizativas y formativas**: formación, políticas, procedimientos, mínimo privilegio, separación de funciones, auditorías y control de accesos.

### Medidas según el momento

| Fase | Objetivo | Ejemplos |
|---|---|---|
| **Prevención** | Evitar que ocurra | Firewall, antivirus/EDR, actualizaciones, control de acceso, formación, redundancia |
| **Detección** | Saber que ocurre o ha ocurrido | Monitorización, IDS/IPS, *logs*, alertas, sensores |
| **Respuesta** | Actuar | Aislar un equipo, bloquear una cuenta o IP, detener un servicio, procedimientos de emergencia |
| **Recuperación** | Volver a la normalidad | Copias de seguridad, RAID, sistemas redundantes, servidores de respaldo, recuperación ante desastres |

---

## 5. Seguridad física y ambiental (Tema 5)

**Idea clave:** la seguridad informática no empieza en una contraseña. Si alguien accede físicamente a un servidor, puede anular muchas medidas lógicas. Protege **equipos, instalaciones, personas, soportes e información**.

### Controles físicos

- **Cerraduras y puertas:** convencionales, electrónicas, reforzadas, con tarjeta.
- **Identificación y autenticación:** tarjetas de proximidad, PIN, biometría; combinar factores. Permite registrar **quién entró y cuándo**.
- **Videovigilancia:** complementa el control de acceso, no lo sustituye.
- **Registro de visitantes:** identificar, registrar entrada, responsable, **acompañar**, registrar salida.
- **Racks cerrados:** impiden manipulaciones, pero **no sustituyen** el control de acceso a la sala.

### Protección frente al robo

- **Anclajes y cables de seguridad**, armarios cerrados.
- **Inventario** (equipo, fabricante, modelo, nº de serie, MAC, ubicación, responsable, estado) para detectar desapariciones.
- **Cifrado de discos:** no evita el robo, pero protege la información si ocurre. *Protección física → evita el robo; cifrado → protege la información aunque se produzca.*
- **Copias de seguridad:** protegerlas físicamente y **no guardarlas en el mismo sitio** que los servidores (backup local, externo y/o remoto).
- **Destrucción segura de soportes:** borrado seguro, sobrescritura, cifrado previo, destrucción física.

### Incendios

Combinar **prevención, detección, extinción y recuperación**: detectores de humo y de temperatura, alarma, extinción automática por **agentes gaseosos** (sin agua sobre los equipos), extintores **adecuados para instalaciones eléctricas**, materiales y compartimentación que frenen el fuego, y **planes de evacuación** (la prioridad es proteger a las personas).

### Alimentación eléctrica

| Problema | Qué es |
|---|---|
| Corte | Pérdida total |
| Sobretensión | Tensión superior a la adecuada |
| Bajada de tensión | Tensión inferior a la necesaria |
| Pico | Aumento muy rápido de tensión |
| Microcorte | Interrupción muy corta |
| Variación de frecuencia | Alteración de la frecuencia |

```
Red → SAI → Servidor                          corte:  Red ❌ → SAI → Servidor ✓
corte prolongado:  Red ❌ → SAI → Grupo electrógeno → SAI → Servidores
```

El SAI mantiene la alimentación mientras el generador arranca. Además: protectores de sobretensión, fuentes de alimentación redundantes (Fuente A → circuito A, Fuente B → circuito B) y así se evitan SPOF.

### Condiciones ambientales

- **Temperatura:** el exceso provoca sobrecalentamiento, apagados y menor vida útil → climatización.
- **Humedad:** demasiada, condensación y corrosión; demasiado poca, **electricidad estática**.
- **Polvo:** obstruye ventiladores, disipadores y filtros.
- **Sensores:** temperatura, humedad, humo, agua y estado de la climatización, con alertas.
- **Agua:** inundaciones, fugas, climatización, extinción, plantas superiores. Sensores, elevar equipos, drenaje, y elegir bien la ubicación del CPD: **la ubicación es una decisión de seguridad**.

### El CPD

- **Seguridad perimetral:** puertas, tarjetas, biometría, cámaras, personal.
- **Seguridad de la sala** por niveles: `Edificio → Zona restringida → CPD → Rack → Equipo`; cuanto más crítico, más control.
- **Redundancia de instalaciones:** alimentación, SAI, generadores, climatización, conectividad, servidores, almacenamiento.
- **Monitorización:** fallos de alimentación, temperatura, humedad, humo, accesos no autorizados, fallos de hardware.

**RAID, redundancia y copias de seguridad no son equivalentes.** RAID mantiene un servidor tras fallar un disco, pero no evita que un incendio lo destruya todo. Las medidas se combinan (**defensa en profundidad**):

```
Control de acceso físico → Sala protegida → Rack cerrado → Discos redundantes
→ SAI y alimentación redundante → Monitorización ambiental → Firewall y control de acceso lógico
→ Copias de seguridad → Plan de recuperación ante desastres
```

### Resumen amenaza → medida

| Amenaza | Consecuencia | Medidas |
|---|---|---|
| Robo | Pérdida de equipos y datos | Control de acceso, anclajes, cifrado |
| Acceso físico no autorizado | Manipulación o robo | Tarjetas, biometría, cámaras |
| Incendio | Destrucción de equipos | Detectores, extinción, planes de emergencia |
| Corte eléctrico | Caída de servicios | SAI, generador, redundancia |
| Sobretensión | Daños en hardware | Protección eléctrica |
| Temperatura elevada | Sobrecalentamiento | Climatización y monitorización |
| Humedad | Corrosión / condensación | Control ambiental |
| Agua | Daños en equipos | Sensores y ubicación adecuada |
| Fallo de hardware | Interrupción | Redundancia, RAID, repuestos |
| Robo de soportes | Exposición de información | Cifrado y destrucción segura |

---

## 6. Sistemas de alimentación ininterrumpida, SAI/UPS (Tema 6)

Un SAI da alimentación temporal por batería y, según su tecnología, filtra o corrige perturbaciones. Objetivo: evitar el apagado inmediato y dar **tiempo para recuperar el suministro o apagar de forma controlada**.

### Tipos

| Característica | Offline / Standby | Line-Interactive | Online (doble conversión) |
|---|---|---|---|
| Funcionamiento | La red alimenta el equipo; ante un fallo, conmuta a batería | Corrige variaciones de tensión sin usar la batería | CA → CC → CA: el equipo siempre se alimenta del inversor |
| Coste | Bajo | Medio | Alto |
| Protección | Básica | Media | Alta |
| Regulación de tensión | Limitada | Sí | Sí |
| Tiempo de transferencia | Sí (pequeño) | Sí | Prácticamente inexistente |
| Uso habitual | Equipos personales | Oficinas y servidores pequeños | CPD y sistemas críticos |

### Magnitudes y fórmulas

| Magnitud | Unidad | Qué es |
|---|---|---|
| Potencia aparente | **VA** (kVA = 1000 VA) | Capacidad que debe soportar el SAI |
| Potencia activa | **W** | La que realmente consume la carga |
| Energía | **Wh** (vatios-hora, no W/h) | Cantidad de energía almacenada o consumida |
| Capacidad de batería | **Ah** | Capacidad eléctrica nominal |
| Factor de potencia | **FP** | Relación entre W y VA |

```
P(W)  = S(VA) × FP          S(VA) = P(W) / FP          FP = P(W) / S(VA)
E(Wh) ≈ V × Ah              Autonomía (h) ≈ E(Wh) / P(W)
```

No asumir siempre FP = 0,75: el fabricante puede especificar los vatios y el FP.

### Ejemplos resueltos

**SAI de 2000 VA con FP 0,75** → P = 2000 × 0,75 = **1500 W**.

**Elegir un SAI** (tres servidores: 400 + 350 + 500 = 1250 W, FP 0,75):
S = 1250 / 0,75 = 1666,7 VA → con **margen del 20 %**: 1667 × 1,2 ≈ **2000 VA**. Se busca un modelo comercial igual o superior y se comprueba también que sus vatios bastan. El 20 % es un criterio orientativo.

**Dimensionamiento completo** (servidor web 400 W + base de datos 500 W + switch 100 W + firewall 50 W + NAS 200 W = 1250 W; FP 0,8):
S = 1250 / 0,8 = 1562,5 VA → × 1,2 = **1875 VA** → un SAI comercial de capacidad superior.

**Autonomía** (batería 12 V, 7 Ah, carga 150 W):
E = 12 × 7 = 84 Wh → t = 84 / 150 = 0,56 h ≈ **34 min** (ideal; la real será menor por pérdidas). Si se apaga un monitor de 50 W (carga 100 W): 84 / 100 = 0,84 h ≈ **50 min**.

### Procedimiento de dimensionamiento

```
1 Identificar equipos → 2 Calcular su consumo → 3 Sumar la potencia total (W)
→ 4 Pasar a VA con el FP → 5 Añadir margen → 6 Fijar la autonomía necesaria
→ 7 Elegir el tipo de SAI → 8 Comprobar las especificaciones reales del fabricante
```

### Autonomía real

Depende de la **potencia consumida** (más consumo, menos autonomía), la **capacidad** de las baterías, la **eficiencia** del SAI, el **estado y antigüedad** de la batería (revisarla y sustituirla) y la **temperatura**.

### Apagado controlado

El SAI no está para mantener el CPD horas, sino para dar tiempo a avisar al administrador y apagar con orden. Se comunica con los equipos por USB o red:

```
Corte → SAI entra en funcionamiento → el servidor detecta que va con batería
→ arranca un temporizador → si no vuelve la red: apagado controlado
```

### SAI ≠ alta disponibilidad

Un SAI mantiene el servidor durante un corte, pero no soluciona una avería de hardware. La alta disponibilidad combina **SAI, redundancia eléctrica, fuentes redundantes, RAID, servidores redundantes, clústeres, balanceadores, copias de seguridad y recuperación**, y elimina los SPOF.
