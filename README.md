# CRM ORM/ODM Lab

API REST de un CRM básico que combina un ORM (Sequelize + PostgreSQL) y un ODM (Mongoose + MongoDB).

## Stack

- Node.js 22, Express 5, CommonJS
- Sequelize + PostgreSQL 16 (`User`, `Company`, `Contact`)
- Mongoose + MongoDB 7 (`Activity`)
- Jest + Supertest
- GitHub Codespaces, Dev Containers, Docker Compose
- Supervisor (`npm run dev`)

## Arquitectura

```text
GitHub Codespace
│
├── app       Node.js 22  ──┬── Sequelize ──> postgres (PostgreSQL)
│                           └── Mongoose  ──> mongo    (MongoDB)
├── postgres
└── mongo
```

La aplicación se conecta por nombre de servicio (`postgres`, `mongo`). Las credenciales de desarrollo llegan como variables de entorno definidas en `.devcontainer/docker-compose.yml` (ver `.env.example`).

## Iniciar el Codespace

1. En GitHub: **Code → Codespaces → Create codespace on main**.
2. Espera a que se levanten los tres servicios (`app`, `postgres`, `mongo`). `postCreateCommand` ejecuta `npm install`.

## Instalar dependencias

```bash
npm install
```

## Seed y reset

```bash
npm run seed    # inserta datos deterministas (3 users, 4 companies, 8 contacts, 10 activities)
npm run reset   # elimina y recrea tablas/base de datos y vuelve a sembrar
```

## Iniciar la API

```bash
npm start       # node ./bin/www
npm run dev     # supervisor ./bin/www
```

Servidor en el puerto `3000` (variable `PORT`).

## Pruebas

```bash
npm test
```

Cada suite restablece PostgreSQL y MongoDB antes de ejecutarse y cierra las conexiones al terminar.

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Health check |
| GET | `/users` | Listar usuarios |
| GET | `/users/:id` | Obtener usuario |
| POST | `/users` | Crear usuario |
| PUT | `/users/:id` | Actualizar usuario |
| DELETE | `/users/:id` | Eliminar usuario |
| GET | `/companies` | Listar compañías (`?industry=`) |
| GET | `/companies/:id` | Obtener compañía |
| POST | `/companies` | Crear compañía |
| PUT | `/companies/:id` | Actualizar compañía |
| DELETE | `/companies/:id` | Eliminar compañía |
| GET | `/contacts` | Listar contactos |
| GET | `/contacts/:id` | Obtener contacto |
| POST | `/contacts` | Crear contacto |
| PUT | `/contacts/:id` | Actualizar contacto |
| DELETE | `/contacts/:id` | Eliminar contacto |
| GET | `/activities` | Listar actividades (`?type=`) |
| GET | `/activities/:id` | Obtener actividad |
| POST | `/activities` | Crear actividad |
| PUT | `/activities/:id` | Actualizar actividad |
| DELETE | `/activities/:id` | Eliminar actividad |

Los errores se devuelven como JSON: `{ "error": "Contact not found" }`.

## Respuestas
**1. Dos motores.**
   Activity resulta favorable para una base documental porque sus metadatos cambian según el tipo que sean. La forma en la que Company y Contact están estructurados es más fija, usan llaves foráneas y permiten hacer consultas utilizando joins.

**2. ORM vs ODM.**
   Un ORM mapea tablas relacionales a objetos. En este caso, la librería utilizada es Sequelize. Un ODM mapea documentos de una base documental a objetos. Se utiliza la librería Mongoose. Se diferencian en que el ORM trabaja con esquemas y relaciones rígidas impuestas por la base de datos, mientras el ODM trabaja con documentos flexibles y el esquema vive únicamente en la aplicación.

**3. Configuración por variables de entorno.**
   Estas están definidas en:
   ```bash
   .devcontainer/docker-compose.yml
   ```
   en la sección environment del servicio app. Los archivos .js solo las leen con process.env.

Se considera mala práctica escribir las variables de entorno en los .js porque como el código se sube en git, las credenciales quedarían expuestas en el repositorio Hosts que usa la app: DB_HOST = postgres MONGODB_URI = mongodb://mongo:27017/crm. 

La razón por la que no son localhost es porque PostgreSQL y MongoDB corren en contenedores distintos al de la app.

**4. Asociaciones.**
Relación: uno a muchos. Una compañía tiene muchos contactos (Company.hasMany(Contact)) y cada contacto pertenece a una sola compañía (Contact.belongsTo(Company)).
Llave foránea: companyId que vive en la tabla contacts.
Alias as: 'contacts': es el nombre con el que se accede a la relación. Se usa en el include y es la propiedad que aparece en el JSON (company.contacts).

**5. Eager loading.**
   Haciendo dos consultas primero se trae la compañía y luego los contactos con otra consulta. Utilizando include, Sequelize trae todo en una sola consulta y arma el objeto con sus contacts anidados. Lo más preferible es utilizar el include, ya que el código es más simple y se hacen menos "vueltas" a la base de datos.

**6. Instancia vs Consulta.**
   Buscar y modificar devuelve la instancia ya actualizada, ejecuta las validaciones del modelo y permite devolver 404 si el registro no existe. Con Model.update({...}, { where }) directo se hace una sola consulta UPDATE y se logra una mayor eficiencia, pero este no devuelve el registro, solo cuantas filas fueron afectadas.

**7. Esquema flexible.**
   El tipo de dato utilizado es mongoose.Schema.Types.Mixed, que acepta cualquier valor u objeto. Gracias a eso, una CALL puede guardar {duration}, un EMAIL {subject} y un MEETING { attendees:[] }. La desventaja aquí está en que Mongoose no valida ni convierte los campos de metadata.

**8. Sin ref.**
   Se debe a que no se pueden usar ref/populate: porque solo funcionan entre colecciones de MongoDB. Esto tiene como consecuencia que MongoDB no valida los IDs y no los marca como existentes, además de que no reacciona a cambios en PostgreSQL.

**9. Documento actualizado.**
    Antes devolvía el documento sin actualizar, pues la actualización si se guardaba en la base de datos pero la respuesta mostraba el estado anterior. Esto se debe a que, por defecto, findByIdAndUpdate devuelve el documento tal como estaba antes del cambio. Lo que se cambió fue que se agregó la opción new:true, la cual le indica a Mongoose que devuelva el documento ya modificado, y runValidators:true para que las validaciones dle esquema se apliquen en la actualización.

**10. Pruebas de comportamiento.**
    La ventaja es que permite cambiar la implementación sin romper las pruebas. Es indiferente si uso findAll, find, o una consulta SQL; si el resultado es correcto, entonces la prueba pasa.

**11. Repetibilidad.**
    Antes de las suites, (beforeAll): contecta con PostgreSQL y MongoDB, y ejecuta reset(), restableciendo así los datos semilla. Después de las suites, (afterAll): cierra ambas conexiones. Se necesita el mismo resultado porque varias pruebas modifican datos (PUT, POST). Sin el reset(), una suite heredaría los cambios de la anterior y cada ejecución tendría un resultado diferente.

**12. Tu experiencia.**
    El desafío que me pareció más difícil fue el 8. Lo logre resolver al agregar new : true; y runValidators : true; a la constante activity. El mensaje de fallo que me ayudó era:
    Expected: "Llamada actualizada"
    Received: "Llamada de seguimiento"

## Evidencia
  
