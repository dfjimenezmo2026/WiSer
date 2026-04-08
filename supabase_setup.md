# Configuración de Supabase para WiSer (Servitel)

Dado que la aplicación frontal "WiSer" es 100% estática (HTML/CSS/JS), el control de acceso, la persistencia de datos de los clientes y las **tareas automatizadas (facturación y suspensiones mes a mes)** se deben implementar en un backend as a service como **Supabase**.

A continuación, tienes las instrucciones exactas y el código SQL que debes ejecutar en Supabase para configurarlo desde cero.

---

## Paso 1: Crear el proyecto
1. Ve a [Supabase.com](https://supabase.com) y regístrate o inicia sesión.
2. Haz clic en **"New Project"**, asigna el nombre `Servitel WiSer`, y genera una contraseña segura para tu base de datos.
3. Elige la región más cercana a tu país (Ej. Sao Paulo o US East).

## Paso 2: Crear Tablas Iniciales
Ve al apartado **SQL Editor** en el menú izquierdo de Supabase, y haz clic en "New Query". Pega el siguiente código para crear la estructura relacional:

```sql
-- 1. Tabla de Planes de Internet
CREATE TABLE planes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    nombre TEXT NOT NULL,
    precio NUMERIC(10,2) NOT NULL,
    megas INTEGER NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 2. Tabla de Clientes
CREATE TABLE clientes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES auth.users, -- En caso de que el cliente pueda iniciar sesión, o para relacionarlo con quien lo creó.
    nombre TEXT NOT NULL,
    direccion TEXT,
    zona TEXT,
    ip TEXT UNIQUE,
    router TEXT,
    estado TEXT DEFAULT 'Activo' CHECK (estado IN ('Activo', 'Suspendido')),
    plan_id UUID REFERENCES planes(id),
    dia_corte INTEGER DEFAULT 1, -- Todos cobran y cortan el dia 1
    saldo_pendiente NUMERIC(10,2) DEFAULT 0.00,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. Tabla de Pagos/Facturación
CREATE TABLE pagos (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    cliente_id UUID REFERENCES clientes(id) ON DELETE CASCADE,
    monto NUMERIC(10,2) NOT NULL,
    fecha_pago TIMESTAMPTZ DEFAULT NOW(),
    concepto TEXT,
    metodo TEXT
);

-- 4. Tabla de Tickets (Soporte Técnico)
CREATE TABLE tickets (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    cliente_id UUID REFERENCES clientes(id) ON DELETE CASCADE,
    descripcion TEXT NOT NULL,
    estado TEXT DEFAULT 'Abierto' CHECK (estado IN ('Abierto', 'En Proceso', 'Cerrado')),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Insertar Datos Reales de Prueba para Servitel
INSERT INTO planes (nombre, precio, megas) VALUES 
('Plan Plata', 60000.00, 6),
('Plan Oro', 80000.00, 20),
('Plan Zafiro', 110000.00, 50);
```

Presiona **RUN** para crear todas las estructuras.

---

## Paso 3: Facturación y Cortes Automáticos (El Corazón de WiSer)

Necesitamos que **cada día 1 del mes** se genere el cobro anticipado, aumentando el saldo del cliente, y si no paga, el sistema lo marque como "Suspendido".
Utilizaremos `pg_cron` que viene integrado en PostgreSQL/Supabase.

### 1. Activar la extensión pg_cron (Solo si no está activa)
```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;
```

### 2. Crear una Función de Facturación
Esta función aumentará el saldo pendiente de todos los clientes según el valor de su plan actual.
```sql
CREATE OR REPLACE FUNCTION facturacion_mensual()
RETURNS void AS $$
BEGIN
    -- Actualizar saldo_pendiente sumándole el costo del plan para el inicio del mes
    UPDATE clientes c
    SET saldo_pendiente = c.saldo_pendiente + p.precio
    FROM planes p
    WHERE c.plan_id = p.id AND c.estado = 'Activo';
END;
$$ LANGUAGE plpgsql;
```

### 3. Crear una Función de Suspensión (Cortes Automáticos)
Si el cliente llega a tener saldo pendiente mayor a cero en determinada fecha límite (ejemplo, el día 5 del mes), se le cortará.
```sql
CREATE OR REPLACE FUNCTION corte_morosos()
RETURNS void AS $$
BEGIN
    UPDATE clientes
    SET estado = 'Suspendido'
    WHERE saldo_pendiente > 0 AND estado = 'Activo';
END;
$$ LANGUAGE plpgsql;
```

### 4. Programar los automatismos (CRON)
Enviaremos a ejecutar la generación de la cuota el primer día del mes a las 00:00.
Y programaremos el *corte automático* para el día 5 a las 02:00.

```sql
-- Facturar el dia 1 de todos los meses a las 00:00 (Midnight)
SELECT cron.schedule('facturacion_dia_1', '0 0 1 * *', 'SELECT facturacion_mensual()');

-- Cortar el servicio el dia 5 de todos los meses a las 02:00 AM a quienes deben
SELECT cron.schedule('corte_dia_5', '0 2 5 * *', 'SELECT corte_morosos()');
```

---

## Paso 4: Seguridad (RLS)
Por defecto te recomiendo que la autenticación solo permita acceder a la app web a usuarios registrados explícitamente en Auth (ya que no habrá niveles de "súperadmin", todos los usuarios de Auth de Servitel serán administradores de WiSer).

```sql
-- Habilitar RLS
ALTER TABLE clientes ENABLE ROW LEVEL SECURITY;
ALTER TABLE planes ENABLE ROW LEVEL SECURITY;
ALTER TABLE pagos ENABLE ROW LEVEL SECURITY;
ALTER TABLE tickets ENABLE ROW LEVEL SECURITY;

-- Crear política única: Todos los usuarios AUTENTICADOS pueden ver, editar, borrar y modificar todo
CREATE POLICY "Permitir todo a personal logueado" ON clientes FOR ALL USING (auth.role() = 'authenticated');
CREATE POLICY "Permitir todo a personal logueado" ON planes FOR ALL USING (auth.role() = 'authenticated');
CREATE POLICY "Permitir todo a personal logueado" ON pagos FOR ALL USING (auth.role() = 'authenticated');
CREATE POLICY "Permitir todo a personal logueado" ON tickets FOR ALL USING (auth.role() = 'authenticated');
```

## Paso 5: Cómo Integrarlo al index.html
Cuando quieras activar Supabase en el `index.html` que hemos creado, solo tienes que ir a tu Supabase en **Settings > API**, copiar tu `Project URL` y tu `anon public key` y usar el CDN en tu HTML:

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<script>
  const supabaseUrl = 'TU_URL_AQUI';
  const supabaseKey = 'TU_ANON_KEY_AQUI';
  const supabase = supabase.createClient(supabaseUrl, supabaseKey);
  // Al reemplazar esta sección en tu código actual de index.html, 
  // estarás conectado a tu BD real en la nube.
</script>
```
