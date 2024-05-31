[![FIWARE Banner](https://fiware.github.io/tutorials.Short-Term-History/img/fiware.png)](https://www.fiware.org/developers)
[![NGSI v2](https://img.shields.io/badge/NGSI-v2-5dc0cf.svg)](https://fiware-ges.github.io/orion/api/v2/stable/)

[![FIWARE Core Context Management](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/core.svg)](https://github.com/FIWARE/catalogue/blob/master/core/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Short-Term-History.svg)](https://opensource.org/licenses/MIT)
[![NGSI v1](https://img.shields.io/badge/NGSI-v1-ff69b4.svg)](http://forge.fiware.org/docman/view.php/7/3213/FI-WARE_NGSI_RESTful_binding_v1.0.zip)
[![Support badge](https://img.shields.io/badge/tag-fiware-orange.svg?logo=stackoverflow)](https://stackoverflow.com/questions/tagged/fiware)
<br/> [![Documentation](https://img.shields.io/readthedocs/fiware-tutorials.svg)](https://fiware-tutorials.rtfd.io)

The **NGSI-v2**  tutorial is an introduction to [FIWARE STH-Comet](https://fiware-sth-comet.readthedocs.io/) - a generic enabler
which is used to retrieve trend data from a MongoDB database. The tutorial activates the IoT sensors connected in the
[previous tutorial](https://github.com/FIWARE/tutorials.IoT-Agent) and persists measurements from those sensors into a
database and retrieves time-based aggregations of that data.

The **NGSI-LD** tutorial is an introduction to the temporal interface of **NGSI-LD**, an **optional** add-on to context broker
implementations. The tutorial activates the IoT animal collars connected in the
[previous tutorial](https://github.com/FIWARE/tutorials.IoT-Agent/tree/NGSI-LD) and persists measurements from those
sensors into a database and retrieves time-based aggregations of that data..

The tutorials use [cUrl](https://ec.haxx.se/) commands throughout, but are also available as
[Postman documentation](https://www.postman.com/downloads/)

# Start-Up

## NGSI-v2 Smart Supermarket

**NGSI-v2** offers JSON based interoperability used in individual Smart Systems. To run this tutorial with **NGSI-v2**, use the `NGSI-v2` branch.

```console
git clone https://github.com/FIWARE/tutorials.Short-Term-History.git
cd tutorials.Short-Term-History
git checkout NGSI-v2

./services create
./services start
```

| [![NGSI v2](https://img.shields.io/badge/NGSI-v2-5dc0cf.svg)](https://fiware-ges.github.io/orion/api/v2/stable/) | :books: [Documentation](https://github.com/FIWARE/tutorials.Short-Term-History/tree/NGSI-v2) | <img src="https://cdn.jsdelivr.net/npm/simple-icons@v3/icons/postman.svg" height="15" width="15"> [Postman Collection](https://fiware.github.io/tutorials.Short-Term-History/) |
| --- | --- | --- |

## NGSI-LD Smart Farm

**NGSI-LD** offers JSON-LD based interoperability used for Federations and Data Spaces. To run this tutorial with **NGSI-LD**, use the `NGSI-LD` branch.

```console
git clone https://github.com/FIWARE/tutorials.Short-Term-History.git
cd tutorials.Short-Term-History
git checkout NGSI-LD

./services create
./services start
```

| [![NGSI LD](https://img.shields.io/badge/NGSI-LD-d6604d.svg)](https://www.etsi.org/deliver/etsi_gs/CIM/001_099/009/01.08.01_60/gs_cim009v010801p.pdf) | :books: [Documentation](https://github.com/FIWARE/tutorials.Short-Term-History/tree/NGSI-LD) | <img  src="https://cdn.jsdelivr.net/npm/simple-icons@v3/icons/postman.svg" height="15" width="15"> [Postman Collection](https://fiware.github.io/tutorials.Short-Term-History/ngsi-ld.html) |
| --- | --- | --- |

---

## License

[MIT](LICENSE) © 2018-2024 FIWARE Foundation e.V.
