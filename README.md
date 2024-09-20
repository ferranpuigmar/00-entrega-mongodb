# Laboratorio MongoDB

## Introducción

En este base de datos puedes encontrar un montón de alojamientos y sus reviews, esto está sacado de hacer webscrapping.

**Pregunta**. Si montaras un sitio real, ¿Qué posibles problemas pontenciales les ves a como está almacenada la información?

```md
Tener todas las reviews en un misma collección hace que estas ocupen demasiado y depende de la consulta podría acabarse el espacio del working set muy rápido lo que afectaría a la performance. Estaría bien usar el patrón subset Patter y dejar las top reviews y si el usuario quiere ver más ya haríamos la petición con la collection de reviews.
```

## Obligatorio

Esta es la parte mínima que tendrás que entregar para superar este laboratorio.

### Consultas

- Saca en una consulta cuantos alojamientos hay en España.

```js
db.listingsAndReviews.find({ "address.country": "Spain"}).count()
```

- Lista los 10 primeros:
  - Ordenados por precio de forma ascendente.
  - Sólo muestra: nombre, precio, camas y la localidad (`address.market`).

```js
db.listingsAndReviews.find(
    { "address.country": "Spain", }, 
    { 
        _id: 0, 
        name: 1, 
        price: 1, 
        beds: 1, 
        "address.market": 1
    }
)
.sort({ "price": 1 })
.limit(10)
```

### Filtrando

- Queremos viajar cómodos, somos 4 personas y queremos:
  - 4 camas.
  - Dos cuartos de baño o más.
  - Sólo muestra: nombre, precio, camas y baños.

```js
db.listingsAndReviews.find(
    {
        beds: {
            $eq: 4
        },
        bathrooms: {
            $gte: 2
        }
    },
    {
        _id: 0,
        name: 1,
        price: 1,
        beds: 1,
        bathrooms: 1

    }
)
```

- Aunque estamos de viaje no queremos estar desconectados, así que necesitamos que el alojamiento también tenga conexión wifi. A los requisitos anteriores, hay que añadir que el alojamiento tenga wifi.
  - Sólo muestra: nombre, precio, camas, baños y servicios (`amenities`).

```js
db.listingsAndReviews.find(
    {
        beds: {
            $eq: 4
        },
        bathrooms: {
            $gte: 2
        },
        amenities: {
            $in: ["Wifi"]
        }
    }
    ,{
        _id: 0,
        name: 1,
        price: 1,
        beds: 1,
        amenities: 1
    }
)
```

- Y bueno, un amigo trae a su perro, así que tenemos que buscar alojamientos que permitan mascota (_Pets allowed_).
  - Sólo muestra: nombre, precio, camas, baños y servicios (`amenities`).

Con los requerimientos previos:

```js
db.listingsAndReviews.find(
    {
        beds: {
            $eq: 4
        },
        bathrooms: {
            $gte: 2
        },
        amenities: {
            $all: ["Wifi", "Pets allowed"]
        }
    }
    ,{
        _id: 0,
        name: 1,
        price: 1,
        beds: 1,
        amenities: 1,
        bathrooms: 1
    }
)
```

Sin los requerimientos previos:

```js
db.listingsAndReviews.find(
    {
        amenities: {
            $all: ["Wifi", "Pets allowed"]
        }
    }
    ,{
        _id: 0,
        name: 1,
        price: 1,
        beds: 1,
        amenities: 1,
        bathrooms: 1
    }
)
```

- Estamos entre ir a Barcelona o a Portugal, los dos destinos nos valen. Pero queremos que el precio nos salga baratito (50 $), y que tenga buen rating de reviews (campo `review_scores.review_scores_rating` igual o superior a 88).
  - Sólo muestra: nombre, precio, camas, baños, rating y localidad.

```js
db.listingsAndReviews.find(
    {
        price: {
            $lte: 50
        },
        "address.country": {
            $in: ["Spain", "Portugal"]
        },
        "host.host_location": { $regex: "Portugal|Barcelona" },
        "review_scores.review_scores_rating": { $gte: 88 }
    },
    {
        _id: 0,
        name: 1,
        price: 1,
        beds: 1,
        bathrooms: 1,
        "review_scores.review_scores_rating": 1,
        "host.host_location": 1
    }
)
```

- También queremos que el huésped sea un superhost (`host.host_is_superhost`) y que no tengamos que pagar depósito de seguridad (`security_deposit`).
  - Sólo muestra: nombre, precio, camas, baños, rating, si el huésped es superhost, depósito de seguridad y localidad.

```js
db.listingsAndReviews.find(
    {
        price: {
            $lte: 50
        },
        "address.country": {
            $in: ["Spain", "Portugal"]
        },
        "host.host_location": { $regex: "Portugal|Barcelona" },
        "review_scores.review_scores_rating": { $gte: 88 },
        "host.host_is_superhost": true,
        security_deposit: 0
    },
    {
        _id: 0,
        name: 1,
        price: 1,
        beds: 1,
        bathrooms: 1,
        "review_scores.review_scores_rating": 1,
        "host.host_is_superhost": 1,
        security_deposit: 1,
        "host.host_location": 1
    }
)
```

### Agregaciones

- Queremos mostrar los alojamientos que hay en España, con los siguientes campos:
  - Nombre.
  - Localidad (no queremos mostrar un objeto, sólo el string con la localidad).
  - Precio

```js
db.listingsAndReviews.aggregate([
    {
        $match: {
            "address.country": {
                $eq: "Spain"
            }
        },
    },
    {
        $project: {
          name: 1,
          location: "$host.host_location",
          price: 1
        }
    }
])
```

- Queremos saber cuantos alojamientos hay disponibles por pais.

```js
db.listingsAndReviews.aggregate([
    {
        $project: {
          "address.country": 1,
          _id: 0
        }
    },
    {
        $group: {
          _id: "$address.country",
          alojamientos: {
            $sum: 1
          }
        }
    }
])
```

## Opcional

- Queremos saber el precio medio de alquiler de airbnb en España.

```js
db.listingsAndReviews.aggregate([
  {
      $match: {
        "address.country": {
          $in: ["Spain"]
        }
      }
  },
  {
      $group: {
        _id: "$address.country",
        preciomedio: {
          $avg: "$price"
        }
      }
  }
])

```

- ¿Y si quisieramos hacer como el anterior, pero sacarlo por paises?

```js
db.listingsAndReviews.aggregate([
  {
      $group: {
        _id: "$address.country",
        preciomedio: {
          $avg: "$price"
        }
      }
  }
])
```

- Repite los mismos pasos pero agrupando también por numero de habitaciones.

Habitaciones promedio en España:

```js
db.listingsAndReviews.aggregate([
  {
    $match: {
      "address.country": { $in: ["Spain"] }
    }
  },
  {
      $project: {
        bedrooms: 1,
        "address.country": 1,
        _id: 0
      }
  },
  {
      $group: {
        _id: "$address.country",
        habitaciones_promedio: {
          $avg: "$bedrooms"
        }
      }
  }
])
```
Habitaciones promedio por país:

```js
db.listingsAndReviews.aggregate([
    {
        $project: {
          bedrooms: 1,
          "address.country": 1,
          _id: 0
        }
    },
    {
        $group: {
          _id: "$address.country",
          habitaciones_promedio: {
            $avg: "$bedrooms"
          }
        }
    }
])
```
## Desafio

Queremos mostrar el top 5 de alojamientos más caros en España, con los siguentes campos:

- Nombre.
- Precio.
- Número de habitaciones
- Número de camas
- Número de baños
- Ciudad.
- Servicios, pero en vez de un array, un string con todos los servicios incluidos.

```js
db.listingsAndReviews.aggregate([
  {
    $match: {
      "address.country": "Spain" 
    }
  },
  {
    $project: {
      name: 1,
      price: 1,
      bedrooms: 1,
      beds: 1,
      bathrooms: 1,
      "host.host_location": 1,
      amenities: {
        $reduce: {
          input: "$amenities",
          initialValue: "",
          in: {
              $cond: {
                  if: { $eq: ["$$value", ""] },
                  then: "$$this",
                  else: { $concat: ["$$value", ", ", "$$this"] }
              }
          }
        }
      }
    }
  },
  {
    $sort: { price: -1 }
  },
  {
    $limit: 5
  }
])
```
