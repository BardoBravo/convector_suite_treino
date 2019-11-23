# Convector Suite

# Playground
Link: https://stackblitz.com/edit/convector

# Instalando a base
```
npm i -g @worldsibu/convector-cli
npm i -g @worldsibu/hurley
```

# Criando uma rede local
```
# Crie projeto-base
conv new attributesDb -c person

# Instale as dependências
cd attributesDb
npm install
```

Dentro da página do projeto, execute:
```
conv generate chaincode participant
```

Para critérios de análise, é possível verificar o código o abrindo no **VSCode**.

Adicione os participantes:
```
npx lerna add participant-cc --scope person-cc  --include-filtered-dependencies
npx lerna bootstrap
```

No projeto, vamos substituir os seguintes arquivos com o conteúdo a seguir, da pasta **participant-cc/src**:

**index.ts**
```
export * from './participant.model';
export * from './participant.controller';
```

**participant.controllers.ts**
``` javascript
import * as yup from 'yup';

import {
  Controller,
  ConvectorController,
  Invokable,
  Param,
  BaseStorage
} from '@worldsibu/convector-core';

import { Participant } from './participant.model';
import { ClientIdentity } from 'fabric-shim';

@Controller('participant')
export class ParticipantController extends ConvectorController {
  get fullIdentity(): ClientIdentity {
    const stub = (BaseStorage.current as any).stubHelper;
    return new ClientIdentity(stub.getStub());
  };

  @Invokable()
  public async register(
    @Param(yup.string())
    id: string,
    @Param(yup.string())
    name: string,
  ) {
    // Retrieve to see if exists
    const existing = await Participant.getOne(id);

    if (!existing || !existing.id) {
      let participant = new Participant();
      participant.id = id;
      participant.name = name || id;
      participant.msp = this.fullIdentity.getMSPID();
      // Create a new identity
      participant.identities = [{
        fingerprint: this.sender,
        status: true
      }];
      await participant.save();
    } else {
      throw new Error('Identity exists already, please call changeIdentity fn for updates');
    }
  }
  @Invokable()
  public async changeIdentity(
    @Param(yup.string())
    id: string,
    @Param(yup.string())
    newIdentity: string
  ) {
    // Check permissions
    let isAdmin = this.fullIdentity.getAttributeValue('admin');
    let requesterMSP = this.fullIdentity.getMSPID();

    // Retrieve to see if exists
    const existing = await Participant.getOne(id);
    if (!existing || !existing.id) {
      throw new Error('No identity exists with that ID');
    }

    if (existing.msp != requesterMSP) {
      throw new Error('Unathorized. MSPs do not match');
    }

    if (!isAdmin) {
      throw new Error('Unathorized. Requester identity is not an admin');
    }

    // Disable previous identities!
    existing.identities = existing.identities.map(identity => {
      identity.status = false;
      return identity;
    });

    // Set the enrolling identity 
    existing.identities.push({
      fingerprint: newIdentity,
      status: true
    });
    await existing.save();
  }
  @Invokable()
  public async get(
    @Param(yup.string())
    id: string
  ) {
    const existing = await Participant.getOne(id);
    if (!existing || !existing.id) {
      throw new Error(`No identity exists with that ID ${id}`);
    }
    return existing;
  }
}
```

**partipant.model.ts**
``` javascript
import * as yup from 'yup';
import {
  ConvectorModel,
  ReadOnly,
  Required,
  Validate,
  FlatConvectorModel
} from '@worldsibu/convector-core';

export class x509Identities extends ConvectorModel<x509Identities>{
  @ReadOnly()
  public readonly type = 'io.worldsibu.examples.x509identity';

  @Validate(yup.boolean())
  @Required()
  status: boolean;
  @Validate(yup.string())
  @Required()
  fingerprint: string;
}

export class Participant extends ConvectorModel<Participant> {
  @ReadOnly()
  public readonly type = 'io.worldsibu.examples.participant';

  @ReadOnly()
  @Required()
  @Validate(yup.string())
  public name: string;

  @ReadOnly()
  @Validate(yup.string())
  public msp: string;

  @Validate(yup.array(x509Identities.schema()))
  public identities: Array<FlatConvectorModel<x509Identities>>;
}
```

## Codificando os modelos
Nós vamos precisar de um array **Atributos** dentro do modelo **Person**. Os atributos serão preenchidos pelos **Network Participant**.

**person.model.ts**

``` javascript
import * as yup from 'yup';
import {
  ConvectorModel,
  Default,
  ReadOnly,
  Required,
  Validate
} from '@worldsibu/convector-core-model';

export class Attribute extends ConvectorModel<Attribute>{
  @ReadOnly()
  @Required()
  public readonly type = 'io.worldsibu.attribute';

  @Required()
  public content: any;

  @Required()
  @ReadOnly()
  @Validate(yup.number())
  public issuedDate: number;

  public expiresDate: Date;

  @Default(false)
  @Validate(yup.boolean())
  public expired: boolean;

  @Required()
  @Validate(yup.string())
  public certifierID: string;
}

export class Person extends ConvectorModel<Person> {
  @ReadOnly()
  @Required()
  public readonly type = 'io.worldsibu.person';

  @Required()
  @Validate(yup.string())
  public name: string;

  @Validate(yup.array(Attribute.schema()))
  public attributes: Array<Attribute>;
}
```

## Codando os controllers
O controller é onde a lógica do smart contract realmente ocorre, operando a criação de pessoas e adição de atributos.

**person.controllers.ts**
```javascript
import * as yup from 'yup';
import { ChaincodeTx } from '@worldsibu/convector-platform-fabric';
import {
  Controller,
  ConvectorController,
  Invokable,
  Param
} from '@worldsibu/convector-core';
import { Participant } from 'participant-cc';

import { Person, Attribute } from './person.model';

@Controller('person')
export class PersonController extends ConvectorController<ChaincodeTx> {
  @Invokable()
  public async create(
    @Param(Person)
    person: Person
  ) {
    let exists = await Person.getOne(person.id);
    if (!!exists && exists.id) {
      throw new Error('There is a person registered with that Id already');
    }
    let gov = await Participant.getOne('gov');

    if (!gov || !gov.identities) {
      throw new Error('No government identity has been registered yet');
    }
    const govActiveIdentity = gov.identities.filter(identity => identity.status === true)[0];

    if (this.sender !== govActiveIdentity.fingerprint) {
      throw new Error(`Just the government - ID=gov - can create people - requesting organization was ${this.sender}`);
    }

    await person.save();
  }
  @Invokable()
  public async addAttribute(
    @Param(yup.string())
    personId: string,
    @Param(Attribute.schema())
    attribute: Attribute
  ) {
    // Check if the "stated" participant as certifier of the attribute is actually the one making the request
    let participant = await Participant.getOne(attribute.certifierID);

    if (!participant || !participant.identities) {
      throw new Error(`No participant found with id ${attribute.certifierID}`);
    }

    const participantActiveIdentity = participant.identities.filter(
      identity => identity.status === true)[0];

    if (this.sender !== participantActiveIdentity.fingerprint) {
      throw new Error(`Requester identity cannot sign with the current certificate ${this.sender}. This means that the user requesting the tx and the user set in the param certifierId do not match`);
    }

    let person = await Person.getOne(personId);

    if (!person || !person.id) {
      throw new Error(`No person found with id ${personId}`);
    }

    if (!person.attributes) {
      person.attributes = [];
    }

    let exists = person.attributes.find(attr => attr.id === attribute.id);

    if (!!exists) {
      let attributeOwner = await Participant.getOne(exists.certifierID);
      let attributeOwnerActiveIdentity = attributeOwner.identities.filter(
        identity => identity.status === true)[0];

      // Already has one, let's see if the requester has permissions to update it
      if (this.sender !== attributeOwnerActiveIdentity.fingerprint) {
        throw new Error(`User already has an attribute for ${attribute.id} but current identity cannot update it`);
      }
      // update as is the right attribute certifier
      exists = attribute;
    } else {
      person.attributes.push(attribute);
    }

    await person.save();
  }
}
```
## Testando com Unit Tests
Antes de executarmos contra nossa blockchain, vamos verificar a lógica do processo usando testes unitários, o que também nos ajuda a entender o proceso.

Primeiro, vamos instalar o package **chai-as-promised**:
```
npx lerna add chai-as-promised -D --scope person-cc  --include-filtered-dependencies

npx lerna add @types/chai-as-promised -D --scope person-cc  --include-filtered-dependencies
```

Agora,vamos alterar o código de testes da classe **person.spec.ts**:
``` javascript
// tslint:disable:no-unused-expression
import { join } from 'path';
import { MockControllerAdapter } from '@worldsibu/convector-adapter-mock';
import { ClientFactory, ConvectorControllerClient } from '@worldsibu/convector-core';

import * as chai from 'chai';
import { expect } from 'chai';
import 'mocha';
import * as chaiAsPromised from 'chai-as-promised';
import { ParticipantController, Participant } from 'participant-cc';

import { Person, PersonController, Attribute } from '../src';

describe('Person', () => {
  //chai.use(chaiAsPromised);
  // A fake certificate to emulate multiple wallets
  const fakeSecondParticipantCert = '-----BEGIN CERTIFICATE-----' +
    'MIICjzCCAjWgAwIBAgIUITsRsw5SIJ+33SKwM4j1Dl4cDXQwCgYIKoZIzj0EAwIw' +
    'czELMAkGA1UEBhMCVVMxEzARBgNVBAgTCkNhbGlmb3JuaWExFjAUBgNVBAcTDVNh' +
    'biBGcmFuY2lzY28xGTAXBgNVBAoTEG9yZzEuZXhhbXBsZS5jb20xHDAaBgNVBAMT' +
    'E2NhLm9yZzEuZXhhbXBsZS5jb20wHhcNMTgwODEzMDEyOTAwWhcNMTkwODEzMDEz' +
    'NDAwWjBCMTAwDQYDVQQLEwZjbGllbnQwCwYDVQQLEwRvcmcxMBIGA1UECxMLZGVw' +
    'YXJ0bWVudDExDjAMBgNVBAMTBXVzZXIzMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcD' +
    'QgAEcrfc0HHq5LG1UbyPSRLNjIQKqYoNY7/zPFC3UTJi3TTaIEqgVL6DF/8JIKuj' +
    'IT/lwkuemafacXj8pdPw3Zyqs6OB1zCB1DAOBgNVHQ8BAf8EBAMCB4AwDAYDVR0T' +
    'AQH/BAIwADAdBgNVHQ4EFgQUHFUlW/XJC7VcJe5pLFkz+xlMNpowKwYDVR0jBCQw' +
    'IoAgQ3hSDt2ktmSXZrQ6AY0EK2UHhXMx8Yq6O7XiA+X6vS4waAYIKgMEBQYHCAEE' +
    'XHsiYXR0cnMiOnsiaGYuQWZmaWxpYXRpb24iOiJvcmcxLmRlcGFydG1lbnQxIiwi' +
    'aGYuRW5yb2xsbWVudElEIjoidXNlcjMiLCJoZi5UeXBlIjoiY2xpZW50In19MAoG' +
    'CCqGSM49BAMCA0gAMEUCIQCNsmDjOXF/NvciSZebfk2hfSr/v5CqRD7pIHCq3lIR' +
    'lwIgPC/qGM1yeVinfN0z7M68l8rWn4M4CVR2DtKMpk3G9k9=' +
    '-----END CERTIFICATE-----';
  // By default, MockControllerAdapter will use this fingerprint as the `this.sender`
  const mockIdentity = 'B6:0B:37:7C:DF:D2:7A:08:0B:98:BF:52:A4:2C:DC:4E:CC:70:91:E1';

  let adapter: MockControllerAdapter;
  let personCtrl: ConvectorControllerClient<PersonController>;
  let participantCtrl: ConvectorControllerClient<ParticipantController>;
  let personId = '1-1002-1002'
  before(async () => {
    // Mocks the blockchain execution environment
    adapter = new MockControllerAdapter();
    await adapter.init([
      {
        version: '*',
        controller: 'PersonController',
        name: join(__dirname, '..')
      },
      {
        version: '*',
        controller: 'ParticipantController',
        name: join(__dirname, '../../participant-cc')
      }
    ]);
    adapter.stub['fingerprint'] = mockIdentity;
    personCtrl = ClientFactory(PersonController, adapter);
    participantCtrl = ClientFactory(ParticipantController, adapter);
  });

  it('should try to create a person but no government identity has been registered', async () => {
    const personSample = new Person({
      id: personId,
      name: 'Walter Montes'
    });

    await expect(personCtrl.create(personSample)).to.be.empty;
  });

  it('should create the government identity', async () => {
    await participantCtrl.register('gov', 'Big Gov');

    const justSavedModel = await adapter.getById<Participant>('gov');

    expect(justSavedModel.id).to.exist;
  });

  it('should create a person', async () => {
    const personSample = new Person({
      id: personId,
      name: 'Walter Montes'
    });

    await personCtrl.create(personSample);

    const justSavedModel = await adapter.getById<Person>(personSample.id);

    expect(justSavedModel.name).to.exist;
  });

  it('should add a birth-year attribute through the gov identity', async () => {
    let attributeId = 'birth-year';
    let attribute = new Attribute(attributeId);
    attribute.certifierID = 'gov';
    attribute.content = '1993';
    attribute.issuedDate = Date.now();

    await personCtrl.addAttribute(personId, attribute);

    const justSavedModel = await adapter.getById<Person>(personId);
    expect(justSavedModel.id).to.exist;

    const resultingAttr = justSavedModel.attributes.find(attr => attr.id === attributeId);

    expect(resultingAttr).to.exist;
    expect(resultingAttr.id).to.eq(attribute.id);
  });

  it('should create a participant for the MIT', async () => {
    // Fake another certificate for tests
    (adapter.stub as any).usercert = fakeSecondParticipantCert;

    await participantCtrl.register('mit', 'MIT - Massachusetts Institute of Technology');

    const justSavedModel = await adapter.getById<Participant>('mit');

    expect(justSavedModel.id).to.exist;
  });

  it('should try to create a person but the MIT cannot', async () => {
    const personSample = new Person({
      id: personId + '1111',
      name: 'Walter Montes'
    });

    await expect(personCtrl.create(personSample)).to.be.empty;
  });

  it('should add a mit-degree attribute through the MIT identity', async () => {
    // Fake another certificate for tests
    (adapter.stub as any).usercert = fakeSecondParticipantCert;

    let attributeId = 'mit-degree';
    let attribute = new Attribute(attributeId);
    attribute.certifierID = 'mit';
    attribute.content = {
      level: 'dummy',
      honours: 'high',
      description: 'Important title!'
    };
    attribute.issuedDate = Date.now();

    await personCtrl.addAttribute(personId, attribute);

    const person = await adapter.getById<Person>(personId);
    expect(person.id).to.exist;

    const resultingAttr = person.attributes.find(attr => attr.id === attributeId);
    expect(resultingAttr).to.exist;
    expect(resultingAttr.id).to.eq(attribute.id);
  });

  it('should try to make the birth-year expire! with the MIT identity', async () => {
    // Fake another certificate for tests
    (adapter.stub as any).usercert = fakeSecondParticipantCert;
    const person = await adapter.getById<Person>(personId);
    let attribute = person.attributes.find(attr => attr.id === 'birth-year');

    attribute.certifierID = 'mit';
    attribute.expired = true;
    await expect(personCtrl.addAttribute(personId, attribute)).to.be.empty;
  });
});
```

Agora, basta executar os testes:
```
npm test
```

## Configurando o Smart Contract
Nesse momento, faremos a excução de nossa aplicação contra a blockchain. Porém, antes é necessário fazer alguns ajustes no arquivo **config.json**:
``` javascript
//..
 "controllers": [
    {
      "name": "participant-cc",
      "version": "file:./packages/participant-cc",
      "controller": "ParticipantController"
    },
    {
      "name": "person-cc",
      "version": "file:./packages/person-cc",
      "controller": "PersonController"
    }
  ],
//..
```

### Instale a network de desenvolvimeno blockchain
```
npm run env:restart
npm run cc:start -- person
```

### Testando ações do smart contract
Agora que o smart contract está rodando, podemos fazer algumas chamada diretas ao channel da blockchain:
```
# Crie um participante do governo com o usuário Admin
hurl invoke person participant_register gov "Big Government" -u admin
```

```
# Adicione a PUC como partipante válida
hurl invoke person participant_register mit "MIT" -u user1
```

```
# Crie um pessoa
hurl invoke person person_create "{\"id\":\"1-100-100\", \"name\": \"Fausto Silva\"}" -u admin
```

```
# Adicione um atributo de data de nascimento
hurl invoke person person_addAttribute "1-100-100" "{\"id\": \"birth-year\", \"certifierID\": \"gov\", \"content\": \"1993\", \"issuedDate\": 1554239270 }" -u admin

```

É sempre possível acompnhar o **World State** através dp link **http://localhost:5084/_utils/#database/ch1_person**.

