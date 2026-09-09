# Medico ZX 2.0 — Consulta de expedientes

Aplicación de **consulta**. Muestra la ficha del colaborador, su historia clínica y el PDF de sus
análisis, para exhibirlos en auditoría cuando se pidan. **No captura datos**: entran por carga
masiva desde los archivos de Recursos Humanos.

---

## 1. Quién entra

Sólo el **servicio médico** (rol `medico`) y el **administrador** (rol `admin`).

Los 338 colaboradores conservan su perfil y su expediente, pero tienen el acceso apagado
(`acceso_habilitado = false`). Si alguno intenta entrar, la aplicación le explica que es de uso
exclusivo del servicio médico y le indica pedir su expediente a Recursos Humanos.

Para reactivar el acceso de alguien:

```sql
update public.perfiles set acceso_habilitado = true where numero_nomina = '1234';
```

---

## 2. Qué se ve

**Pacientes** — lista de las personas activas, con búsqueda por nombre, nómina o área.

**Expediente** — se abre en *Ficha del colaborador* y tiene tres pestañas:

| Pestaña | Qué muestra |
|---|---|
| Ficha del colaborador | Datos laborales, de Recursos Humanos |
| Historia clínica | Lo capturado en el registro médico |
| Análisis clínicos | Los PDF del laboratorio, con carga y descarga |

Dentro del expediente se puede pasar al paciente **anterior o siguiente**, o saltar a cualquiera
con el selector, sin volver a la lista.

---

## 3. Los PDF de análisis

Viven en Supabase Storage, bucket `analisis`, **privado**. Sólo PDF, hasta 20 MB.

Que el bucket sea privado es la diferencia entre que un análisis se vea sólo con sesión válida o
que quede accesible para cualquiera que dé con la dirección. El enlace para ver un PDF **se genera
en el momento y caduca en un minuto**.

**Carga masiva** — para no entrar persona por persona:

```powershell
$env:SUPABASE_URL="https://hpheqabqglvgrmaofher.supabase.co"
$env:SUPABASE_SECRET_KEY="sb_secret_..."

node scripts/cargar-analisis.mjs "C:\ruta\a\los\pdf" --revisar
node scripts/cargar-analisis.mjs "C:\ruta\a\los\pdf"
```

El script identifica a la persona por el **número de nómina al inicio del nombre del archivo**
(`1234.pdf`, `1234 - JUAN PEREZ.pdf`, `1234_analisis_2026.pdf`). Con `--revisar` te dice qué haría,
cuánto espacio va a ocupar y qué archivos no encontraron dueño, sin subir nada.

**Capacidad del plan gratuito:** 1 GB de almacenamiento, 50 MB por archivo y 5 GB de descarga al
mes. Con 338 PDF digitales del laboratorio (~300 KB) se ocupa cerca del 10%.

---

## 4. Archivos

```
app/
├── index.html          Acceso con número de nómina
├── clinico.html        Pacientes y expedientes  ← la aplicación
├── cuenta.html         Cambio de contraseña
├── privacidad.html     Aviso de privacidad (enlazado en el pie)
├── _headers            Control de caché (sólo funciona en Netlify)
└── assets/
    ├── config.js       ⭐ URL y llave pública de Supabase
    ├── estilos.css     Imagen institucional Zubex
    ├── zx.js           Núcleo: cliente, sesión, permisos, utilidades
    └── pagina-*.js     Lógica de cada pantalla
```

---

## 5. Lo que quedó desactivado, no borrado

Las tablas de citas, consultas, casos de seguimiento, programación de análisis, resultados y
solicitudes ARCO **siguen en la base con sus datos y sus reglas**. Sólo se retiraron de la
interfaz. Si más adelante se retoma la captura, se vuelven a mostrar sin reconstruir nada.

Lo único que se quitó de verdad es el **disparador de consentimiento**: la base ya no comprueba
nada antes de guardar datos de salud, porque el consentimiento lo resguarda Recursos Humanos en
papel. La función quedó instalada por si se decide reactivarla, y el SQL para registrar la
evidencia dentro del sistema está comentado al inicio de `supabase/06_solo_consulta.sql`.

La **bitácora sigue registrando** cada apertura de expediente, carga y descarga de archivo, aunque
ya no haya pantalla que la muestre. Se consulta en Table Editor → `bitacora`.
