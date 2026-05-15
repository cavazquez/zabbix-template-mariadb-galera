# 📊 zabbix-template-mariadb-galera

[![CI](https://github.com/cavazquez/zabbix-template-mariadb-galera/actions/workflows/validate.yml/badge.svg)](https://github.com/cavazquez/zabbix-template-mariadb-galera/actions/workflows/validate.yml)
[![Zabbix 6.0+](https://img.shields.io/badge/Zabbix-6.0%2B-CC2936?logo=zabbix&logoColor=white)](https://www.zabbix.com/documentation/6.0/en/manual/installation/upgrade_notes_600)
[![Zabbix agent 2](https://img.shields.io/badge/Zabbix_agent-2-CC2936?logo=zabbix&logoColor=white)](https://www.zabbix.com/documentation/6.0/en/manual/appendix/config/zabbix_agent2)

Plantilla **Zabbix 6.0+** para supervisar **MariaDB Galera Cluster** (`wsrep_*`) con **Zabbix agent 2** y el **plugin MySQL** oficial. No se usan scripts en el nodo ni `UserParameter`: las métricas salen de `mysql.get_status_variables` (equivalente a `SHOW GLOBAL STATUS`) y de ítems dependientes con **JSONPath**, igual que en la plantilla *MySQL by Zabbix agent 2*.

No incluye datos de ningún servidor concreto: rutas, puertos y usuarios en la documentación o en `mysql.conf.example` son **solo ejemplos**; debes sustituirlos por los de tu entorno (sobre todo `{$MYSQL.DSN}` en cada host).

## 📦 Qué incluye el repositorio

| Ruta | Descripción |
|------|-------------|
| `templates/template_mariadb_galera_6.0.yaml` | Plantilla **MariaDB Galera (Zabbix agent 2)** (`TemplateMariaDBGaleraAgent2`): ítems, disparadores anidados bajo ítems, macros. |
| `zabbix_agent2.d/plugins.d/mysql.conf.example` | Ejemplos de `Plugins.Mysql.Sessions.*` / `Default.*` (socket, TCP, Docker). |
| `zabbix_agent2.d/README.txt` | Nota breve sobre la carpeta `zabbix_agent2.d/` del repo frente a `/etc/zabbix/` en el servidor. |

## ✅ Requisitos

- **Zabbix server 6.0 o superior** (import YAML y claves del plugin agent 2).
- En cada nodo a vigilar: **zabbix-agent2** instalado y accesible desde el servidor o proxy Zabbix.
- Usuario de solo lectura en MariaDB con permisos acordes al [README del plugin MySQL](https://raw.githubusercontent.com/zabbix/zabbix/master/src/go/plugins/mysql/README.md) (habitualmente `REPLICATION CLIENT`, `PROCESS`, `SHOW DATABASES`, `SHOW VIEW`).
- Instancia **Galera** (las variables `wsrep_*` deben existir en el estado global).

## 🔄 Flujo de trabajo recomendado

1. **MariaDB**: crea el usuario de monitorización (p. ej. `zbx_monitor`) y comprueba con el cliente `mysql` que puedes leer `SHOW GLOBAL STATUS LIKE 'wsrep%'` desde el mismo sitio donde correrá el agente 2.
2. **Agente 2**: configura conexión por **macros en el host** Zabbix (opción A) o por **sesión nombrada** en `mysql.conf` del plugin (opción B). Reinicia `zabbix-agent2`.
3. **Zabbix**: importa la plantilla, enlázala a un host por nodo Galera y ajusta las macros del host.

## ⚙️ Configuración del agente 2

Documentación del fabricante: [MySQL plugin (Zabbix agent 2)](https://www.zabbix.com/documentation/current/en/manual/appendix/config/zabbix_agent2_plugins/mysql_plugin).

### Opción A — Todo desde macros del host (sin editar `mysql.conf`)

En cada **host** en Zabbix define:

| Macro | Ejemplo | Uso |
|-------|---------|-----|
| `{$MYSQL.DSN}` | *(definir en cada host)* | URI (`tcp://…`, `unix:/…`) o nombre de sesión del plugin (opción B). |
| `{$MYSQL.USER}` | `zbx_monitor` | Usuario (vacío si usas solo sesión nombrada). |
| `{$MYSQL.PASSWORD}` | *(secreto)* | Contraseña (vacía si usas solo sesión). |

La plantilla ya usa esas macros en `mysql.get_status_variables[...]` y `mysql.ping[...]`.

### Opción B — Sesión nombrada en el fichero del plugin

1. Copia o fusiona `zabbix_agent2.d/plugins.d/mysql.conf.example` en la ruta que use tu paquete (suele existir un `Include` hacia `zabbix_agent2.d/plugins.d/*.conf`).
2. Define `Plugins.Mysql.Sessions.MiSesion.Uri`, `User`, `Password`, etc.
3. En el host Zabbix: `{$MYSQL.DSN}` = `MiSesion` (nombre exacto) y deja **vacíos** `{$MYSQL.USER}` y `{$MYSQL.PASSWORD}`.

Reinicia el servicio: `systemctl restart zabbix-agent2`.

### Probar una clave en el propio nodo

```bash
sudo zabbix_agent2 -c /etc/zabbix/zabbix_agent2.conf -v -t 'mysql.ping["tcp://127.0.0.1:3306","zbx_monitor","tu_password"]'
```

(Ajusta `-c`, DSN, usuario y contraseña; si usas sesión nombrada, la clave puede ser `mysql.ping["MiSesion","",""]`.)

## 🐳 Galera en Docker

El plugin solo necesita un **DSN alcanzable**; en contenedores suele usarse **TCP**.

- **Agente en el host**, bases en Docker: publica puertos (`3306`, `33061`, …) y usa `{$MYSQL.DSN}` = `tcp://127.0.0.1:<puerto>` por nodo. El usuario MySQL debe aceptar conexiones desde donde conecta el agente (p. ej. `zbx_monitor@'%'` o la subred del bridge).
- **Agente en contenedor** en la misma red que Galera: `{$MYSQL.DSN}` = `tcp://<nombre_servicio>:3306`.

Más ejemplos comentados en `zabbix_agent2.d/plugins.d/mysql.conf.example`.

## 📥 Zabbix: importar plantilla y enlazar

1. **Datos → Plantillas → Importar** el fichero `templates/template_mariadb_galera_6.0.yaml`.
2. Asocia la plantilla **MariaDB Galera (Zabbix agent 2)** al host de cada nodo.
3. Revisa las **macros** del host (`{$MYSQL.DSN}`, credenciales, y si aplica `{$GALERA.FLOW_CONTROL.MAX}` / `{$GALERA.RECV_QUEUE.MAX}`).

### Importación y gráficos

- El YAML exporta solo **`items`** y **`macros`**: el orden evita problemas con algunos importadores. En ciertos entornos la etiqueta `graphs` en el export provocaba error (`unexpected tag "graphs"`); por eso **no** va el gráfico en el fichero.
- Puedes crear en el frontend un **gráfico clásico** en la plantilla con los ítems `galera.wsrep_flow_control_paused`, `galera.wsrep_local_recv_queue` y `galera.wsrep_local_recv_queue_avg`.
- Los **disparadores** van definidos **bajo el ítem** correspondiente (formato de exportación Zabbix 6.x).

Si un ítem `wsrep_*` queda no soportado, comprueba que el servidor sea realmente Galera y que el nombre en el JSON coincida con el JSONPath (`$.wsrep_cluster_status`, etc., según mayúsculas/minúsculas de tu versión).

## 📈 Métricas y disparadores (resumen)

- **Estado:** `wsrep_cluster_status`, `wsrep_cluster_size`, `wsrep_cluster_state_uuid`, `wsrep_connected`, `wsrep_ready`, `wsrep_local_state_comment`.
- **Presión:** `wsrep_flow_control_paused`, `wsrep_local_recv_queue`, `wsrep_local_recv_queue_avg`, `wsrep_cert_deps_distance`.
- **Disparadores:** fallo de `mysql.ping`; nodo no en `Primary` (insensible a mayúsculas); `wsrep_connected` / `wsrep_ready` distintos de `ON`; flow control y cola por encima de los umbrales en macros.

Para **particiones** o **split-brain** suele compararse `wsrep_cluster_state_uuid` entre nodos en el mismo instante; eso queda fuera de esta plantilla (haría falta correlación manual u otra automatización).

## 🔗 Plantilla oficial MySQL (opcional)

Puedes enlazar además la plantilla oficial **MySQL by Zabbix agent 2** si quieres métricas generales de instancia; revisa duplicados de ítems maestros o intervalos para no sobrecargar MariaDB.

## 🤖 CI (GitHub Actions)

La primera insignia del encabezado muestra el **estado del último workflow** [`validate.yml`](.github/workflows/validate.yml) en la rama por defecto (pasa / falla según GitHub).

En cada *push* o *pull request* a `main`, ese workflow comprueba que la plantilla YAML sea sintácticamente válida (PyYAML).

## 📄 Licencia

Este proyecto se publica bajo la [**Apache License, versión 2.0**](https://www.apache.org/licenses/LICENSE-2.0). El texto legal completo está en el fichero [`LICENSE`](LICENSE) del repositorio (SPDX: `Apache-2.0`).
