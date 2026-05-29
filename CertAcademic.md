# CertiLedger: Sistema de Gestión de Certificados Académicos en Hyperledger Fabric

Este proyecto implementa una red permisionada basada en **Hyperledger Fabric** y **Fabric CA** para emitir, consultar y revocar títulos académicos de forma segura. El sistema aprovecha la **Client Identity Library (CID)** para aplicar un control de acceso basado en roles (RBAC) directamente en el Smart Contract (Chaincode), garantizando que solo las entidades autorizadas puedan alterar el estado de los certificados.

---

## 1. Arquitectura de la Red y Actores

Para este proyecto de práctica, replicamos una infraestructura multi-organización estándar que simula un entorno real de consorcio:

* **Org1MSP (Universidad):** Actúa como la institución educativa. Es la única organización con permisos para emitir nuevos títulos y revocar certificados existentes.
* **Org2MSP (Empresa Reclutadora / Verificador):** Tiene acceso garantizado de lectura para consultar y validar la autenticidad de cualquier certificado en la red, pero el chaincode bloqueará cualquier intento de escritura por su parte.
* **Orderer (Servicio de Ordenación):** Nodo Raft independiente que ordena las transacciones y genera los bloques en el canal común `mychannel`.

### Estructura y Flujo de Transacciones

```mermaid
graph TD
    subgraph "Servicio de Ordenación"
        Orderer[orderer.example.com<br/>Puerto: 7050]
    end

    subgraph "Org1MSP: Institución Educativa"
        admin1[Admin@org1.example.com<br/>Rol: Emisión y Revocación]
        peer1[peer0.org1.example.com<br/>Puerto: 7051<br/>CouchDB: Estado de Certificados]
        
        admin1 -->|Invoca Escritura| peer1
    end

    subgraph "Org2MSP: Empresa Verificadora"
        admin2[Admin@org2.example.com<br/>Rol: Solo Consulta]
        peer2[peer0.org2.example.com<br/>Puerto: 9051<br/>CouchDB: Estado de Certificados]
        
        admin2 -->|Invoca Lectura| peer2
    end

    peer1 <-->|Canal: mychannel| peer2
    peer1 -->|Envía Propuestas| Orderer
    peer2 -->|Envía Propuestas| Orderer

    classDef uni fill:#1e40af,color:#fff,stroke:#1d4ed8,stroke-width:2px;
    classDef emp fill:#065f46,color:#fff,stroke:#047857,stroke-width:2px;
    classDef ord fill:#92400e,color:#fff,stroke:#b45309,stroke-width:2px;

    class peer1,admin1 uni;
    class peer2,admin2 emp;
    class Orderer ord;