# Git: manual de supervivencia

Receta de cocina para el día a día. Los comandos están **en el orden en que se usan**.
Lo que va entre `< >` hay que sustituirlo por mi valor.

---

## 0. Una sola vez (preparar la cocina)

**Instalar Git**
```bash
sudo apt update && sudo apt install -y git
```
En Windows: descargar de https://git-scm.com/downloads

**Decir quién eres** (saldrá en cada commit)
```bash
git config --global user.name "Tu Nombre"
git config --global user.email "tu@correo.com"
git config --global init.defaultBranch main
```
La tercera línea hace que la rama principal se llame `main` y no `master`.

**Comprobar la configuración**
```bash
git config --list
```

**Conectar con GitHub sin contraseña (SSH)**

GitHub ya no acepta la contraseña normal por HTTPS. Lo más cómodo es una clave SSH:
```bash
ssh-keygen -t ed25519 -C "mi@correo.com"
cat ~/.ssh/id_ed25519.pub
```
Copio el resultado en GitHub → *Settings → SSH and GPG keys → New SSH key*. Después:
```bash
ssh -T git@github.com
```
Debe saludarme con mi usuario. Si es así, ya está.

---

## 1. Empezar un proyecto

### Opción A: el repositorio ya existe en GitHub (la más habitual)
```bash
git clone git@github.com:<usuario>/<repo>.git
cd <repo>
```
Descarga el proyecto entero y deja el remoto ya conectado.

### Opción B: tengo una carpeta local y quiero subirla
```bash
cd <mi-carpeta>
git init
git remote add origin git@github.com:<usuario>/<repo>.git
git remote -v
```
`git init` crea el repositorio, `remote add` lo une con GitHub y `remote -v` comprueba que quedó bien.

---

## 2. El ciclo de cada día

Repetir esto siempre, en este orden:

```bash
git pull
```
**1. Traer** lo que hayan subido los demás. Hacerlo siempre antes de empezar a trabajar.

*(trabajo: edito, creo, borro ficheros...)*

```bash
git status
```
**2. Mirar** qué ha cambiado. El comando que más vaoy a usar: ante la duda, `git status`.

```bash
git diff
```
**3. Ver** qué líneas cambiaron (opcional, pero evita sustos).

```bash
git add <fichero>
git add -A
```
**4. Preparar** los cambios para guardarlos. Un fichero concreto, o todo de golpe con `-A`.

```bash
git commit -m "Descripción breve de lo que he hecho"
```
**5. Guardar** una "foto" en mi repositorio local. El mensaje debe decir qué hice, no "cambios" ni "asdf".

```bash
git push
```
**6. Subir** los commits a GitHub.

> Primera vez que subo un repositorio nuevo (Opción B): `git push -u origin main`.
> Después basta con `git push`.

---

## 3. Ramas (trabajar sin romper lo que funciona)

```bash
git switch -c <nueva-rama>
```
Crea una rama y se pasa a ella.

*(trabajo y hago commits como siempre)*

> En tutoriales antiguos: `git checkout -b <nueva-rama>` hace lo mismo (ver nota al final de la sección 4).

```bash
git push -u origin <nueva-rama>
```
Subo la rama a GitHub (solo la primera vez uso `-u`).

```bash
git switch main
git pull
git merge <nueva-rama>
git push
```
Cuando esté lista: volver a `main`, actualizar, **fusionar** la rama y subir.

```bash
git branch -d <nueva-rama>
```
Borra la rama ya fusionada.

Otros: `git branch` lista las ramas y `git switch <rama>` cambia de una a otra.

> En equipo, lo normal es subir la rama y abrir una **Pull Request** en GitHub para que otro la revise antes de fusionar.

---

## 4. Me he equivocado (deshacer)

| Situación | Comando |
|---|---|
| Quiero corregir el **mensaje** del último commit (aún sin `push`) | `git commit --amend -m "Nuevo mensaje"` |
| Me olvidé un fichero en el último commit (aún sin `push`) | `git add <fichero>` y `git commit --amend --no-edit` |
| He hecho `add` y me arrepiento | `git restore --staged <fichero>` |
| Quiero **descartar** mis cambios en un fichero | `git restore <fichero>` |
| Un commit **ya subido** estaba mal | `git revert <hash>` (crea un commit que lo anula) |

⚠️ `git restore <fichero>` **no se puede deshacer**: los cambios se pierden para siempre.
⚠️ No usar `--amend` sobre commits ya subidos: reescribo la historia y lio a todo el equipo.

**Guardar cambios a medias para cambiar de rama o hacer `pull`:**
```bash
git stash
git stash pop
```
`stash` los aparca y `pop` los recupera.

### Nota: `checkout` vs `switch` / `restore`

En tutoriales y foros veré `git checkout` para casi todo. Es el comando antiguo: hacía **dos cosas distintas** (cambiar de rama y recuperar ficheros), y por eso confundía. Desde Git 2.23 (2019) se separó en dos comandos más claros. `checkout` **sigue funcionando** y no está retirado, así que si copio un comando antiguo, me valdrá.

| Para qué | Antiguo (`checkout`) | Actual |
|---|---|---|
| Cambiar de rama | `git checkout <rama>` | `git switch <rama>` |
| Crear rama y cambiarse a ella | `git checkout -b <rama>` | `git switch -c <rama>` |
| Descartar cambios de un fichero | `git checkout -- <fichero>` | `git restore <fichero>` |
| Sacar un fichero del *staging* | `git reset HEAD <fichero>` | `git restore --staged <fichero>` |

En este manual uso los actuales. Si veo `checkout` en otro sitio, busco en la tabla su equivalente.
---

## 5. Conflictos (cuando dos tocan lo mismo)

Pasa al hacer `pull` o `merge`. Git avisa con `CONFLICT` y marca el fichero así:

```text
<<<<<<< HEAD
tu versión
=======
la versión del otro
>>>>>>> origin/main
```

Para resolverlo:
1. Abro el fichero y **dejo solo lo que debe quedar**, borrando las tres líneas de marcas (`<<<<<<<`, `=======`, `>>>>>>>`).
2. Guardo y ejecuto:
```bash
git add <fichero>
git commit -m "Resuelvo conflicto en <fichero>"
git push
```

No es el fin del mundo: es Git preguntando "¿cuál de las dos versiones vale?".

---

## 6. Otras operaciones habituales

**Borrar un fichero del proyecto**
```bash
git rm <fichero>
git commit -m "Elimino <fichero>"
```

**Renombrar o mover un fichero**
```bash
git mv <viejo> <nuevo>
git commit -m "Renombro <viejo> a <nuevo>"
```

**Ver el historial**
```bash
git log --oneline
git log --oneline --graph
```

**Ignorar ficheros** (logs, contraseñas, temporales...): crear un fichero `.gitignore` en la raíz:
```text
*.log
*.class
.env
```

---

## Chuleta rápida

```text
git pull → (trabajar) → git status → git add -A → git commit -m "..." → git push
```

**Reglas de oro**
1. `git pull` antes de empezar, `git push` al terminar.
2. `git status` ante cualquier duda.
3. Commits pequeños y con mensaje claro.
4. Nunca subir contraseñas ni claves privadas (`.gitignore` es mi amigo).
5. Si algo sale raro, **no improviso**: paro, leo el mensaje de error y miro `git status`.

---
