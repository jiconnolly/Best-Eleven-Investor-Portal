# Acceso restringido al portal — Cloudflare Access

Guía para dejar el deck detrás de autenticación real, con registro de quién entra y cuándo.

**Resultado final:** cada inversor recibe el link. Al abrirlo, Cloudflare le pide su email,
le manda un PIN de un solo uso, y recién ahí ve el deck. Cada acceso queda registrado con
email, fecha, hora, IP y país.

**Costo:** gratis hasta 50 usuarios en el plan Free de Cloudflare Zero Trust.

**Requisito:** un dominio propio administrado por Cloudflare (por ejemplo
`besteleven.com`). Sin dominio no se puede: Access necesita un hostname dentro de una zona
que vos controles.

---

## 1. Cerrar la puerta que está abierta hoy

Esto es lo primero y no es opcional. Mientras el repo sea público, el HTML del deck se
descarga directo de GitHub y cualquier protección que pongamos arriba es irrelevante.

1. GitHub → repo → **Settings** → abajo de todo, **Danger Zone** → *Change repository
   visibility* → **Make private**.
2. GitHub → **Settings** → **Pages** → en *Source*, elegir **None**. Esto apaga
   `jiconnolly.github.io/Best-Eleven-Investor-Portal`, que hoy sirve el deck sin ninguna
   restricción.

Después de esto el deck queda temporalmente offline hasta terminar el paso 3. Es lo
esperado.

## 2. Poner el dominio en Cloudflare

Si el dominio ya está en Cloudflare, saltear.

1. Crear cuenta en [cloudflare.com](https://cloudflare.com).
2. **Add a site** → escribir el dominio → plan **Free**.
3. Cloudflare da dos nameservers. Entrar al registrador donde compraste el dominio
   (GoDaddy, Namecheap, Nic.ar, etc.) y reemplazar los nameservers por esos dos.
4. La propagación tarda entre minutos y unas horas. Cloudflare avisa por mail cuando el
   dominio está activo.

## 3. Publicar el sitio con Cloudflare Pages

Pages sirve el repo directamente y funciona con repos privados.

1. Panel de Cloudflare → **Workers & Pages** → **Create** → pestaña **Pages** →
   **Connect to Git**.
2. Autorizar GitHub y elegir `jiconnolly/Best-Eleven-Investor-Portal`.
3. Configuración del build:
   - Production branch: `main`
   - Framework preset: **None**
   - Build command: **vacío**
   - Build output directory: `/`
4. **Save and Deploy**. En menos de un minuto queda en `algo.pages.dev`.
5. Dentro del proyecto → **Custom domains** → **Set up a custom domain** →
   `deck.tudominio.com`. Cloudflare crea el registro DNS solo.

En este punto el deck ya está online en `https://deck.tudominio.com`, todavía sin
contraseña. Verificá que se vea bien antes de seguir.

> Cada push a `main` redeploya solo. Los cambios del deck se publican con un `git push`,
> igual que antes.

## 4. Poner la puerta con Access

1. Panel de Cloudflare → **Zero Trust** (menú izquierdo). La primera vez pide elegir un
   nombre de equipo y un plan: elegir **Free**. Pide tarjeta pero no cobra con menos de
   50 usuarios.
2. **Access** → **Applications** → **Add an application** → **Self-hosted**.
3. Configuración:
   - Application name: `Deck BESG`
   - Session duration: **24 hours** (cuánto dura sin volver a pedir PIN)
   - Public hostname: subdominio `deck`, dominio `tudominio.com`, path vacío
4. **Next** → crear la policy:
   - Policy name: `Inversores`
   - Action: **Allow**
   - Include → selector **Emails** → pegar la lista de mails, uno por línea.
     (Si querés dar acceso a toda una empresa: selector **Emails ending in** →
     `@fondodeinversion.com`.)
5. **Next** → en *Login methods* dejar **One-time PIN** activado. Es el que manda el
   código por mail sin que el visitante necesite cuenta de nada.
6. **Add application**.

Listo. Entrá desde una ventana incógnito para probarlo.

## 5. Ver quién entró

**Zero Trust** → **Logs** → **Access**.

Una fila por cada intento, con email, fecha y hora, IP, país, y si fue permitido o
rechazado. Se exporta a CSV desde el mismo panel. Los intentos rechazados también quedan,
así que si alguien reenvía el link a un tercero, lo ves.

## 6. Operación de todos los días

| Qué querés hacer | Dónde |
|---|---|
| Dar acceso a alguien nuevo | Zero Trust → Access → Applications → `Deck BESG` → editar policy → agregar el mail |
| Sacarle el acceso a alguien | Misma pantalla, borrar el mail. Efecto inmediato al vencer su sesión |
| Ver quién entró | Zero Trust → Logs → Access |
| Actualizar el deck | `git push` a `main`, como siempre |

## Detalle: preview de links

Access bloquea también a los bots que arman la vista previa de los links. Si mandás el
link por WhatsApp o mail, va a aparecer sin imagen ni título.

Si te importa, agregá una segunda policy en la misma aplicación:

- Policy name: `Preview publico`
- Action: **Bypass**
- Include: **Everyone**
- Y en la app, path: `og-image.jpg`

Eso deja la imagen de preview accesible sin login, y el resto protegido.

---

## Lo que esto sí resuelve y lo que no

Resuelve que nadie sin invitación abra el deck, y que sepas exactamente quién lo abrió y
cuándo.

No resuelve qué hace la persona una vez adentro: puede sacar capturas, imprimir a PDF o
guardar la página. Ninguna herramienta web evita eso, ni DocSend ni ninguna otra. El
control es sobre quién entra, no sobre qué se lleva.
