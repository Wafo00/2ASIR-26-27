# Git: manual de supervivencia

Receta de cocina para el día a día. Los comandos están **en el orden en que se usan**.
Lo que va entre `< >` hay que sustituirlo por vuestro valor.

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
ssh-keygen -t ed25519 -C "tu@correo.com"
cat ~/.ssh/id_ed25519.pub
```
Copiad el resultado en GitHub → *Settings → SSH and GPG keys → New SSH key*. Después:
```bash
ssh -T git@github.com
```
Debe saludaros con vuestro usuario. Si es así, ya está.

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

Repetid esto siempre, en este orden:

```bash
git pull
```
**1. Traer** lo que hayan subido los demás. Hacedlo siempre antes de empezar a trabajar.

*(trabajáis: editáis, creáis, borráis ficheros...)*

```bash
git status
```
**2. Mirar** qué ha cambiado. El comando que más vais a usar: ante la duda, `git status`.

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
**5. Guardar** una "foto" en vuestro repositorio local. El mensaje debe decir qué hicisteis, no "cambios" ni "asdf".

```bash
git push
```
**6. Subir** los commits a GitHub.

> Primera vez que subís un repositorio nuevo (Opción B): `git push -u origin main`.
> Después basta con `git push`.

---

## 3. Ramas (trabajar sin romper lo que funciona)

```bash
git switch -c <nueva-rama>
```
Crea una rama y se pasa a ella.

*(trabajáis y hacéis commits como siempre)*

```bash
git push -u origin <nueva-rama>
```
Sube la rama a GitHub (solo la primera vez usa `-u`).

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
⚠️ No uséis `--amend` sobre commits ya subidos: reescribís la historia y liáis a todo el equipo.

**Guardar cambios a medias para cambiar de rama o hacer `pull`:**
```bash
git stash
git stash pop
```
`stash` los aparca y `pop` los recupera.

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
1. Abrid el fichero y **dejad solo lo que debe quedar**, borrando las tres líneas de marcas (`<<<<<<<`, `=======`, `>>>>>>>`).
2. Guardad y ejecutad:
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
4. Nunca subáis contraseñas ni claves privadas (`.gitignore` es vuestro amigo).
5. Si algo sale raro, **no improvises**: parad, leed el mensaje de error y mirad `git status`.

---

*Basado en el taller de introducción a Git y GitHub de José Juan Sánchez (CC BY-SA 4.0), ampliado y adaptado.*
