# SAD · Tema 7 — Criptografía y GnuPG

Manual de consulta: conceptos mínimos y, después, los comandos de GPG en el orden en que se usan. Los `< >` son valores que hay que sustituir.

---

## 1. Conceptos

### Qué hace cada técnica

| Necesito... | Uso | Propiedad |
|---|---|---|
| Que nadie lea el mensaje | **Cifrar** | Confidencialidad |
| Saber que no ha cambiado | **Hash** / firma | Integridad |
| Saber quién lo envía | **Firma digital** | Autenticidad y no repudio |

> **Firmar ≠ cifrar.** Un mensaje firmado va en claro: cualquiera puede leerlo, pero se comprueba quién lo firmó y que no se ha tocado. Para ambas cosas hay que **firmar y cifrar**.

### Simétrica vs asimétrica

| | Simétrica | Asimétrica |
|---|---|---|
| Claves | **Una** clave compartida | **Par**: pública + privada |
| Velocidad | Rápida | Lenta |
| Problema | ¿Cómo comparto la clave? | Gestionar y verificar claves públicas |
| Algoritmos | AES, ChaCha20-Poly1305 | RSA, Diffie-Hellman, ECDSA, EdDSA (Ed25519) |
| Uso típico | Cifrar datos (discos, ficheros, tráfico) | Intercambio de claves, firmas, identidad |

### Reglas del par de claves

| Quiero... | Se usa... |
|---|---|
| **Cifrar** para alguien | su clave **pública** |
| **Descifrar** lo que me envían | mi clave **privada** |
| **Firmar** | mi clave **privada** |
| **Verificar** una firma | la clave **pública** del firmante |

La privada no se comparte nunca; la pública se reparte libremente.

### Cifrado híbrido (así funciona HTTPS y GPG)

```
1. Se genera una clave simétrica aleatoria (de sesión)
2. Los datos se cifran con la clave simétrica         → rápido
3. La clave simétrica se cifra con la clave pública   → seguro
4. El receptor descifra la clave de sesión con su privada y luego los datos
```

### Hash

Función que convierte cualquier dato en una huella de tamaño fijo. Mismo dato → mismo hash; un cambio mínimo → hash totalmente distinto; no se puede revertir.

| Algoritmo | Estado |
|---|---|
| **SHA-256** y familia SHA-2/SHA-3 | Válido |
| MD5, SHA-1 | **Débiles**, no usar para seguridad |

```bash
sha256sum <fichero>            # calcular
sha256sum -c <fichero>.sha256  # verificar
```

### Certificados y CA

Un **certificado** une una identidad con una clave pública y lo firma una **CA** (autoridad de certificación) en la que se confía. Es lo que permite confiar en una web (HTTPS) sin conocer previamente su clave. GPG usa otro modelo: la confianza la decide cada usuario (**red de confianza**).

---

## 2. GnuPG — preparación

```bash
gpg --version                     # comprobar instalación
sudo apt install gnupg            # si falta
ls -ld ~/.gnupg                   # debe ser 700 y de tu usuario
chmod 700 ~/.gnupg                # si da aviso "unsafe ownership/permissions"
```

---

## 3. Crear y gestionar claves

```bash
gpg --full-generate-key           # asistente completo (tipo, tamaño, caducidad, nombre, correo, contraseña)
gpg --gen-key                     # asistente rápido (valores por defecto)
```

Recomendado: **ECC (Ed25519)** o **RSA 3072/4096**, con caducidad (por ejemplo `1y`) y una frase de paso robusta.

| Capacidad | Letra | Para qué |
|---|---|---|
| Sign | `S` | Firmar |
| Certify | `C` | Certificar otras claves/identidades (clave primaria) |
| Encrypt | `E` | Cifrar (suele ser una **subclave**) |
| Authenticate | `A` | Autenticarse (p. ej. SSH) |

### Listar y consultar

```bash
gpg -k                            # claves públicas (--list-keys)
gpg -K                            # claves privadas (--list-secret-keys)
gpg --fingerprint <correo>        # huella completa
gpg --list-keys --keyid-format long
```

### Exportar e importar

```bash
gpg --export --armor <correo> > <nombre>_publica.asc          # clave pública (texto ASCII)
gpg --export-secret-keys --armor <correo> > privada.asc        # privada: solo para copia de seguridad, guardarla a buen recaudo
gpg --import <nombre>_publica.asc                              # importar la pública de otra persona
```

### Verificar la huella antes de confiar

Tras importar una clave, **comprueba la huella por otro canal** (en persona, por teléfono) antes de fiarte.

```bash
gpg --fingerprint <correo>
```

### Confiar en una clave

```bash
gpg --edit-key <correo>
  gpg> trust                      # elegir nivel (1-5)
  gpg> quit
```

| Nivel | Significado |
|---|---|
| 1 | No sé |
| 2 | No confío |
| 3 | Confío marginalmente |
| 4 | Confío plenamente |
| 5 | Confío absolutamente (**mis propias claves**) |

Aviso típico al cifrar para una clave importada: *«No se tiene seguridad de que la clave pertenezca a la persona que se nombra»* (key not certified). Si ya has comprobado la huella, responde **`s`** (sí); lo correcto es firmar/confiar en la clave.

### Revocación (hacerla desde el principio)

```bash
gpg --gen-revoke <correo> > revocacion.asc    # certificado de revocación; guardarlo fuera del equipo
gpg --import revocacion.asc                    # aplicarlo si la clave se pierde o compromete
```

### Eliminar claves

```bash
gpg --delete-secret-keys <correo>   # primero la privada
gpg --delete-keys <correo>          # luego la pública
```

---

## 4. Cifrar y descifrar

### Con clave asimétrica (para otra persona)

```bash
gpg --encrypt -r <correo> <fichero>              # genera <fichero>.gpg (binario)
gpg --encrypt --armor -r <correo> <fichero>      # genera <fichero>.asc (texto)
gpg --decrypt <fichero>.gpg                      # muestra en pantalla
gpg --decrypt <fichero>.gpg > <fichero>          # guarda en un fichero
gpg -o <salida> --decrypt <fichero>.gpg          # equivalente con -o
```

Varios destinatarios: añadir varios `-r`. Hace falta **la clave pública del destinatario** para cifrar y **tu privada** para descifrar lo que te envían.

### Con contraseña (simétrico)

```bash
gpg -c <fichero>                                 # pide contraseña, genera <fichero>.gpg
gpg --decrypt <fichero>.gpg                      # pide la misma contraseña
```

---

## 5. Firmar y verificar

| Comando | Resultado | Cuándo |
|---|---|---|
| `gpg --sign <fichero>` | `<fichero>.gpg`: contenido + firma (binario) | Firmar y empaquetar |
| `gpg --clearsign <fichero>` | `<fichero>.asc`: texto legible con firma | Mensajes de texto |
| `gpg --detach-sign <fichero>` | `<fichero>.sig`: firma **aparte** | Distribuir ficheros (ISO, instaladores) |
| `gpg --detach-sign --armor <fichero>` | `<fichero>.asc` con la firma aparte | Igual, en ASCII |

```bash
gpg --verify <fichero>.sig <fichero>   # firma separada: primero la firma, luego el fichero
gpg --verify <fichero>.asc             # clearsign / firma incluida
```

Resultados posibles:

- **`Good signature from ...`** → el fichero es el firmado y no ha cambiado.
- **`BAD signature`** → el fichero **se ha modificado** (o no es el firmado). Prueba: copia el fichero, cambia una letra y verifica la copia con la firma original → sale BAD.
- **`Can't check signature: No public key`** → falta importar la clave pública del firmante.

Aviso *«This key is not certified with a trusted signature»*: la firma es válida, pero no has marcado esa clave como de confianza.

### Firmar y cifrar a la vez

```bash
gpg --sign --encrypt -r <correo> <fichero>
gpg --sign --armor --encrypt -r <correo> <fichero>
```

Quien recibe: `gpg --decrypt <fichero>.gpg` descifra **y** verifica la firma.

> Errata típica: **firmar usa la clave privada** del firmante; **verificar usa su pública**.

---

## 6. Enviar los ficheros a otra máquina

```bash
scp -P <puerto> <fichero> <usuario>@<ip>:<ruta>     # -P en MAYÚSCULA (ssh usa -p)
```

Comprobar que el fichero llegó íntegro con `sha256sum` en ambos extremos.

---

## 7. Errores frecuentes

| Mensaje | Causa y solución |
|---|---|
| `unsafe ownership / permissions on homedir` | `chmod 700 ~/.gnupg` y comprobar que es de tu usuario |
| `Host desconocido` / fallo de WKD al buscar o cifrar | GPG intenta localizar la clave por internet (WKD). No afecta si la clave ya está importada localmente |
| `No public key` | Falta importar la clave pública del destinatario/firmante |
| `No secret key` al descifrar | Estás en una máquina sin la privada, o el mensaje se cifró para otra clave |
| `[unknown]` en la confianza | Clave sin nivel de confianza: `gpg --edit-key` → `trust` |
| `inappropriate ioctl for device` al pedir contraseña por SSH | `export GPG_TTY=$(tty)` |
| Clave caducada o revocada | Ampliar caducidad con `gpg --edit-key` → `expire`; si está revocada, crear una nueva |

---

## 8. Uso no interactivo (scripts)

```bash
gpg --batch --yes --pinentry-mode loopback --passphrase '<contraseña>' --symmetric <fichero>
```

Evita dejar contraseñas en el historial o en scripts compartidos; mejor `--passphrase-file`.

---

## 9. Chuleta

```bash
gpg --full-generate-key                              # crear par de claves
gpg -k ; gpg -K                                      # listar públicas / privadas
gpg --fingerprint <correo>                           # huella
gpg --export --armor <correo> > pub.asc              # exportar pública
gpg --import pub.asc                                 # importar
gpg --edit-key <correo>  →  trust                    # nivel de confianza
gpg --gen-revoke <correo> > rev.asc                  # revocación
gpg --encrypt --armor -r <correo> <f>                # cifrar para otra persona
gpg --decrypt <f>.asc > <f>                          # descifrar
gpg -c <f>                                           # cifrar con contraseña
gpg --detach-sign --armor <f>                        # firma aparte
gpg --verify <f>.asc <f>                             # verificar
gpg --sign --encrypt -r <correo> <f>                 # firmar + cifrar
sha256sum <f>                                        # hash
```
