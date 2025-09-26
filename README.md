# 🗄️ Clase MongoDB + Mongoose
## De Arrays en Memoria a Base de Datos Real

**Duración:** 2 horas  
**Objetivo:** Migrar la API de Torneos de Gaming de arrays en memoria a MongoDB usando Mongoose

---

## 📅 Cronograma de la Clase

| Tiempo | Actividad |
|--------|-----------|
| 0:00 - 0:15 | 🎯 Repaso y Configuración de MongoDB Atlas |
| 0:15 - 0:45 | 🐍 Instalación de Mongoose y Primer Modelo |
| 0:45 - 1:15 | 🔧 Migración de Endpoints GET y POST |
| 1:15 - 1:30 | ☕ **DESCANSO** |
| 1:30 - 2:00 | ✏️ Migración de PUT y DELETE + Pruebas |

---

## 🎯 Parte 1: Repaso y Configuración (15 min)

### 🔄 Repaso Rápido de la Clase Anterior

**¿Qué teníamos funcionando?**
- API de torneos con jugadores en arrays
- CRUD completo (GET, POST, PUT, DELETE)
- Validaciones básicas
- Filtros y búsquedas

**¿Cuál es el problema actual?**
- Los datos se pierden al reiniciar el servidor
- No hay persistencia real
- No escalable para múltiples usuarios

### 🌐 ¿Qué es MongoDB?

**Analogía del Archivador:**
- **Arrays en memoria** = Post-its que se pierden
- **MongoDB** = Archivador gigante que nunca se pierde
- **Mongoose** = El organizador que nos ayuda a encontrar todo
---

## 🐍 Parte 2: Instalación de Mongoose y Primer Modelo (30 min)

### 📦 Instalación de Dependencias

```bash
# En la terminal, dentro de la carpeta del proyecto
npm install mongoose
```

### 🏗️ Estructura de Carpetas Mejorada

```
proyecto-torneo/
├── models/
│   └── Jugador.js
├── package.json
└── servidor.js
```

### 📊 Creando el Modelo de Jugador

**Crear `models/Jugador.js`:**

```javascript
// models/Jugador.js
const mongoose = require('mongoose');

// Definir el esquema del jugador
const jugadorSchema = new mongoose.Schema({
    nickname: {
        type: String,
        required: [true, 'El nickname es obligatorio'],
        unique: true,
        trim: true,
        minlength: [3, 'El nickname debe tener al menos 3 caracteres'],
        maxlength: [20, 'El nickname no puede tener más de 20 caracteres']
    },
    juego: {
        type: String,
        required: [true, 'El juego es obligatorio'],
        enum: {
            values: ['League of Legends', 'CS:GO', 'Valorant', 'Fortnite', 'Dota 2', 'Overwatch', 'Apex Legends'],
            message: 'Juego no válido'
        }
    },
    nivel: {
        type: String,
        required: [true, 'El nivel es obligatorio'],
        enum: {
            values: ['Amateur', 'Semi-Pro', 'Pro'],
            message: 'Nivel debe ser Amateur, Semi-Pro o Pro'
        }
    },
    pais: {
        type: String,
        required: [true, 'El país es obligatorio'],
        trim: true
    }
});

module.exports = mongoose.model('Jugador', jugadorSchema);
```

### 🔌 Conectar MongoDB en el Servidor

**Actualizar `servidor.js`:**

```javascript
// servidor.js
const express = require('express');
const mongoose = require('mongoose');

const app = express();

// Middleware
app.use(express.json());

// URL de conexión a MongoDB Atlas (reemplazar con sus datos)
const MONGODB_URI = 'mongodb+srv://jacobogarcesoquendo:aFJzVMGN3o7fA38A@cluster0.mqwbn.mongodb.net/{nombre-mentor}';

// Conectar a MongoDB
const conectarDB = async () => {
    try {
        await mongoose.connect(MONGODB_URI);
        console.log('🍃 MongoDB conectado exitosamente');
    } catch (error) {
        console.error('❌ Error conectando MongoDB:', error.message);
        process.exit(1);
    }
};

// Llamar función de conexión
conectarDB();

// Importar modelo
const Jugador = require('./models/Jugador');

// Middleware para errores de MongoDB
app.use((error, req, res, next) => {
    if (error.name === 'ValidationError') {
        const errors = Object.values(error.errors).map(err => err.message);
        return res.status(400).json({ error: errors });
    }
    if (error.code === 11000) {
        return res.status(409).json({ error: 'Ese nickname ya está registrado' });
    }
    res.status(500).json({ error: 'Error interno del servidor' });
});

const PORT = 3000;

app.listen(PORT, () => {
    console.log(`🏆 API Torneo Gaming funcionando en puerto ${PORT}`);
});
```

### 🧪 Prueba de Conexión

**Ejecutar el servidor:**

```bash
npm start
# o
node servidor.js
```

**Deberías ver:**
```
🍃 MongoDB conectado exitosamente
🏆 API Torneo Gaming funcionando en puerto 3000
```

---

## 🔧 Parte 3: Migración de Endpoints GET y POST (30 min)

### 📖 Endpoint GET - Obtener Todos los Jugadores

**Agregar a `servidor.js`:**

```javascript
// GET - Obtener todos los jugadores con filtros
app.get('/jugadores', async (req, res) => {
    try {
        const { limite, juego, nivel, pais, buscar } = req.query;
        
        // Construir filtro dinámico
        let filtro = {};
        
        if (juego) {
            filtro.juego = new RegExp(juego, 'i'); // Búsqueda insensible a mayúsculas
        }
        
        if (nivel) {
            filtro.nivel = nivel;
        }
        
        if (pais) {
            filtro.pais = new RegExp(pais, 'i');
        }
        
        if (buscar) {
            filtro.nickname = new RegExp(buscar, 'i');
        }
        
        // Ejecutar consulta
        let query = Jugador.find(filtro);
        
        // Aplicar límite si existe
        if (limite) {
            query = query.limit(parseInt(limite));
        }
        
        const jugadores = await query.exec();
        const total = await Jugador.countDocuments(filtro);
        
        res.json({
            total,
            resultados: jugadores.length,
            jugadores
        });
        
    } catch (error) {
        console.error('Error obteniendo jugadores:', error);
        res.status(500).json({ error: 'Error interno del servidor' });
    }
});
```

### 🔍 Endpoint GET - Obtener Jugador por ID

```javascript
// GET - Obtener jugador por ID
app.get('/jugador/:id', async (req, res) => {
    try {
        const { id } = req.params;
        
        // Validar que el ID sea válido de MongoDB
        if (!mongoose.Types.ObjectId.isValid(id)) {
            return res.status(400).json({ error: 'ID no válido' });
        }
        
        const jugador = await Jugador.findById(id);
        
        if (!jugador) {
            return res.status(404).json({ error: 'Jugador no encontrado' });
        }
        
        res.json(jugador);
        
    } catch (error) {
        console.error('Error obteniendo jugador:', error);
        res.status(500).json({ error: 'Error interno del servidor' });
    }
});
```

### ➕ Endpoint POST - Registrar Nuevo Jugador

```javascript
// POST - Registrar nuevo jugador
app.post('/jugadores', async (req, res) => {
    try {
        const { nickname, juego, nivel, pais } = req.body;
        
        // Crear nueva instancia del modelo
        const nuevoJugador = new Jugador({
            nickname,
            juego,
            nivel,
            pais
        });
        
        // Guardar en la base de datos
        const jugadorGuardado = await nuevoJugador.save();
        
        res.status(201).json({
            mensaje: '¡Jugador registrado exitosamente en el torneo!',
            jugador: jugadorGuardado
        });
        
    } catch (error) {
        console.error('Error registrando jugador:', error);
        
        // Mongoose ya maneja las validaciones automáticamente
        // El middleware de errores se encarga de formatear la respuesta
        next(error);
    }
});
```

### 🧪 Pruebas con RapidAPI Client

**1. Registrar jugador:**
```json
POST http://localhost:3000/jugadores
Content-Type: application/json

{
    "nickname": "CodeMaster",
    "juego": "Valorant",
    "nivel": "Semi-Pro",
    "pais": "Colombia"
}
```

**2. Obtener todos los jugadores:**
```
GET http://localhost:3000/jugadores
```

**3. Buscar por filtros:**
```
GET http://localhost:3000/jugadores?juego=Valorant&nivel=Semi-Pro
```

### 🎮 Dinámica: "Poblando la Base de Datos"

**Actividad (15 min):**
- Cada estudiante registra 2-3 jugadores diferentes
- Prueban diferentes filtros y búsquedas
- Verifican que los datos persisten al reiniciar el servidor
- Comparan las respuestas con las versiones anteriores

---

## ☕ DESCANSO - 15 MINUTOS
*¡Momento perfecto para verificar que todos tienen datos en su MongoDB!*

---

## ✏️ Parte 4: Migración de PUT y DELETE (30 min)

### 🔄 Endpoint PUT - Actualizar Jugador

```javascript
// PUT - Actualizar jugador completo
app.put('/jugador/:id', async (req, res) => {
    try {
        const { id } = req.params;
        const { nickname, juego, nivel, pais } = req.body;
        
        // Validar ID
        if (!mongoose.Types.ObjectId.isValid(id)) {
            return res.status(400).json({ error: 'ID no válido' });
        }
        
        // Buscar y actualizar
        const jugadorActualizado = await Jugador.findByIdAndUpdate(
            id,
            { 
                nickname, 
                juego, 
                nivel, 
                pais 
            },
            { 
                new: true,          // Retorna el documento actualizado
                runValidators: true // Ejecuta las validaciones del schema
            }
        );
        
        if (!jugadorActualizado) {
            return res.status(404).json({ error: 'Jugador no encontrado' });
        }
        
        res.json({
            mensaje: 'Información del jugador actualizada exitosamente',
            jugador: jugadorActualizado
        });
        
    } catch (error) {
        console.error('Error actualizando jugador:', error);
        next(error);
    }
});
```

### 🗑️ Endpoint DELETE - Eliminar Jugador

```javascript
// DELETE - Eliminar jugador permanentemente
app.delete('/jugador/:id', async (req, res) => {
    try {
        const { id } = req.params;
        
        // Validar ID
        if (!mongoose.Types.ObjectId.isValid(id)) {
            return res.status(400).json({ error: 'ID no válido' });
        }
        
        // Eliminar permanentemente el jugador
        const jugadorEliminado = await Jugador.findByIdAndDelete(id);
        
        if (!jugadorEliminado) {
            return res.status(404).json({ error: 'Jugador no encontrado' });
        }
        
        // Contar participantes restantes
        const participantesRestantes = await Jugador.countDocuments();
        
        res.json({
            mensaje: 'Jugador eliminado permanentemente del torneo',
            jugador: {
                id: jugadorEliminado._id,
                nickname: jugadorEliminado.nickname,
                fechaEliminacion: new Date()
            },
            participantesRestantes: participantesRestantes
        });
        
    } catch (error) {
        console.error('Error eliminando jugador:', error);
        res.status(500).json({ error: 'Error interno del servidor' });
    }
});
```

### 🧪 Pruebas Completas

**Flujo de pruebas (15 min):**

1. **Actualizar jugador:**
```json
PUT http://localhost:3000/jugador/[ID_DEL_JUGADOR]
Content-Type: application/json

{
    "nickname": "CodeMaster_Pro",
    "juego": "Valorant",
    "nivel": "Pro",
    "pais": "Colombia"
}
```

3. **Eliminar jugador:**
```
DELETE http://localhost:3000/jugador/[ID_DEL_JUGADOR]
```

4. **Verificar que se eliminó completamente:**
```
GET http://localhost:3000/jugadores
```

### 👥 Ejercicio Final: "Torneo Completo"

**En parejas (10 min):**
1. **Persona A:** Registra 3 jugadores de diferentes países
2. **Persona B:** Actualiza el nivel de uno a "Pro"
3. **Ambos:** Consultan estadísticas del torneo
4. **Persona A:** Elimina un jugador permanentemente
5. **Persona B:** Verifica los cambios y estadísticas finales

---

## 📋 Código Completo Final

**Archivo `servidor.js` completo:**

```javascript
// servidor.js - API Torneo Gaming con MongoDB
const express = require('express');
const mongoose = require('mongoose');

const app = express();

// Middleware
app.use(express.json());

// URL de conexión a MongoDB Atlas
const MONGODB_URI = 'mongodb+srv://username:password@cluster.mongodb.net/torneo-gaming?retryWrites=true&w=majority';

// Conectar a MongoDB
const conectarDB = async () => {
    try {
        await mongoose.connect(MONGODB_URI);
        console.log('🍃 MongoDB conectado exitosamente');
    } catch (error) {
        console.error('❌ Error conectando MongoDB:', error.message);
        process.exit(1);
    }
};

conectarDB();

// Importar modelo
const Jugador = require('./models/Jugador');

// Middleware para errores de MongoDB
app.use((error, req, res, next) => {
    if (error.name === 'ValidationError') {
        const errors = Object.values(error.errors).map(err => err.message);
        return res.status(400).json({ error: errors });
    }
    if (error.code === 11000) {
        return res.status(409).json({ error: 'Ese nickname ya está registrado' });
    }
    res.status(500).json({ error: 'Error interno del servidor' });
});

// RUTAS

// GET - Obtener todos los jugadores con filtros
app.get('/jugadores', async (req, res) => {
    try {
        const { limite, juego, nivel, pais, buscar } = req.query;
        
        let filtro = {};
        
        if (juego) filtro.juego = new RegExp(juego, 'i');
        if (nivel) filtro.nivel = nivel;
        if (pais) filtro.pais = new RegExp(pais, 'i');
        if (buscar) filtro.nickname = new RegExp(buscar, 'i');
        
        let query = Jugador.find(filtro);
        
        if (limite) query = query.limit(parseInt(limite));
                
        const jugadores = await query.exec();
        const total = await Jugador.countDocuments(filtro);
        
        res.json({
            total,
            resultados: jugadores.length,
            jugadores
        });
        
    } catch (error) {
        console.error('Error obteniendo jugadores:', error);
        res.status(500).json({ error: 'Error interno del servidor' });
    }
});

// GET - Obtener jugador por ID
app.get('/jugador/:id', async (req, res) => {
    try {
        const { id } = req.params;
        
        if (!mongoose.Types.ObjectId.isValid(id)) {
            return res.status(400).json({ error: 'ID no válido' });
        }
        
        const jugador = await Jugador.findById(id);
        
        if (!jugador) {
            return res.status(404).json({ error: 'Jugador no encontrado' });
        }
        
        res.json(jugador);
        
    } catch (error) {
        console.error('Error obteniendo jugador:', error);
        res.status(500).json({ error: 'Error interno del servidor' });
    }
});

// POST - Registrar nuevo jugador
app.post('/jugadores', async (req, res, next) => {
    try {
        const { nickname, juego, nivel, pais } = req.body;
        
        const nuevoJugador = new Jugador({
            nickname,
            juego,
            nivel,
            pais
        });
        
        const jugadorGuardado = await nuevoJugador.save();
        
        res.status(201).json({
            mensaje: '¡Jugador registrado exitosamente en el torneo!',
            jugador: jugadorGuardado
        });
        
    } catch (error) {
        next(error);
    }
});

// PUT - Actualizar jugador
app.put('/jugador/:id', async (req, res, next) => {
    try {
        const { id } = req.params;
        const { nickname, juego, nivel, pais } = req.body;
        
        if (!mongoose.Types.ObjectId.isValid(id)) {
            return res.status(400).json({ error: 'ID no válido' });
        }
        
        const jugadorActualizado = await Jugador.findByIdAndUpdate(
            id,
            { nickname, juego, nivel, pais },
            { new: true, runValidators: true }
        );
        
        if (!jugadorActualizado) {
            return res.status(404).json({ error: 'Jugador no encontrado' });
        }
        
        res.json({
            mensaje: 'Información del jugador actualizada exitosamente',
            jugador: jugadorActualizado
        });
        
    } catch (error) {
        next(error);
    }
});

// DELETE - Eliminar jugador permanentemente
app.delete('/jugador/:id', async (req, res) => {
    try {
        const { id } = req.params;
        
        if (!mongoose.Types.ObjectId.isValid(id)) {
            return res.status(400).json({ error: 'ID no válido' });
        }
        
        const jugadorEliminado = await Jugador.findByIdAndDelete(id);
        
        if (!jugadorEliminado) {
            return res.status(404).json({ error: 'Jugador no encontrado' });
        }
        
        const participantesRestantes = await Jugador.countDocuments();
        
        res.json({
            mensaje: 'Jugador eliminado permanentemente del torneo',
            jugador: {
                id: jugadorEliminado._id,
                nickname: jugadorEliminado.nickname,
                fechaEliminacion: new Date()
            },
            participantesRestantes: participantesRestantes
        });
        
    } catch (error) {
        console.error('Error eliminando jugador:', error);
        res.status(500).json({ error: 'Error interno del servidor' });
    }
});

const PORT = 3000;

app.listen(PORT, () => {
    console.log(`🏆 API Torneo Gaming funcionando en puerto ${PORT}`);
});
```

> 📝 **Nota:** Recuerden reemplazar la URL de MongoDB con sus propios datos de Atlas.

---

## 📚 Resumen de lo Aprendido

### 🎯 Conceptos Clave de MongoDB + Mongoose:

- **Conexión a MongoDB Atlas:** Configuración de cluster en la nube con URL hardcodeada
- **Modelos y Esquemas:** Definición de estructura de datos
- **Validaciones:** Rules automáticas en el nivel de base de datos
- **Operaciones CRUD:** Métodos de Mongoose para interactuar con MongoDB
- **Consultas avanzadas:** Filtros, ordenamiento, agregaciones
- **Eliminación física:** Borrado permanente de registros con `findByIdAndDelete()`
- **Manejo de errores:** Middleware específico para errores de MongoDB

### 🛠️ Habilidades Prácticas Adquiridas:

- ✅ Configurar MongoDB Atlas desde cero
- ✅ Crear modelos robustos con validaciones
- ✅ Migrar de arrays a base de datos real
- ✅ Implementar filtros y búsquedas eficientes
- ✅ Manejar errores de validación y duplicados
- ✅ Crear consultas de agregación para estadísticas
- ✅ Conectar directamente sin variables de entorno

### 🚀 Diferencias Clave: Arrays vs MongoDB

| Aspecto | Arrays en Memoria | MongoDB + Mongoose |
|---------|------------------|-------------------|
| **Persistencia** | Se pierde al reiniciar | Datos permanentes |
| **Validaciones** | Manuales en código | Automáticas en esquema |
| **Escalabilidad** | Limitada | Prácticamente ilimitada |
| **Consultas** | Loops y filtros manuales | Optimizadas por MongoDB |
| **Eliminación** | Splice del array | Eliminación física en BD |
| **Concurrencia** | Problemática | Manejada automáticamente |
| **Backup** | No existe | Automático en Atlas |


## 🎊 ¡Felicidades Database Masters!

**Han migrado exitosamente de arrays temporales a una base de datos real y profesional**  
**Su API de torneos ahora es escalable, persistente y robusta 🚀**

### Tarea para Casa:
1. **Experimenten:** Agreguen más jugadores y exploren las estadísticas
2. **Investiguen:** ¿Qué otras validaciones podrían ser útiles para un torneo real?
3. **Prepárense:** Para la próxima clase crearemos el frontend en React para visualizar todos estos datos

---

**¡Ahora están listos para construir APIs profesionales con MongoDB! 🗄️✨**
