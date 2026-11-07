# Chuleta: Oracle XE + SQL Developer en Windows 11

Resumen exprés del manual completo ([Manual_Oracle_XE.md](Manual_Oracle_XE.md)). Rutas siempre **sin espacios, tildes ni ñ**.

## 1. Antes de instalar (PowerShell)

```
winver
(Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory / 1GB
Get-PSDrive C
net localgroup Administradores
echo $env:ORACLE_HOME
echo $env:TNS_ADMIN
netstat -ano | findstr ":1521 :5500"
```

Las dos `echo` y el `netstat` no deben mostrar nada. Crear un **punto de restauración** (Inicio → *Crear un punto de restauración* → **Crear**).

## 2. Instalar Oracle XE

1. Descomprimir el ZIP en `C:\xe_temp` (nunca en `C:\oracle`).
2. `C:\xe_temp\setup.exe` → Siguiente → aceptar licencia.
3. Carpeta de destino: **Cambiar** → `C:\oracle\`
4. Contraseña única para SYS, SYSTEM y PDBADMIN. **Apuntarla.**
5. **Instalar** y esperar unos 10 minutos sin cancelar, aunque no se vea actividad. Permitir el firewall si lo pide.
6. Anotar los datos de la pantalla final y pulsar **Finalizar**. Después se puede borrar `C:\xe_temp`.

## 3. Comprobar el servidor (PowerShell nuevo)

```
lsnrctl status
sqlplus / as sysdba
```

Dentro de SQL\*Plus:

```
SELECT banner FROM v$version;
SHOW PDBS
SELECT dbms_xdb_config.gethttpsport FROM dual;
EXIT
```

XEPDB1 debe salir en **READ WRITE**. Web: `https://localhost:5500/em` → usuario `system`, contenedor en blanco.

## 4. Servicios en inicio manual (opcional)

`services.msc` → **Manual** en estos tres (OracleVssWriterXE no se toca):
`OracleServiceXE`, `OracleOraDB21Home1TNSListener`, `OracleOraDB21Home1MTSRecoveryService`

Terminal **como administrador**:

```
net start OracleOraDB21Home1TNSListener
net start OracleServiceXE
net start OracleOraDB21Home1MTSRecoveryService
```

Para parar, `net stop` en orden inverso. Scripts en `C:\oracle\scripts` (sección 6.3 del manual). Para consultar el estado: `Get-Service Oracle*` (en PowerShell, `sc` a secas no funciona: usar `sc.exe`).

## 5. Usuario de trabajo

```
sqlplus system@localhost:1521/XEPDB1
SHOW CON_NAME
```

Debe salir **XEPDB1**. Después:

```
CREATE USER jordi IDENTIFIED BY "Provisional1"
  DEFAULT TABLESPACE users
  QUOTA UNLIMITED ON users;
PASSWORD jordi
GRANT CREATE SESSION, CREATE TABLE, CREATE VIEW, CREATE SEQUENCE, CREATE PROCEDURE TO jordi;
EXIT
```

Prueba rápida:

```
sqlplus jordi@localhost:1521/XEPDB1
SHOW USER
CREATE TABLE prueba (id NUMBER);
INSERT INTO prueba VALUES (1);
COMMIT;
SELECT * FROM prueba;
DROP TABLE prueba;
EXIT
```

## 6. SQL Developer

1. Descomprimir el ZIP en `C:\oracle` (crea `C:\oracle\sqldeveloper`). Acceso directo a `sqldeveloper.exe`.
2. Primer arranque: no importar preferencias.
3. **+** Nueva conexión:
   - `XE_system`: usuario `system`, host `localhost`, puerto `1521`, **Nombre del servicio** `XEPDB1`.
   - `XE_jordi`: igual, con usuario `jordi`. **Volver a marcar Nombre del servicio**, porque el formulario vuelve a SID.
4. **Probar** → *Estado: Correcto* → **Guardar** → **Conectar**.
5. En la hoja de trabajo, ejecutar con **Ctrl+Intro**:

```
SELECT user, sys_context('USERENV','CON_NAME') FROM dual;
```

Debe salir `JORDI | XEPDB1`.

## 7. Si algo falla

| Error | Causa probable y solución |
|---|---|
| ORA-12560 | Servicios parados → arrancarlos |
| ORA-12541 | Listener parado → `lsnrctl start` |
| ORA-12514 | Servicio mal escrito, o acaba de arrancar → revisar el nombre o esperar 1 minuto |
| ORA-01017 | Contraseña mal escrita, o conexión por SID → usar *Nombre del servicio* `XEPDB1` |
| ORA-01918 | Estás en CDB$ROOT → `ALTER SESSION SET CONTAINER = XEPDB1;` |
| ORA-28000 | Cuenta bloqueada → `ALTER USER jordi ACCOUNT UNLOCK;` |
| Tildes raras | `set NLS_LANG=SPANISH_SPAIN.WE8PC850` (CMD) antes de `sqlplus` |
| No reconoce `lsnrctl` o `sqlplus` | Cerrar y abrir PowerShell de nuevo |

Alert log: `C:\oracle\diag\rdbms\xe\xe\trace\alert_XE.log`
