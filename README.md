# MongoDB — Plataforma Airbnb

Este documento detalla el análisis de la arquitectura de la base de datos utilizada para la plataforma tipo Airbnb y las recomendaciones de mejora para un entorno de producción.

---

## 1. Problemas potenciales del modelo de datos actual

Tras analizar la colección `listingsAndReviews`, se han identificado los siguientes **riesgos críticos** si este modelo se implementara en un entorno de producción a gran escala:

---

#### Problema 1 — Crecimiento ilimitado del array de reseñas (`reviews`)

La estructura actual anida **todas las reseñas dentro del documento principal**, lo que puede generar documentos excesivamente grandes con el tiempo, llegando incluso al **límite de tamaño permitido por MongoDB (16 MB)**.

**✅ Solución propuesta:**
Optimizar el array para almacenar únicamente las últimas reseñas o las `top_reviews` más relevantes, y realizar una consulta independiente solo cuando el usuario solicite verlas todas.

---

#### Problema 2 — Inconsistencia de datos en amenities (`amenities`)

El campo `amenities` es un array de strings con **valores duplicados o equivalentes**, como `"wifi"` e `"internet"` tratados como servicios distintos. Esto provoca que búsquedas por `"internet"` descarten propiedades que solo tienen `"wifi"`, generando **resultados incompletos e inconsistentes**.

**✅ Solución propuesta:**
Extraer las amenidades a una **colección separada** con un proceso de normalización en tres pasos:

1. Identificar todos los términos que hacen referencia al mismo servicio (ej.: `"Wifi"`, `"Internet"`, `"Acceso a red"`, `"WLAN"`).
2. Crear un único registro en la nueva colección: `{ _id: 1, nombre: "Conectividad (Wifi/Internet)" }`.
3. Actualizar todos los listings para que apunten al `_id` correspondiente en lugar de almacenar el string directamente.

De esta forma se **elimina la ambigüedad** en los nombres y se reduce la redundancia de datos.

---

#### Problema 3 — Datos de disponibilidad obsoletos (`availability`)

La presencia del campo `calendar_last_scraped` indica que los datos son una **fotografía de un momento pasado**. Si el proceso de actualización falla o se ralentiza, el sistema seguirá sirviendo datos desactualizados, lo que podría llevar al usuario a intentar una **reserva sobre fechas ya ocupadas**.

**✅ Solución propuesta:**
Implementar un patrón de actualización reactiva sin cambios estructurales radicales:

- **Caché para búsquedas:** Mantener los campos actuales para las búsquedas generales, aprovechando su velocidad.
- **Validación síncrona en el momento de reserva:** Forzar una consulta real contra la fuente de verdad únicamente cuando el usuario selecciona fechas y confirma la reserva.
- **Refresco reactivo:** Aprovechar esa consulta de validación para actualizar automáticamente los campos `availability_X` y `calendar_last_scraped`, garantizando que la caché se refresque con cada reserva exitosa sin necesidad de procesos batch costosos.

---

#### Problema 4 — Redundancia en los datos del anfitrión (`host`)

El modelo actual incluye el objeto `host` **embebido dentro de cada documento listing**. Esto provoca una **redundancia masiva de datos**: si un anfitrión gestiona varias propiedades, la misma información se repite en cada una. Además, si el anfitrión actualiza su foto o su estado de verificación, el sistema debe realizar una **operación de escritura costosa en todos los documentos** donde aparezca.

**✅ Solución propuesta:**
Normalizar los datos mediante el desacoplamiento:

- **Creación de una colección `hosts`:** Extraer toda la información del anfitrión a una colección dedicada, donde cada perfil tenga un `host_id` único.
- **Referenciación en el listing:** El documento solo almacenará el `host_id` y únicamente los campos que el frontend requiera de forma inmediata (ej.: nombre y `thumbnail_url` para la tarjeta de la propiedad).
- **Consulta bajo demanda:** Si el usuario accede al perfil completo del anfitrión, el sistema realizará un $lookup hacia la colección `hosts`.

## Consultas

- Saca en una consulta cuantos alojamientos hay en España.

`db.listingsAndReviews.countDocuments({"address.country": "Spain"})`

También podríamos hacerlo de esta manera:

```
    db.listingsAndReviews.aggregate([
  { $match: { "address.country": "Spain" } },
  { $count: "total_alojamientos" }
])
```

- Lista los 10 primeros alojamientos de España:
  Ordenados por precio de forma ascendente.
  Sólo muestra: nombre, precio, camas y la localidad (address.market).

```db.listingsAndReviews.aggregate([
  { $match: { "address.country": "Spain" } },
  { $sort: {price:1}},
  { $limit: 10 },
  { $project: {
    _id:0, name:1,price:1, beds:1, "address.market":1
  }}
])

```

## 4. Filtrando

- Queremos viajar cómodos, somos 4 personas y queremos:
  4 camas.
  Dos cuartos de baño o más.
  Sólo muestra: nombre, precio, camas y baños.

```db.listingsAndReviews.aggregate(
  [
    {
      $match: {
        beds: {
          $eq: 4
        },
        bathrooms:{
          $gte: 2
        }
      }
    },
     {
      $project: {
        '_id': 0,
        'name': 1,
        'price': 1,
        'beds': 1,
        'bathrooms': 1
      }
    }
  ]
)
```

Podríamos plantearnos el decimal de bathrooms pasarlo a entero, para mostrarlo en el resultado final:

```
 bathrooms: { $toInt: "$bathrooms" }

```

- Aunque estamos de viaje no queremos estar desconectados, así que necesitamos que el alojamiento también tenga conexión wifi. A los requisitos anteriores, hay que añadir que el alojamiento tenga wifi.
  Sólo muestra: nombre, precio, camas, baños y servicios (amenities).

```
db.listingsAndReviews.aggregate(
  [
    {
      $match: {
        beds: {
          $eq: 4
        },
        bathrooms:{
          $gte: 2
        },
        amenities: "Wifi"
      }
    },
     {
      $project: {
        '_id': 0,
        'name': 1,
        'price': 1,
        'beds': 1,
        bathrooms: 1,
        amenities:1
      }
    }
  ]
)

```

- Y bueno, un amigo trae a su perro, así que tenemos que buscar alojamientos que permitan mascota (Pets allowed).
  Sólo muestra: nombre, precio, camas, baños y servicios (amenities).

```

db.listingsAndReviews.aggregate(
  [
    {
      $match: {
        beds: {
          $eq: 4
        },
        bathrooms:{
          $gte: 2
        },
        amenities: {$all: ["Wifi", "Pets allowed"]}
      }
    },
     {
      $project: {
        '_id': 0,
        'name': 1,
        'price': 1,
        'beds': 1,
        bathrooms: 1,
        amenities:1
      }
    }
  ]
)

```

- Estamos entre ir a Barcelona o a Portugal, los dos destinos nos valen. Pero queremos que el precio nos salga baratito (50 $), y que tenga buen rating de reviews (campo review_scores.review_scores_rating igual o superior a 88).
  Sólo muestra: nombre, precio, camas, baños, rating, localidad y país.

```

db.listingsAndReviews.aggregate(
  [
    {
      $match: {
        beds: {
          $eq: 4
        },
        bathrooms:{
          $gte: 2
        },
        amenities: {$all: ["Wifi", "Pets allowed"]},
        $or: [
          { "address.market": "Barcelona" },
          { "address.country": "Portugal" }
        ],
        price: {$lte:50},
        "review_scores.review_scores_rating" : {$gte:88}
      }
    },
     {
      $project: {
        '_id': 0,
        'name': 1,
        'price': 1,
        'beds': 1,
        bathrooms: 1,
        rating: "$review_scores.review_scores_rating",
        localidad:"$address.market",
        pais:"$address.country"

      }
    }
  ]
)

```

- También queremos que el huésped sea un superhost (host.host_is_superhost) y que no tengamos que pagar depósito de seguridad (security_deposit).
  Sólo muestra: nombre, precio, camas, baños, rating, si el huésped es superhost, depósito de seguridad, localidad y país.

```
db.listingsAndReviews.aggregate(
  [
    {
      $match: {
        beds:4,
        bathrooms:{
          $gte: 2
        },
        amenities: {$all: ["Wifi", "Pets allowed"]},
        $or: [
          { "address.market": "Barcelona" },
          { "address.country": "Portugal" }
        ],
        price: {$lte:50},
        "review_scores.review_scores_rating" : {$gte:88},
        "host.host_is_superhost":true,
        "security_deposit": 0
      }
    },
     {
      $project: {
        '_id': 0,
        'name': 1,
        'price': 1,
        'beds': 1,
        bathrooms: 1,
        rating: "$review_scores.review_scores_rating",
        isSuperhost:"$host.host_is_superhost",
        localidad:"$address.market",
        pais:"$address.country",
        security_deposit:1
      }
    }
  ]
)
```

## Agregaciones

- Queremos mostrar los alojamientos que hay en España, con los siguientes campos:
  Nombre.
  Localidad (no queremos mostrar un objeto, sólo el string con la localidad).
  Precio

```

db.listingsAndReviews.aggregate([
  {
    $match: {
      "address.country": "Spain"
    }
  },
  {
    $project: {
      _id:0,
      nombre: '$name',
      localidad:'$address.market',
      precio: "$price"
    }
  }
])

```

Igual que anteriormente podriamos poner el precio en formato entero con $toInt

- Queremos saber cuantos alojamientos hay disponibles por pais.

```

db.listingsAndReviews.aggregate(
  [
    {
      $group: {
        _id: '$address.country',
        numAlojamientos: {
          $sum: 1
        }
      }
    }
  ]
)

```

## Opcional

- Queremos saber el precio medio de alquiler de airbnb en España.

```
db.listingsAndReviews.aggregate([
  {
    $match: {
      "address.country": "Spain"
    },
  },
 {
  $group: {
    _id:'$address.country', averagePrice: { $avg: "$price" }
  }
 }
])
```

- ¿Y si quisieramos hacer como el anterior, pero sacarlo por paises?

```
db.listingsAndReviews.aggregate([
  {
    $group: {
      _id: '$address.country',
      averagePrice: { $avg: '$price' },
    },
  },
]);
```

- Repite los mismos pasos para calcular el precio medio de alquiler, pero agrupando también por numero de habitaciones.

```
db.listingsAndReviews.aggregate([
  {
    $group: {
      _id: {
        pais:'$address.country',
        habitaciones: '$bedrooms'
      },
      averagePrice: { $avg: '$price' },
    },
  },
]);

```

Aquí sería interesante añadir una ordenación por país y por habitaciones para que sea mas legible:

```

db.listingsAndReviews.aggregate([
  {
    $group: {
      _id: {
        pais:'$address.country',
        habitaciones: '$bedrooms'
      },
      averagePrice: { $avg: '$price' },
    },
  },
  {
    $sort:{
    "_id.pais": 1,
    "_id.habitaciones": 1
    }
  },
]);
```

## Desafio

Queremos mostrar el top 5 de alojamientos más caros en España, con los siguentes campos:

Nombre.
Precio.
Número de habitaciones
Número de camas
Número de baños
Ciudad.
Servicios, pero en vez de un array, un string con todos los servicios incluidos.

```
db.listingsAndReviews.aggregate([
  {
    $match: {
      "address.country": "Spain"
    }
  },
  {
    $sort: {
      price: -1
    }
  },
  {
    $limit:5
  },
  {
    $project: {
      _id:0,
      name:1,
      precio: '$price',
      habitaciones: "$bedrooms",
      camas: "$beds",
      baños: "$bathrooms",
      ciudad: "$address.market",
      servicios: {
        $reduce: {
          input: "$amenities",
          initialValue: "",
          in: {
            $cond: [
              { $eq: ["$$value", ""] },
              "$$this",
              { $concat: ["$$value",", ", "$$this"] }
            ]
          }
        }
      }
    }
  },
]);
```
