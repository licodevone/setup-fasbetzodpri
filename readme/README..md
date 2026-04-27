# API Fastify-BetterAuth-Zod-Prisma template

Abaixo a demonstração dos `passos` necessário para rodar o programa

## **1º Passo :**

- Inserir o arquivo `package.json` com as bibliotecas e instalar

## **2º Passo :**

```ah
$ pnpm i
```

## **3º Passo :**

- Criar o banco de dados

```ah
$ Docker compose up -d
```

## **4º Passo :**

- Transformar o arquivo .env.exmplo em .env

```ah
$  cp .env.example .env
```

## **5º Passo :**

- Gerar o client

```ah
$ npx prisma generate
```

## **6º Passo :**

-Formatar o arquivo de schema

```ah
$ npx prisma format
```

## **7º Passo :**

- Criar uma migração do Sql para implantação do banco de dados

```ah
$ npx prisma migrate dev --name init
```

## **8º Passo :**

- Verificar graficamente as tabelas

```ah
$ npx prisma studio
```

![alt text](image-11.png)

![alt text](image-12.png)

## **9º Passo :**

- Gerar o Better_Auth_secret

> https://better-auth.com/docs/installation#configure-database

## **10º Passo :**

- integração do BetterAuth com o Fastify

![alt text](image-3.png)

> https://better-auth.com/docs/integrations/fastify

```js
import "dotenv/config";

import fastifyCors from "@fastify/cors";
import fastifySwagger from "@fastify/swagger";
import fastifyApiReference from "@scalar/fastify-api-reference";
import Fastify from "fastify";
import {
  jsonSchemaTransform,
  serializerCompiler,
  validatorCompiler,
  ZodTypeProvider,
} from "fastify-type-provider-zod";
import { z } from "zod";

import { auth } from "./lib/auth.js";

const app = Fastify({
  logger: true,
});

app.setValidatorCompiler(validatorCompiler);
app.setSerializerCompiler(serializerCompiler);

await app.register(fastifySwagger, {
  openapi: {
    info: {
      title: "SampleApi",
      description: "Sample backend service",
      version: "1.0.0",
    },
    servers: [],
  },
  transform: jsonSchemaTransform,
});

app.register(fastifyCors, {
  origin: ["http://localhost:3000"],
  credentials: true,
});

await app.register(fastifyApiReference, {
  routePrefix: "/docs",
  configuration: {
    sources: [
      {
        title: "Template de API com Fastify, Zod, Better-Auth e Prisma",
        slug: "template-api",
        url: "/swagger.json",
      },
      {
        title: "Auth API",
        slug: "auth-api",
        url: "/api/auth/open-api/generate-schema",
      },
    ],
  },
});

app.withTypeProvider<ZodTypeProvider>().route({
  method: "GET",
  url: "/swagger.json",
  handler() {
    return app.swagger();
  },
});

app.withTypeProvider<ZodTypeProvider>().route({
  method: "GET",
  url: "/",
  // Define your schema
  schema: {
    description: "Hello world",
    tags: ["Hello world"],
    response: {
      200: z.object({
        message: z.string(),
      }),
    },
  },
  handler: () => {
    return {
      message: "Hello world",
    };
  },
});

app.route({
  method: ["GET", "POST"],
  url: "/api/auth/*",
  async handler(request, reply) {
    try {
      // Construct request URL
      const url = new URL(request.url, `http://${request.headers.host}`);

      // Convert Fastify headers to standard Headers object
      const headers = new Headers();
      // Create Fetch API-compatible request
      const req = new Request(url.toString(), {
        method: request.method,
        headers,
        ...(request.body ? { body: JSON.stringify(request.body) } : {}),
      });
      // Process authentication request
      const response = await auth.handler(req);
      // Forward response to client
      reply.status(response.status);
      response.headers.forEach((value, key) => reply.header(key, value));
      return reply.send(response.body ? await response.text() : null);
    } catch (error) {
      app.log.error(error);
      return reply.status(500).send({
        error: "Internal authentication error",
        code: "AUTH_FAILURE",
      });
    }
  },
});

try {
  await app.listen({ port: Number(process.env.PORT) || 8081 });
} catch (err) {
  app.log.error(err);
  process.exit(1);
}

```

## **11º Passo :**

- Instalação da extensão do Prisma para melhorar a visualização

![alt text](image-1.png)

![alt text](image-2.png)

## **12º Passo :**

- Criação do arquivo auth.ts

`src/lib/auth.ts`

```js
/// <reference types="node" />
import { PrismaPg } from "@prisma/adapter-pg";
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { openAPI } from "better-auth/plugins";

import { PrismaClient } from "../generated/prisma/client.js";

const prisma = new PrismaClient({
  adapter: new PrismaPg({ connectionString: process.env.DATABASE_URL! }),
});

export const auth = betterAuth({
  trustedOrigins: ["http://localhost:3000"],
  emailAndPassword: {
    enabled: true,
  },
  database: prismaAdapter(prisma, {
    provider: "postgresql",
  }),
  plugins: [openAPI()],
});
```

## **13º Passo :**

- Criar o arquivo prisma.config.ts

`prisma.config.ts`

```js
/// <reference types="node" />
import "dotenv/config";

import { defineConfig } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: {
    path: "prisma/migrations",
  },
  datasource: {
    url: process.env["DATABASE_URL"],
  },
});
```

## **14º Passo :**

- Criar o arquivo `.env` com as configurações abaixo

> https://better-auth.com/docs/installation#configure-database

![alt text](image-4.png)

- O BETTER_AUTH_SECRET deve ser gerado na página do link acima

```js
DATABASE_URL="postgresql://postgres:password@localhost:5432/mydatabase?schema=public"

PORT=8080

BETTER_AUTH_SECRET=secretkey
BETTER_AUTH_URI=http://localhost:8080
```

## **15º Passo :**

- Próximo passo é gerar as tabelas no arquivo Schema.prisma

```sh
$ npx @better-auth/cli generate
```

## **16º Passo :**

- Gerar as tabelas de sessão, conta e verificação

![alt text](image-6.png)

```js
model Session {
  id        String   @id
  expiresAt DateTime
  token     String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  ipAddress String?
  userAgent String?
  userId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([token])
  @@index([userId])
  @@map("session")
}

model Account {
  id                    String    @id
  accountId             String
  providerId            String
  userId                String
  user                  User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  accessToken           String?
  refreshToken          String?
  idToken               String?
  accessTokenExpiresAt  DateTime?
  refreshTokenExpiresAt DateTime?
  scope                 String?
  password              String?
  createdAt             DateTime  @default(now())
  updatedAt             DateTime  @updatedAt

  @@index([userId])
  @@map("account")
}

model Verification {
  id         String   @id
  identifier String
  value      String
  expiresAt  DateTime
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  @@index([identifier])
  @@map("verification")
}
```

## **17º Passo :**

\_ Agora fazer a nova migração e gerar as tabelas no banco de dados

```sh
$ npx prisma migrate dev --name init
```

![alt text](image-5.png)

![alt text](image-9.png)

## **18º Passo :**

- Instalar a biblioteca `Scalar` - API de referência para o Fastify

```sh
$ pnpm add @scalar/fastify-api-reference@1.44.20
```

> https://www.npmjs.com/package/@scalar/fastify-api-reference?activeTab=dependencies

> https://scalar.com/products/api-references/integrations/fastify

![alt text](image-7.png)

![alt text](image-8.png)

## **19º Passo :**

- Retirar o Swagger-ui, as linhas 36 a 38, importar o Scalar e inserir a rota

![alt text](image-10.png)

```js
import ScalarApiReference from "@scalar/fastify-api-reference";
```

## **20º Passo :**

- Rodar a aplicação

```ah
$ pnpm dev
```

## Daqui prafrente agora serão os testes da Aplicação

## **1º Passo :**

- Cadastro de usuário

```json
{
  "name": "Luis Eduardo dos S. Pinheiro",
  "email": "licodevone@gmail.com",
  "password": "Lico123",
  "image": "",
  "callbackURL": "",
  "rememberMe": true
}
```

- Fazendo o Login do usuário cadastrado

```json
{
  "email": "licodevone@gmail.com",
  "password": "Lico123",
  "callbackURL": "",
  "image": "",
  "callbackURL": "",
  "rememberMe": true
}
```
