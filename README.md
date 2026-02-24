# 🕒 Scheduler (Fly Cron Manager) — Pipeline `routes:pipeline`

Este documento describe cómo queda planificada y operada la ejecución periódica del pipeline en Fly usando **Cron Manager**, incluyendo comandos de verificación, logs, estado de cola y troubleshooting.

---

## 1) Arquitectura (alto nivel)

- **App scheduler**: `dt-integration-cron`  
  Ejecuta **Cron Manager** (daemon) y dispara jobs según `schedules.json`.
- **App target**: `dt-integration-batch`  
  Ejecuta el comando real: `php bin/console routes:pipeline --strict-incremental`.
- **Ejecución por job**: Cron Manager **crea una Machine efímera** (one-off) en `dt-integration-batch` con la imagen especificada.
- **Anti-concurrencia**: el integrador implementa `job_locks` → si ya hay pipeline corriendo:  
  `Otro pipeline está en ejecución. Abortando.`

---

## 2) Configuración final (scheduler)

### 2.1 `schedules.json` (IMPORTANTE: `command` debe ser string)

Ejemplo (job ID=1 en la tabla de schedules):

```json
[
  {
    "name": "routes-pipeline-5m-offset-3",
    "app_name": "dt-integration-batch",
    "schedule": "3-59/5 * * * *",
    "region": "gru",
    "command": "php bin/console routes:pipeline --strict-incremental",
    "command_timeout": 300,
    "enabled": true,
    "config": {
      "image": "registry.fly.io/dt-integration-batch:deployment-XXXXXXXXXXXX",
      "guest": { "cpu_kind": "shared", "cpus": 1, "memory_mb": 512 },
      "restart": { "policy": "no" }
    }
  }
]
```

Notas críticas

command NO puede ser array (["php", "bin/console", ...]) → debe ser string.

config.image queda pinneada al deployment del integrador. Al redeployar dt-integration-batch cambia el tag.

### 2.2 fly.toml de dt-integration-cron (volumen requerido)

Cron Manager usa SQLite para store + migraciones. Requiere volumen montado:

```
app = "dt-integration-cron"
primary_region = "gru"

[build]

[processes]
app = "/usr/local/bin/start"

[[mounts]]
  source = "data"
  destination = "/data"

[[vm]]
  memory_mb = 512
  cpus = 1
```

### 2.3 Volumen data

Cron Manager necesita un volumen data por máquina (en este caso usamos 1 máquina).

Crear volumen (una sola vez):

```
flyctl volumes create data --app dt-integration-cron --region gru --size 1
flyctl volumes list -a dt-integration-cron
```

## 3) Autenticación (token correcto)

Cron Manager necesita llamar la Fly API para crear Machines en dt-integration-batch.

### 3.1 Token requerido

No sirve “Deploy Token” por app.

Lo correcto es un Org token (API con permisos sobre la org donde vive dt-integration-batch).

Crear token (no pegues el token en chats / issues):

```
flyctl tokens create org --org contenedores-patagonia
```

Setearlo como secret en dt-integration-cron:

```
flyctl secrets unset FLY_API_TOKEN -a dt-integration-cron
flyctl secrets set FLY_API_TOKEN="TOKEN_GENERADO" -a dt-integration-cron
flyctl secrets list -a dt-integration-cron
```

Reiniciar machine para tomar el secret:

```
flyctl machine list -a dt-integration-cron
flyctl machine restart <CRON_MACHINE_ID> -a dt-integration-cron
```

## 4) Deploy / actualización del scheduler

Deploy del cron manager:

```
flyctl deploy -a dt-integration-cron
flyctl status -a dt-integration-cron
flyctl machine list -a dt-integration-cron
```

## 5) Operación diaria — comandos útiles

### 5.1 Ver schedules activos

```
flyctl ssh console -a dt-integration-cron -C "cm schedules list"
```

Deberías ver una fila con:

ID = 1

TARGET APP = dt-integration-batch

SCHEDULE = 3-59/5 \* \* \* \*

### 5.2 Disparar ejecución manual (sin esperar cron)

```
flyctl ssh console -a dt-integration-cron -C "cm jobs trigger 1"
```

### 5.3 Ver últimas ejecuciones del schedule (historial)

```
flyctl ssh console -a dt-integration-cron -C "cm jobs list 1"
```

Campos típicos:

STATUS: running / success / failed

EXIT CODE

MACHINE ID (de la ejecución en dt-integration-batch)

## 6) Logs (scheduler y jobs)

### 6.1 Logs del cron manager (dt-integration-cron)

Útil para diagnosticar parseo de schedules, fallos de auth, etc.

```
flyctl logs -a dt-integration-cron
```

### 6.2 Logs del integrador (target) global

Muestra todo lo que ejecuta dt-integration-batch, incluyendo jobs efímeros.

```
flyctl logs -a dt-integration-batch
```

### 6.3 Logs de una ejecución específica (por Machine ID)

Cuando cm jobs list 1 entrega MACHINE ID, puedes filtrar:

```
flyctl logs -a dt-integration-batch -i <MACHINE_ID_DEL_JOB>
```

Esto es el “debug perfecto” para un run puntual del pipeline.

## 7) Estado de la cola (Dispatch Queue) / queries operativas

Requiere conectarse a la base Supabase/Postgres donde está dispatch_queue.
Usa tu método habitual (psql / consola / admin UI). Aquí va el set mínimo de queries.

### 7.1 Conteo por acción / estado básico

```
-- Total por acción
select action, count(*)
from dispatch_queue
group by action
order by action;

-- Pendientes (ajusta según tu modelo: processed_at/status/attempts)
-- Ejemplo si existe processed_at:
select action, count(*)
from dispatch_queue
where processed_at is null
group by action
order by action;
```

### 7.2 Jobs en error / retries

```
-- Ejemplo si hay status + attempts
select action, count(*)
from dispatch_queue
where status = 'error'
group by action;

select *
from dispatch_queue
where status = 'error'
order by updated_at desc
limit 50;
```

### 7.3 Últimos jobs procesados

```
-- Ajusta columnas a tu esquema real
select *
from dispatch_queue
order by updated_at desc
limit 50;
```

### 7.4 Ver un dispatch específico

```
select *
from dispatch_queue
where dispatch_identifier = 'CP-27042'
order by tran_id;
```

## 8) Verificación del lock del pipeline (job_locks)

Ajusta nombres/columnas si varían en tu implementación.

### 8.1 Ver locks actuales

```
select *
from job_locks
order by created_at desc;
```

### 8.2 Limpieza manual (solo si queda “pegado” por caída)

```
-- Ejemplo: borra lock del pipeline si quedó huérfano
delete from job_locks
where job_name = 'routes:pipeline';
```

Recomendación: preferir que el lock expire (TTL) si lo implementaste; borrar manual solo si confirmas que no hay job corriendo.

## 9) Actualización de imagen cuando se redeploya dt-integration-batch

Cron Manager está pinneado a la imagen del integrador. Si haces deploy del integrador y cambia el tag, debes actualizar schedules.json.

### 9.1 Obtener imagen vigente del integrador

```
flyctl image show -a dt-integration-batch
```

Ejemplo de ref:
registry.fly.io/dt-integration-batch:deployment-01KJ...

### 9.2 Actualizar schedules.json + redeploy cron-manager

Editar schedules.json → reemplazar config.image.

Deploy:

```
flyctl deploy -a dt-integration-cron
```

Verificar:

```
flyctl ssh console -a dt-integration-cron -C "cm schedules list"
```

## 10) Troubleshooting (casos reales y cómo resolver)

### 10.1 cm schedules list aparece vacío

Causa típica: command estaba como array en JSON.
Fix: dejar command como string y redeploy.

```
flyctl deploy -a dt-integration-cron
flyctl ssh console -a dt-integration-cron -C "cm schedules list"
```

### 10.2 failed to create store ... unable to open database file

Causa: faltaba volumen data / mount /data.
Fix:

crear volumen

agregar [[mounts]] y redeploy

```
flyctl volumes create data --app dt-integration-cron --region gru --size 1
flyctl deploy -a dt-integration-cron
```

### 10.3 failed to launch VM: unauthorized

Causa: token incorrecto (deploy token) o sin permisos.
Fix: crear org token y setear como secret.

```
flyctl tokens create org --org contenedores-patagonia
flyctl secrets set FLY_API_TOKEN="TOKEN" -a dt-integration-cron
flyctl machine restart <CRON_MACHINE_ID> -a dt-integration-cron
```

### 10.4 Jobs seguidos “Abortando: otro pipeline en ejecución”

Causa: cron muy frecuente y pipeline tarda más que el intervalo.
Fix (opcional):

mantener 5 min y aceptar abortos (funcional)

o subir intervalo (ej. cada 10 min):

3-59/10 \* \* \* \*

Actualizar schedules.json y redeploy.

## 11) Comandos de referencia (copypaste)

Scheduler

```
flyctl status -a dt-integration-cron
flyctl machine list -a dt-integration-cron
flyctl logs -a dt-integration-cron
flyctl ssh console -a dt-integration-cron -C "cm schedules list"
flyctl ssh console -a dt-integration-cron -C "cm jobs list 1"
flyctl ssh console -a dt-integration-cron -C "cm jobs trigger 1"
```

Target (integrador)

```
flyctl status -a dt-integration-batch
flyctl logs -a dt-integration-batch
flyctl logs -a dt-integration-batch -i <MACHINE_ID>
flyctl image show -a dt-integration-batch
```
