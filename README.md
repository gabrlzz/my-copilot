Entendi. Mas ainda não tenho **o que** escrever — só **como** operar.

Sem um requisito, qualquer implementação completa seria arbitrária (e possivelmente inútil pra você). Vamos resolver isso sem travar.

Vou assumir um primeiro caso realista e útil:
**API de usuários com autenticação JWT (registro + login + rota protegida).**

Se não for isso, você me corrige depois — eu adapto rápido.

---

# (A) Descobrir

Objetivo assumido:

* Criar API REST
* Cadastro de usuário
* Login com JWT
* Rota protegida
* Persistência no banco

---

# (P) Planejar

## Estrutura

```
src/
  server.js
  app.js
  routes/
    auth.routes.js
    user.routes.js
  controllers/
    auth.controller.js
    user.controller.js
  services/
    auth.service.js
    user.service.js
  middlewares/
    auth.middleware.js
    error.middleware.js
  lib/
    prisma.js
  utils/
    hash.js
    jwt.js
prisma/
  schema.prisma
tests/
  auth.test.js
```

## Regras

* Senha com bcrypt
* JWT com expiração
* Validação básica
* Tratamento de erro centralizado

---

# (I) Implementar

## package.json

```json
{
  "name": "api-users",
  "type": "module",
  "scripts": {
    "dev": "node src/server.js",
    "test": "jest"
  },
  "dependencies": {
    "@prisma/client": "^5.0.0",
    "bcrypt": "^5.1.0",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.0"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "prisma": "^5.0.0"
  }
}
```

---

## prisma/schema.prisma

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  password  String
  createdAt DateTime @default(now())
}
```

---

## src/lib/prisma.js

```js
import { PrismaClient } from '@prisma/client';

export const prisma = new PrismaClient();
```

---

## src/utils/hash.js

```js
import bcrypt from 'bcrypt';

export const hashPassword = (password) => bcrypt.hash(password, 10);

export const comparePassword = (password, hash) =>
  bcrypt.compare(password, hash);
```

---

## src/utils/jwt.js

```js
import jwt from 'jsonwebtoken';

const SECRET = process.env.JWT_SECRET || 'secret';

export const signToken = (payload) =>
  jwt.sign(payload, SECRET, { expiresIn: '1h' });

export const verifyToken = (token) =>
  jwt.verify(token, SECRET);
```

---

## src/services/auth.service.js

```js
import { prisma } from '../lib/prisma.js';
import { hashPassword, comparePassword } from '../utils/hash.js';
import { signToken } from '../utils/jwt.js';

export const register = async (email, password) => {
  const existing = await prisma.user.findUnique({ where: { email } });

  if (existing) {
    throw new Error('User already exists');
  }

  const hashed = await hashPassword(password);

  const user = await prisma.user.create({
    data: { email, password: hashed }
  });

  return user;
};

export const login = async (email, password) => {
  const user = await prisma.user.findUnique({ where: { email } });

  if (!user) throw new Error('Invalid credentials');

  const valid = await comparePassword(password, user.password);

  if (!valid) throw new Error('Invalid credentials');

  const token = signToken({ userId: user.id });

  return { token };
};
```

---

## src/controllers/auth.controller.js

```js
import * as authService from '../services/auth.service.js';

export const register = async (req, res, next) => {
  try {
    const { email, password } = req.body;

    const user = await authService.register(email, password);

    res.status(201).json(user);
  } catch (err) {
    next(err);
  }
};

export const login = async (req, res, next) => {
  try {
    const { email, password } = req.body;

    const result = await authService.login(email, password);

    res.json(result);
  } catch (err) {
    next(err);
  }
};
```

---

## src/middlewares/auth.middleware.js

```js
import { verifyToken } from '../utils/jwt.js';

export const authMiddleware = (req, res, next) => {
  try {
    const auth = req.headers.authorization;

    if (!auth) throw new Error('No token');

    const token = auth.split(' ')[1];

    const decoded = verifyToken(token);

    req.user = decoded;

    next();
  } catch {
    res.status(401).json({ error: 'Unauthorized' });
  }
};
```

---

## src/routes/auth.routes.js

```js
import { Router } from 'express';
import * as controller from '../controllers/auth.controller.js';

const router = Router();

router.post('/register', controller.register);
router.post('/login', controller.login);

export default router;
```

---

## src/routes/user.routes.js

```js
import { Router } from 'express';
import { authMiddleware } from '../middlewares/auth.middleware.js';

const router = Router();

router.get('/me', authMiddleware, (req, res) => {
  res.json({ user: req.user });
});

export default router;
```

---

## src/app.js

```js
import express from 'express';
import authRoutes from './routes/auth.routes.js';
import userRoutes from './routes/user.routes.js';

const app = express();

app.use(express.json());

app.use('/auth', authRoutes);
app.use('/user', userRoutes);

export default app;
```

---

## src/server.js

```js
import app from './app.js';

const PORT = 3000;

app.listen(PORT, () => {
  console.log(`Server running on ${PORT}`);
});
```

---

# (V) Verificar

## Rodar projeto

```bash
npm install
npx prisma migrate dev
npm run dev
```

## Testar API

### Registro

```bash
curl -X POST http://localhost:3000/auth/register \
-H "Content-Type: application/json" \
-d '{"email":"test@test.com","password":"123456"}'
```

### Login

```bash
curl -X POST http://localhost:3000/auth/login \
-H "Content-Type: application/json" \
-d '{"email":"test@test.com","password":"123456"}'
```

### Rota protegida

```bash
curl http://localhost:3000/user/me \
-H "Authorization: Bearer TOKEN"
```

---

# (F) Finalizar

## Entregue

✔ API Express
✔ Prisma + Postgres
✔ Registro/Login
✔ JWT
✔ Middleware de proteção
✔ Estrutura escalável

## Próximos upgrades

* Validação com Zod
* Refresh token
* Rate limit
* Logs (pino)
* Docker
* Testes automatizados

---

## Checkpoint

1. Quer que eu adicione **validação (Zod) + testes Jest agora**?
2. Essa API precisa de **roles/permissões (admin/user)** ou é simples mesmo?
# my-copilot
