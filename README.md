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

Adicionae os participantes:
```
npx lerna add participant-cc --scope person-cc  --include-filtered-dependencies
npx lerna bootstrap
```