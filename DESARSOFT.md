# Fork de DESARSOFT

Fork de [`evolution-foundation/evolution-api`](https://github.com/evolution-foundation/evolution-api) con dos objetivos: **poder vincular un teléfono nuevo**, cosa que el upstream no permite desde el 2026-07-28, y **mostrar en Chatwoot el nombre real de quien escribe en un grupo** (desde el 2026-10-27).

Se usa para capturar las líneas de WhatsApp de la oficina y espejar Ventas y Soporte en Chatwoot. El detalle del servicio (servidor, secretos, operación) vive en el vault `PCGerenteAdmin`, en `infraestructura/evolution.md`.

## Ramas

| Rama | Qué es |
|---|---|
| `main` | Espejo del upstream. No se toca |
| `desarsoft/2.3.7-companion-fix` | Lo que corre en producción: la etiqueta `2.3.7` del upstream más el arreglo de vinculación y el de nombres en grupos |

La rama parte de la etiqueta **`2.3.7`**, la última estable del upstream, no de `main`. `main` es la línea 2.4 en desarrollo y trae cosas que no queremos: un módulo de licencias, un canal "evohub" y dependencias a medio migrar.

## El problema

1. El 2026-07-28 WhatsApp agregó un paso nuevo al vincular un dispositivo: manda `<notification type="companion_reg_refresh">` después de escanear el QR, y con eso retira la clave `advSecretKey` que el QR anunciaba.
2. Baileys acusa recibo del aviso y lo descarta. El QR en pantalla sigue anunciando una clave que el servidor ya retiró.
3. El teléfono escanea, WhatsApp rechaza la vinculación y muestra **"No se pudo vincular el dispositivo, vuelve a intentarlo más tarde"**. El `pair-success` nunca llega.
4. Afecta a todos los clientes no oficiales, no solo a Evolution: también a whatsmeow.

Referencias: [Baileys#2737](https://github.com/WhiskeySockets/Baileys/issues/2737) y [evolution-api#2727](https://github.com/evolution-foundation/evolution-api/pull/2727).

## Qué cambia este fork

Tres cosas, nada más:

| Archivo | Cambio |
|---|---|
| `patches/baileys+7.0.0-rc14.patch` | El arreglo, tomado del PR 2727 sin modificar. Aplica sobre `node_modules/baileys` con `patch-package` |
| `package.json` | Baileys sube de `7.0.0-rc.9` a `7.0.0-rc14`, se agrega `patch-package` y el script `postinstall` |
| `Dockerfile` | Copia `./patches` antes de `npm ci`, para que el `postinstall` los encuentre |

**Por qué sube Baileys:** el parche modifica `lib/Utils/companion-reg-client-utils.js`, un archivo que la `rc.9` todavía no tiene. Verificado: sobre `rc.9` los cuatro bloques fallan; sobre `rc14` aplica limpio.

**Qué hace el parche** (revisado línea por línea antes de adoptarlo):

1. Atiende la notificación `companion_reg_refresh` en lugar de descartarla.
2. Genera una `advSecretKey` nueva, igual que hace WhatsApp Web: 32 bytes al azar en base64.
3. Vuelve a dibujar **el mismo** QR con la clave nueva, sin consumir otro del lote que asignó el servidor.
4. No rota nada si la sesión ya está registrada, que rompería una sesión viva.

No agrega llamadas de red, no toca el manejo de mensajes y no cambia nada fuera del emparejamiento.

**Lo que no se trajo del PR 2727:** todo lo demás. El PR está hecho sobre la rama de desarrollo del autor y arrastra el módulo de licencias, el canal "evohub" y una migración de dependencias. De ahí salió solo el archivo del parche.

## Nombres de los participantes de un grupo

En un grupo, Evolution encabeza cada mensaje que manda a Chatwoot con `**<teléfono> - <nombre>:**`. Upstream usa el `pushName`: el nombre que cada persona se pone en su propio WhatsApp, que a veces es solo un emoji.

| Archivo | Cambio |
|---|---|
| `src/api/integrations/chatbot/chatwoot/services/chatwoot.service.ts` | `nombreParticipante()`: busca al participante en Chatwoot por su teléfono (`findContact`) y usa el nombre de su ficha. Se usa en los dos lugares que arman el encabezado: mensajes con adjunto y mensajes de texto |

- Si no hay ficha, si la ficha solo tiene el número, o si la búsqueda falla, queda el `pushName` como en upstream.
- El resultado se guarda 10 minutos en memoria por instancia y teléfono, para no consultar a Chatwoot en cada mensaje.
- No toca nada fuera de los grupos.

## Reproducir la imagen desde cero

Sirve para el servidor de oficina y para cualquier máquina nueva si ese servidor se pierde. Solo hace falta Docker y acceso a internet.

```bash
git clone -b desarsoft/2.3.7-companion-fix https://github.com/RonaldPaguay/evolution-api.git
cd evolution-api
docker build -t evolution-api:2.3.7-companion-fix .
```

Verificar que el parche quedó dentro de la imagen:

```bash
docker run --rm evolution-api:2.3.7-companion-fix \
  grep -c handleCompanionRegRefresh node_modules/baileys/lib/Socket/socket.js
# tiene que devolver 1 (o más)
```

`docker build` falla si el parche no aplica, así que una imagen construida es una imagen parchada.

El stack que la usa (compose, variables y volúmenes) está en el servidor, en `/home/sysadmin/evolution/`, y se respalda con `backup.sh`. La sesión de WhatsApp vive en el volumen `evolution_instances` y en su Postgres: si se pierden, hay que volver a escanear el QR, y el historial se importa de nuevo desde el teléfono.

## Traer cambios del upstream

```bash
git remote add upstream https://github.com/evolution-foundation/evolution-api.git   # una sola vez
git fetch --tags upstream
git checkout -b desarsoft/<nueva-version>-companion-fix <nueva-etiqueta>
git cherry-pick <el commit de este fork>
```

**Cuando el upstream integre el arreglo** (el PR 2727 o el que lo reemplace), este fork deja de hacer falta: se vuelve a la imagen oficial `evoapicloud/evolution-api:<versión>` y se borra la rama. Conviene revisar cada tanto si ya se integró.

**Ojo con el parche al actualizar Baileys.** `patch-package` valida la versión exacta del nombre del archivo. Si Baileys sube de `rc14`, hay que regenerar el parche o renombrarlo, y comprobar que sigue aplicando.
