Revisé el proyecto. Esto es lo que encontré:

- `backend/` es CodeIgniter 4.
- `app/Database/Migrations/` existe pero está vacía.
- `deploy.yml` ya tiene el TODO `php spark migrate` en el paso Post-deploy.
- Tu BD de producción se creó a mano, así que no tiene historial de migraciones. Este dato es el más importante de todos.

## Cómo funcionan las migraciones en CI4

> Una analogía: hoy remodelas el edificio sin planos. Entras con el SQL y tumbas una pared. Las migraciones son la bitácora de obra versionada: cada cambio queda en un archivo con fecha, y el sistema sabe cuáles ya se aplicaron.

### 1. Creas un archivo por cada cambio

```bash
   php spark make:migration AddRucToClientes

# → app/Database/Migrations/2026-10-20-153000_AddRucToClientes.php
```

### 2. Escribes `up()` (aplicar) y `down()` (revertir)

```php
    class AddRucToClientes extends Migration
    {
    public function up()
    {
    $this->forge->addColumn('clientes', [
    'ruc' => ['type' => 'VARCHAR', 'constraint' => 13, 'null' => true],
    ]);
    // También puedes usar SQL directo: $this->db->query('ALTER TABLE ...');
    }

        public function down()
        {
            $this->forge->dropColumn('clientes', 'ruc');
        }

    }
```

### 3. Lo ejecutas

```bash
    php spark migrate # aplica las pendien
    php spark migrate:status # muestra qué se aplicó y qué no
    php spark migrate:rollback # revierte el último
```

CI4 crea una tabla `migrations` en tu BD y regist corrió. `migrate` compara los archivos que hay en el repo con esa tabla y ejecuta solo los que faltan, en orden de timestamp. Por eso puedes correrlo en cada deploy sin
riesgo: si no hay nada nuevo, no hace nada.

## ¿Lo recomiendo? SÍ, pero con condiciones

### A favor

- El esquema queda versionado junto al código, y el código y la BD se despliegan juntos. e termina el "¿ya corrí ese ALTER en producc
- Si tienes varias BD (test, prod, otros clientes), todas quedan iguales sin trabajo manual.

### Los riesgos reales, y cómo cubrirlos

1. El baseline. Tu BD ya existe. NO intentes recrear todo el esquema actual en migraciones. Tomas la BD actual como punde partida, y solo los cambios nuevos se esc
2. Una migración que falla a mitad de camino. MySQL hace commit implícito en cada ALTER, así que un fallo puede dejar lBD a medias. Por eso hace falta un mysqldumpr, sin excepción.
3. Rollback automático en CI: NO. down() sirve para desarrollo. En producción, si borras una columna, down() no recuperlos datos. Revertir en producción debe ser u
4. Cambios compatibles (expand/contract). Si vas a renombrar una columna, primero agregas la nueva, despliegas, migras los datos, y la vieja se elimina en un deplo viejo y el nuevo funcionan durante latransición.
5. Disciplina del equipo. Si alguien vuelve a c, la tabla migrations deja de reflejar larealidad. La regla tiene que ser absoluta: todo cambio de esquema va en una migración.

Alternativa sin migraciones: versionar los .sql en una carpeta y aplicarlos con un script propio. Funciona, pero terminas reinventando la tabla de control que CI4 ya te da. No la recomiendo.

---

Tu workflow solo despliega desde `master`: se dispara con push a `master`, y el job de deploy tiene la condición `if: github.ref == 'refs/heads/master'`. Eso define cómo se hace un rollback en tu caso, así que lo uso como base.

# Parte 1: Rollback en producción

## Primero: un "rollback" son DOS cosas distintas

> Analogía de obra: el código son los muebles y la BD es la estructura con gente viviendo adentro. Sacar los muebles nuevos y volver a poner los viejos es fácil. Demoler una pared con gente adentro no lo es.

| | Código | Base de datos |
| --- | --- | --- |
| ¿Tiene estado? | No | Sí: datos que se crearon DESPUÉS del deploy |
| ¿Volver atrás es seguro? | Sí, siempre | No. Puede destruir datos |
| ¿Cómo se revierte? | Volver a desplegar la versión anterior | Depende del tipo de cambio (ver abajo) |

Este es el error más común: pensar que "revertir el deploy" significa revertir las dos cosas. La meta es que revertir el código NUNCA obligue a revertir la BD.

## Cómo revertir el código (en tu pipeline)

### Opción A, la recomendada: `git revert`

```bash
git revert <sha-del-commit-malo>
git push origin master # el pipeline despliega solo
```

Queda registrado en el historial qué se revirtió y por qué. Es auditable.

### Opción B, la más rápida: re-ejecutar un run anterior

En GitHub, entras a Actions, abres el último deploy exitoso y le das "Re-run all jobs". Un re-run usa el commit original de ese run, así que vuelve a construir y despliega la versión vieja.

- Ventaja: tarda unos 2 minutos y no necesitas tocar git.
- Desventaja: master sigue apuntando al código malo. El próximo push lo vuelve a desplegar si no lo corriges.

Cómo interactúa con las migraciones: el código viejo no trae los archivos de las migraciones nuevas, así que `php spark migrate` no hace nada. No revierte nada, y la BD se queda con el esquema nuevo. Por eso el código viejo TIENE que funcionar con ese esquema nuevo. Eso lleva al punto clave.

## Por qué `migrate:rollback` en producción es peligroso

Un ejemplo concreto en un ERP:

1. 10:00. Despliegas una migración que agrega la columna descuento a ventas.
2. Entre 10:00 y 14:00 se registran 300 ventas con descuento.
3. 14:00. Detectas un bug y corres migrate:rollback. El down() hace dropColumn('ventas', 'descuento').
4. Perdiste los descuentos de 300 ventas. No hay forma de recuperarlos.

"Restauro el backup de las 9:59" es peor todavía: pierdes TODAS las transacciones de 4 horas, en todas las tablas.

> **Regla de oro:** en producción no se retrocede, se avanza (roll forward). Si una migración salió mal, escribes una migración NUEVA que lo corrige.

## La técnica que lo hace posible: Expand / Contract

Cada cambio de esquema se divide para que la BD sea compatible con la versión actual del código Y con la anterior.

| Cambio | Cómo hacerlo seguro |
| --- | --- |
| Agregar tabla | Directo. El código viejo la ignora |
| Agregar columna | NULL o con DEFAULT. El código viejo la ignora |
| Agregar índice | Directo. Cuidado con tablas grandes, porque puede bloquear la tabla |
| Renombrar columna | Deploy 1: agregas la nueva, copias los datos y el código escribe en ambas. Deploy 2: el código lee solo la nueva. Deploy 3, días después: borras la vieja |
| Eliminar columna/tabla | Primero quitas su uso del código y despliegas. Semanas después, la migración la elimina |
| Cambiar tipo (ej. INT → DECIMAL) | Igual que renombrar: columna nueva, copia, cambio de código, borrado posterior |
| NOT NULL en columna existente | Primero rellenas los datos y después agregas la restricción en otra migración |

Con este método, cualquier deploy se puede revertir solo con código, y la BD nunca se toca hacia atrás.

## Entonces, ¿para qué sirve `down()`?

- En desarrollo local: para iterar (`migrate`, pruebas, `rollback`, corriges, `migrate`).
- En producción: solo lo ejecuta un humano, de forma manual, y únicamente si la migración fue aditiva y aún no tiene datos. Por ejemplo, una tabla nueva que sigue vacía.
- Nunca automático en CI.

## Protocolo ante un deploy fallido

```text
¿El problema es solo de código? (lo más común)
└─ SÍ → git revert + push. Fin. La BD no se toca.

¿La migración falló a mitad de camino?
└─ Revisa `php spark migrate:status` y el esquema real.
MySQL no revierte los ALTER dentro de una transacción: la BD puede quedar a medias.
→ Escribe una migración que complete o corrija el cambio (roll forward).

¿La migración CORROMPIÓ datos? (ej. un UPDATE sin WHERE)
└─ Solo aquí se usa el backup, con este procedimiento: 1. Modo mantenimiento: que nadie siga escribiendo. 2. mysqldump del estado ACTUAL, aunque esté roto. Ahí están las transacciones nuevas. 3. Restaurar SOLO las tablas afectadas, desde el backup previo al deploy. 4. Reconciliar a mano las filas creadas entre el backup y el incidente.
```

Para que ese último caso sea viable, el backup previo al deploy tiene que existir siempre:

```bash
mysqldump --single-transaction --routines --triggers --events bd > pre-deploy-<sha>.sql
```

- `--single-transaction`: hace un snapshot consistente sin bloquear las tablas InnoDB.
- `--triggers`: ya viene activado por defecto, pero conviene dejarlo explícito.
- `--routines` y `--events`: estos NO vienen por defecto. Si los omites, pierdes los stored procedures y los eventos programados.

---

# Parte 2: Triggers en migraciones

Sí, se puede. Se hace con SQL directo a través de `$this->db->query()`:

```php
class CreateTriggerStockVenta extends Migration
{
public function up()
{
$this->db->query('DROP TRIGGER IF EXISTS trg_ventas_descuenta_stock');
$this->db->query("
CREATE TRIGGER trg_ventas_descuenta_stock
AFTER INSERT ON venta_detalle
FOR EACH ROW
BEGIN
UPDATE productos
SET stock = stock - NEW.cantidad
WHERE id = NEW.producto_id;
END
");
}

    public function down()
    {
        $this->db->query('DROP TRIGGER IF EXISTS trg_ventas_descuenta_stock');
    }

}
```

## Detalles que te van a morder si no los conoces

### 1. NO uses `DELIMITER`

`DELIMITER $$` es un comando del cliente mysql de terminal, no es SQL. Desde PHP cada `query()` envía una sentencia completa, así que el `;` interno del `BEGIN...END` no causa problemas. Si copias un script que trae `DELIMITER`, quítalo. Si lo dejas, la migración falla con un error de sintaxis.

### 2. Modificar un trigger existente = drop + create en una migración NUEVA

MySQL 8 no tiene `CREATE OR REPLACE TRIGGER`. MariaDB sí lo tiene, desde la 10.1.4. Si un trigger cambia, la migración nueva hace:

- `up()`: DROP y luego CREATE con la versión nueva.
- `down()`: DROP y luego CREATE con la versión anterior, copiada tal cual.

Con los triggers que ya existen en producción, que forman parte del baseline, saca su definición actual con `SHOW CREATE TRIGGER nombre;` antes de modificarlos. Nunca edites una migración ya ejecutada. Ya corrió, así que CI4 no la vuelve a ejecutar, y tu cambio nunca llega a producción.

### 3. Permisos en el VPS

- El usuario de la BD necesita el privilegio `TRIGGER`.
- Si el servidor tiene el binary log activo, crear triggers puede fallar con `You do not have the SUPER privilege and binary logging is enabled`. La solución es `log_bin_trust_function_creators=1` en el servidor, o hacerlo con un usuario con más permisos.
- Esto hay que verificarlo en tu VPS antes de confiar en el pipeline. Lo puedes probar a mano con un trigger de prueba.

### 4. `DEFINER` en los backups

`mysqldump` guarda cada trigger con `DEFINER=usuario@host`. Si restauras en otra BD u otro servidor y ese usuario no existe, el trigger falla al ejecutarse, no al restaurar. Revisa este punto cuando restaures en el entorno de test.

### 5. Los triggers interactúan con el rollback de código

Un trigger es lógica de negocio que vive en la BD, no en el código. Si revierte el código pero el trigger nuevo sigue activo, el código viejo convive con una lógica que no conoce. Aplica la misma regla de expand/contract: un trigger nuevo debe funcionar bien con la versión anterior del código. Por ejemplo, si el trigger lee una columna, esa columna debe existir en ambas versiones del esquema.

---

# Resumen

1. El código se revierte, la BD avanza. Los dos rollbacks están desacoplados a propósito.
2. Expand/contract es lo que hace seguro revertir el código.
3. migrate:rollback en producción solo lo ejecuta una persona, y solo para cambios aditivos sin datos.
4. El backup previo al deploy es para desastres, no es tu mecanismo normal de rollback.
5. Los triggers se manejan con $this->db->query(), sin DELIMITER, siempre con drop + create, y verificando los permisos en el VPS.
