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
    subgraph Servicio de Ordenación
        Orderer[orderer.example.com<br/>Puerto: 7050]
    end

    subgraph Org1MSP: Institución Educativa
        admin1[Admin@org1.example.com<br/>Rol: Emisión y Revocación]
        peer1[peer0.org1.example.com<br/>Puerto: 7051<br/>CouchDB: Estado de Certificados]
        
        admin1 -->|Invoca Escritura| peer1
    end

    subgraph Org2MSP: Empresa Verificadora
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
```

---

## 2. Código del Smart Contract

Crea un directorio para almacenar los archivos del chaincode:

```bash
mkdir -p $HOME/red-con-ca/chaincode-certificados
cd $HOME/red-con-ca/chaincode-certificados
```

### 2.1. Lógica del Contrato (`CertificadoContract.js`)

Crea el archivo `CertificadoContract.js`. Aquí se implementa el control `ctx.clientIdentity.getMSPID()` para verificar criptográficamente quién firma la transacción.

```javascript
'use strict';

const { Contract } = require('fabric-contract-api');

class CertificadoContract extends Contract {

    async InitLedger(ctx) {
        console.info('============= Inicializando CertiLedger =============');
        const certificados = [
            {
                id: 'CERT-2026-001',
                estudiante: 'María Gómez',
                grado: 'Máster Avanzado en Tecnologías Blockchain',
                fechaEmision: '2026-05-29',
                estatus: 'Activo'
            }
        ];

        for (const cert of certificados) {
            await ctx.stub.putState(cert.id, Buffer.from(JSON.stringify(cert)));
            console.info(`Certificado inicializado: ${cert.id}`);
        }
    }

    // RESTRINGIDO: Solo invocable por miembros de Org1MSP (Universidad)
    async EmitirCertificado(ctx, id, estudiante, grado, fechaEmision) {
        const mspId = ctx.clientIdentity.getMSPID();
        if (mspId !== 'Org1MSP') {
            throw new Error(`ACCESO DENEGADO: Tu organización (${mspId}) no tiene permisos. Solo la Universidad (Org1MSP) puede emitir títulos.`);
        }

        const existe = await this._CertificadoExiste(ctx, id);
        if (existe) {
            throw new Error(`El certificado con ID ${id} ya está registrado.`);
        }

        const nuevoCertificado = {
            id,
            estudiante,
            grado,
            fechaEmision,
            estatus: 'Activo'
        };

        await ctx.stub.putState(id, Buffer.from(JSON.stringify(nuevoCertificado)));
        return JSON.stringify(nuevoCertificado);
    }

    // ACCESO PÚBLICO: Cualquier organización autorizada en el canal
    async ConsultarCertificado(ctx, id) {
        const certBytes = await ctx.stub.getState(id);
        if (!certBytes || certBytes.length === 0) {
            throw new Error(`El certificado con ID ${id} no existe.`);
        }
        return certBytes.toString();
    }

    // RESTRINGIDO: Solo invocable por miembros de Org1MSP (Universidad)
    async RevocarCertificado(ctx, id) {
        const mspId = ctx.clientIdentity.getMSPID();
        if (mspId !== 'Org1MSP') {
            throw new Error(`ACCESO DENEGADO: Solo la Universidad (Org1MSP) puede revocar un título.`);
        }

        const certBytes = await ctx.stub.getState(id);
        if (!certBytes || certBytes.length === 0) {
            throw new Error(`El certificado con ID ${id} no existe.`);
        }

        const certificado = JSON.parse(certBytes.toString());
        certificado.estatus = 'Revocado'; // Mutación de estado, preservando historial

        await ctx.stub.putState(id, Buffer.from(JSON.stringify(certificado)));
        return JSON.stringify(certificado);
    }

    async _CertificadoExiste(ctx, id) {
        const certBytes = await ctx.stub.getState(id);
        return certBytes && certBytes.length > 0;
    }
}

module.exports = CertificadoContract;
```

### 2.2. Dependencias Node.js (`package.json`)

Crea el archivo `package.json` en la misma ubicación:

```json
{
  "name": "certiledger-chaincode",
  "version": "1.0.0",
  "description": "Smart Contract para gestión de títulos académicos con control de identidad",
  "main": "CertificadoContract.js",
  "engines": {
    "node": ">=18"
  },
  "dependencies": {
    "fabric-contract-api": "^2.5.0",
    "fabric-shim": "^2.5.0"
  }
}
```

---

## 3. Despliegue en la Red (Lifecycle)

Asegúrate de estar en el directorio raíz de la red de CA (`$HOME/red-con-ca`) antes de ejecutar los comandos.

### 3.1. Empaquetar
```bash
cd $HOME/red-con-ca

peer lifecycle chaincode package certiledger.tar.gz \
  --path ./chaincode-certificados/ \
  --lang node \
  --label certiledger_1.0
```

### 3.2. Instalar en Peer Org1 (Universidad)
```bash
export ORDERER_CA=$PWD/organizations/ordererOrganizations/[example.com/orderers/orderer.example.com/tls/ca.crt](https://example.com/orderers/orderer.example.com/tls/ca.crt)
export PEER_ORG1_TLS=$PWD/organizations/peerOrganizations/[org1.example.com/peers/peer0.org1.example.com/tls/ca.crt](https://org1.example.com/peers/peer0.org1.example.com/tls/ca.crt)
export FABRIC_CFG_PATH=$HOME/fabric/fabric-samples/config

export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER_ORG1_TLS
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/[org1.example.com/users/Admin@org1.example.com/msp](https://org1.example.com/users/Admin@org1.example.com/msp)
export CORE_PEER_TLS_ENABLED=true

peer lifecycle chaincode install certiledger.tar.gz
```

### 3.3. Instalar en Peer Org2 (Empresa)
```bash
export PEER_ORG2_TLS=$PWD/organizations/peerOrganizations/[org2.example.com/peers/peer0.org2.example.com/tls/ca.crt](https://org2.example.com/peers/peer0.org2.example.com/tls/ca.crt)

export CORE_PEER_LOCALMSPID=Org2MSP
export CORE_PEER_ADDRESS=localhost:9051
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER_ORG2_TLS
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/[org2.example.com/users/Admin@org2.example.com/msp](https://org2.example.com/users/Admin@org2.example.com/msp)

peer lifecycle chaincode install certiledger.tar.gz
```

### 3.4. Aprobar en ambas Organizaciones

Primero, obtenemos el Package ID:
```bash
peer lifecycle chaincode queryinstalled
```
*Guarda el hash devuelto en una variable:*
```bash
export CC_PACKAGE_ID=certiledger_1.0:PEGAR_AQUI_EL_HASH_EXACTO
```

**Aprobar como Org2:**
```bash
peer lifecycle chaincode approveformyorg \
  -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com \
  --tls --cafile $ORDERER_CA \
  --channelID mychannel --name certiledger --version 1.0 \
  --package-id $CC_PACKAGE_ID --sequence 1
```

**Aprobar como Org1:**
```bash
export CORE_PEER_LOCALMSPID=Org1MSP
export CORE_PEER_ADDRESS=localhost:7051
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER_ORG1_TLS
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/[org1.example.com/users/Admin@org1.example.com/msp](https://org1.example.com/users/Admin@org1.example.com/msp)

peer lifecycle chaincode approveformyorg \
  -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com \
  --tls --cafile $ORDERER_CA \
  --channelID mychannel --name certiledger --version 1.0 \
  --package-id $CC_PACKAGE_ID --sequence 1
```

### 3.5. Commit del Chaincode
```bash
peer lifecycle chaincode commit \
  -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com \
  --tls --cafile $ORDERER_CA \
  --channelID mychannel --name certiledger --version 1.0 --sequence 1 \
  --peerAddresses localhost:7051 --tlsRootCertFiles $PEER_ORG1_TLS \
  --peerAddresses localhost:9051 --tlsRootCertFiles $PEER_ORG2_TLS
```

---

## 4. Pruebas Operativas de Seguridad (Demostración)

### 4.1. Inicializar el Ledger (Como Universidad / Org1)
```bash
peer chaincode invoke \
  -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com \
  --tls --cafile $ORDERER_CA \
  -C mychannel -n certiledger \
  --peerAddresses localhost:7051 --tlsRootCertFiles $PEER_ORG1_TLS \
  --peerAddresses localhost:9051 --tlsRootCertFiles $PEER_ORG2_TLS \
  -c '{"function":"InitLedger","Args":[]}'
```

### 4.2. Emitir Título (Transacción de Éxito)
```bash
peer chaincode invoke \
  -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com \
  --tls --cafile $ORDERER_CA \
  -C mychannel -n certiledger \
  --peerAddresses localhost:7051 --tlsRootCertFiles $PEER_ORG1_TLS \
  --peerAddresses localhost:9051 --tlsRootCertFiles $PEER_ORG2_TLS \
  -c '{"function":"EmitirCertificado","Args":["CERT-2026-002","Juan Pérez","Ingeniería de Software","2026-05-29"]}'
```

### 4.3. Consulta de Título (Como Empresa / Org2)
Cambiamos a la identidad de Org2:
```bash
export CORE_PEER_LOCALMSPID=Org2MSP
export CORE_PEER_ADDRESS=localhost:9051
export CORE_PEER_TLS_ROOTCERT_FILE=$PEER_ORG2_TLS
export CORE_PEER_MSPCONFIGPATH=$PWD/organizations/peerOrganizations/[org2.example.com/users/Admin@org2.example.com/msp](https://org2.example.com/users/Admin@org2.example.com/msp)

peer chaincode query -C mychannel -n certiledger -c '{"Args":["ConsultarCertificado","CERT-2026-002"]}'
```

### 4.4. Hackeo Simulado: Rechazo Criptográfico
Estando en la identidad de la **Empresa**, intentamos saltarnos la seguridad y emitir un título:
```bash
peer chaincode invoke \
  -o localhost:7050 --ordererTLSHostnameOverride orderer.example.com \
  --tls --cafile $ORDERER_CA \
  -C mychannel -n certiledger \
  --peerAddresses localhost:7051 --tlsRootCertFiles $PEER_ORG1_TLS \
  --peerAddresses localhost:9051 --tlsRootCertFiles $PEER_ORG2_TLS \
  -c '{"function":"EmitirCertificado","Args":["CERT-2026-HACK","Atacante","Título Falso","2026-05-29"]}'
```
> **Resultado esperado:** La blockchain bloquea la escritura en base a las políticas definidas en el Chaincode. Aparecerá el error programado: `ACCESO DENEGADO: Tu organización (Org2MSP) no tiene permisos...`