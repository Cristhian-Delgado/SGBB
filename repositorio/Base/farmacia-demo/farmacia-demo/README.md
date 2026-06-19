# Demo Tema 9 — Microservicios, Patrones de Datos y Arquitectura Integradora
## IS-404 Administración de Bases de Datos Distribuidas 

---

##  Arquitectura

```
┌──────────────────────────────────────────────────────────────┐
│                     CADENA DE FARMACIAS                      │
│                                                              │
│  [Cliente/Postman]                                           │
│       │                                                      │
│       ▼                                                      │
│  ┌─────────────┐   POST /ventas    ┌──────────────────────┐  │
│  │Sales Service│ ──────────────►  │   MongoDB (ventas)   │  │
│  │  :8002      │                  └──────────────────────┘  │
│  └─────────────┘                                            │
│       │                                                      │
│       │  publica evento "venta.creada"                       │
│       ▼                                                      │
│  ┌─────────────┐  ← RABBITMQ (broker) →                     │
│  │  RabbitMQ   │       exchange: farmacia.eventos            │
│  │  :5672      │                                            │
│  └─────────────┘                                            │
│       │                    │                                 │
│       ▼                    ▼                                 │
│  ┌─────────────┐    ┌─────────────┐                         │
│  │  Inventory  │    │   Audit     │                         │
│  │  Service    │    │  Service    │                         │
│  │  :8001      │    │  :8003      │                         │
│  └──────┬──────┘    └──────┬──────┘                         │
│         │                  │                                 │
│         ▼                  ▼                                 │
│  ┌─────────────┐    ┌─────────────┐                         │
│  │ PostgreSQL  │    │ PostgreSQL  │                         │
│  │ inventario  │    │ auditoria   │                         │
│  │ :5433       │    │ :5434       │                         │
│  └─────────────┘    └─────────────┘                         │
└──────────────────────────────────────────────────────────────┘
```

---

##  Inicio rápido

### 1. Levantar todos los contenedores

```bash
cd farmacia-demo
docker compose up -d --build
```

### 2. Verificar que todo esté corriendo

```bash
docker compose ps
```

Debes ver 7 contenedores en estado **Up**:
- `farmacia_rabbitmq`
- `farmacia_inventory_db`
- `farmacia_sales_db`
- `farmacia_audit_db`
- `farmacia_inventory`
- `farmacia_sales`
- `farmacia_audit`

### 3. Ver logs en tiempo real (abrir otra terminal)

```bash
# Ver todos los servicios juntos
docker compose logs -f

# O solo un servicio
docker compose logs -f inventory-service
docker compose logs -f sales-service
docker compose logs -f audit-service
```

### 4. Ejecutar la demo de exposición

```bash
pip3 install requests
python demo.py
```

---

##  Comandos manuales para la demo

### Ver stock inicial
```bash
curl http://localhost:8001/productos | python -m json.tool
```

### Registrar una venta
```bash
Invoke-RestMethod -Method POST -Uri "http://localhost:8002/ventas" -ContentType "application/json" -Body '{"producto_id": 1, "nombre_producto": "Paracetamol 500mg", "cantidad": 5, "precio_unitario": 0.50, "sucursal": "Guayaquil-Norte", "cliente": "Maria Garcia"}'
```

### Verificar que el stock se descontó
```bash
Invoke-RestMethod http://localhost:8001/productos
```

### Ver el outbox del inventario (patrón outbox)
```bash
Invoke-RestMethod http://localhost:8001/outbox
```

### Ver registro de auditoría
```bash
Invoke-RestMethod http://localhost:8003/auditoria
```


### Ver ventas en MongoDB
```bash
Invoke-RestMethod http://localhost:8002/ventas
```

### Panel de RabbitMQ (interfaz web)
Abrir en navegador: http://localhost:15672
- Usuario: `admin`
- Contraseña: `admin123`

---

##  Conexión directa a las bases de datos

### PostgreSQL Inventario
```bash
docker exec -it farmacia_inventory_db psql -U inv_user -d inventario

-- Dentro de psql:
SELECT * FROM productos;
SELECT * FROM inventory_outbox;
```

### PostgreSQL Auditoría
```bash
docker exec -it farmacia_audit_db psql -U audit_user -d auditoria

-- Dentro de psql:
SELECT evento, COUNT(*) FROM audit_log GROUP BY evento;
SELECT * FROM audit_log ORDER BY recibido_en DESC LIMIT 10;
```

### MongoDB Ventas
```bash
docker exec -it farmacia_sales_db mongosh -u sales_user -p sales_pass --authenticationDatabase admin

# Dentro de mongosh:
use ventas
db.ventas.find().pretty()
db.ventas.countDocuments()
```

---

##  Apagar y limpiar

```bash
# Apagar contenedores
docker compose down

# Apagar y borrar volúmenes (datos)
docker compose down -v
```

--