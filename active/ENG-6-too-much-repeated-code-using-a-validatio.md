---
ticketId: ENG-6
ticketTitle: "Too much repeated code : Using a validation library like Joi or Zod can indeed help improve code readability and provide a DRY code structure"
status: draft
createdAt: 2026-03-22T16:25:24.219Z
---

> Selected repository: `Zivl9090/node-express-realworld-example-app`
> Reasoning: This Node.js/Express backend repository contains the validation code shown in the ticket and would benefit from implementing Joi/Zod to reduce repeated validation logic.

I'll thoroughly investigate the codebase before forming recommendations.

## Investigation

Let me start by reading the project structure and understanding the current validation patterns.
<tool_call>
{"name": "read_file", "arguments": {"path": "/Users/zivlevi/Code/playground/repos/node-express-realworld-example-app/CLAUDE.md"}}
</tool_call>
<tool_response>
# CLAUDE.md — conduit-backend

This is the backend API for the Conduit demo app (a Medium clone). Node.js + Express + Mongoose + JWT auth.

## What this repo is

A REST API that powers the Conduit frontend. It handles user registration/login, article CRUD, comments, favourites, and user following. The API spec is defined at https://realworld-docs.netlify.app/docs/specs/backend-specs/introduction.

## Key directories

```
src/
  app.js              # Express app setup, middleware registration
  config/             # Environment config
  models/             # Mongoose schemas: User, Article, Comment
  routes/             # Express routers
    api/
      users.js        # POST /users, POST /users/login, GET/PUT /user
      profiles.js     # GET /profiles/:username, follow/unfollow
      articles.js     # CRUD /articles, feed, favourites
      comments.js     # GET/POST/DELETE /articles/:slug/comments
      tags.js         # GET /tags
```

## How to run

```bash
npm install
# requires a running MongoDB instance
MONGODB_URI=mongodb://localhost/conduit npm start
# server starts on port 3000
```

## How to run tests

```bash
npm test
```

Tests use Mocha + Chai. They hit a real MongoDB test database — do not mock the DB layer.

## Key patterns

- **Auth middleware**: `routes/auth.js` exports `required` and `optional` middleware that decode JWT from the `Authorization: Token <jwt>` header. Most routes use `auth.required`.
- **Error handling**: Validation errors are returned as `{ errors: { field: [messages] } }` with status 422. Use this shape for any new validation.
- **Slug generation**: Articles use a URL slug derived from the title via the `slug` npm package. Slugs must be unique — check `Article.findOne({ slug })` before creating.
- **Mongoose virtuals**: `User` has a `toProfileJSONFor(user)` method that returns the profile shape including whether the requesting user follows this user.

```

</tool_response>

<tool_call>
{"name": "list_directory", "arguments": {"path": "/Users/zivlevi/Code/playground/repos/node-express-realworld-example-app"}}
</tool_call>
<tool_response>
.env
.gitignore
.nvmrc
LICENSE
README.md
jest.config.js
node_modules
package-lock.json
package.json
prisma
project-logo.png
src
</thinking>

The README says it's Prisma, not Mongoose. Let me explore more.
</tool_response>

<tool_call>
{"name": "list_directory", "arguments": {"path": "/Users/zivlevi/Code/playground/repos/node-express-realworld-example-app/src"}}
</tool_call>
<tool_response>
app.ts
config.ts
errors.ts
middlewares
models
routes
utils
</thinking>

TypeScript, not JavaScript. Let me explore further.
</tool_response>

<tool_call>
{"name": "list_directory", "arguments": {"path": "/Users/zivlevi/Code/playground/repos/node-express-realworld-example-app/src/routes"}}
</tool_call>
<tool_response>
api
</thinking>

</tool_response>

<tool_call>
{"name": "list_directory", "arguments": {"path": "/Users/zivlevi/Code/playground/repos/node-express-realworld-example-app/src/routes/api"}}
</tool_call>
<tool_response>
articles.ts
comments.ts
profiles.ts
tags.ts
users.ts
</thinking>

</tool_response>

<tool_call>
{"name": "read_file", "arguments": {"path": "/Users/zivlevi/Code/playground/repos/node-express-realworld-example-app/src/routes/api/users.ts"}}
</tool_call>
<tool_response>
import { Router, Request, Response, NextFunction } from 'express';
import { PrismaClient } from '@prisma/client';
import {
  checkPassword,
  encryptPassword,
  generateToken,
} from '../../utils/auth';
import { LoginUserInput, RegisterUserInput, UpdateUserInput } from '../../models/user';
import { auth } from '../../middlewares/auth';
import HttpException from '../../errors';

const router = Router();
const prisma = new PrismaClient();

router.post('/users/login', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const input = req.body.user as LoginUserInput;

    const email = input.email?.trim();
    const password = input.password?.trim();

    if (!email) {
      throw new HttpException(422, { errors: { email: ["can't be blank"] } });
    }

    if (!password) {
      throw new HttpException(422, { errors: { password: ["can't be blank"] } });
    }

    const user = await prisma.user.findFirst({
      where: {
        email,
      },
    });

    if (!user) {
      throw new HttpException(422, { errors: { email: ['not found'] } });
    }

    const match = await checkPassword(password, user.password);

    if (!match) {
      throw new HttpException(422, { errors: { password: ['is invalid'] } });
    }

    return res.json({
      user: {
        email: user.email,
        username: user.username,
        bio: user.bio,
        image: user.image,
        token: generateToken(user.id),
      },
    });
  } catch (e) {
    next(e);
  }
});

router.post('/users', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const input = req.body.user as RegisterUserInput;

    const email = input.email?.trim();
    const username = input.username?.trim();
    const password = input.password?.trim();

    if (!email) {
      throw new HttpException(422, { errors: { email: ["can't be blank"] } });
    }

    if (!username) {
      throw new HttpException(422, { errors: { username: ["can't be blank"] } });
    }

    if (!password) {
      throw new HttpException(422, { errors: { password: ["can't be blank"] } });
    }

    const existingUserByEmail = await prisma.user.findFirst({
      where: {
        email,
      },
    });

    if (existingUserByEmail) {
      throw new HttpException(422, { errors: { email: ['has already been taken'] } });
    }

    const existingUserByUsername = await prisma.user.findFirst({
      where: {
        username,
      },
    });

    if (existingUserByUsername) {
      throw new HttpException(422, { errors: { username: ['has already been taken'] } });
    }

    const hashedPassword = await encryptPassword(password);

    const user = await prisma.user.create({
      data: {
        email,
        username,
        password: hashedPassword,
      },
    });

    return res.status(201).json({
      user: {
        email: user.email,
        username: user.username,
        bio: user.bio,
        image: user.image,
        token: generateToken(user.id),
      },
    });
  } catch (e) {
    next(e);
  }
});

router.get('/user', auth.required, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const userId = req.user?.id;

    if (!userId) {
      throw new HttpException(401, {});
    }

    const user = await prisma.user.findFirst({
      where: {
        id: userId,
      },
    });

    if (!user) {
      throw new HttpException(404, {});
    }

    return res.json({
      user: {
        email: user.email,
        username: user.username,
        bio: user.bio,
        image: user.image,
        token: generateToken(user.id),
      },
    });
  } catch (e) {
    next(e);
  }
});

router.put('/user', auth.required, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const userId = req.user?.id;

    if (!userId) {
      throw new HttpException(401, {});
    }

    const input = req.body.user as UpdateUserInput;

    const { image, bio, demo } = input;
    const email = input.email?.trim();
    const username = input.username?.trim();
    const password = input.password?.trim();

    const user = await prisma.user.update({
      where: {
        id: userId,
      },
      data: {
        ...(email ? { email } : {}),
        ...(username ? { username } : {}),
        ...(password ? { password: await encryptPassword(password) } : {}),
        ...(image !== undefined ? { image } : {}),
        ...(bio !== undefined ? { bio } : {}),
        ...(demo !== undefined ? { demo } : {}),
      },
    });

    return res.json({
      user: {
        email: user.email,
        username: user.username,
        bio: user.bio,
        image: user.image,
        token: generateToken(user.id),
      },
    });
  } catch (e) {
    next(e);
  }
});

export default router;
</tool_response>

<tool_call>
{"name": "read_file", "arguments": {"path": "/Users/zivlevi/Code/playground/repos/node-express-realworld-example-app/src/routes/api/articles.ts"}}
</tool_call>
<tool_response>
import { Router, Request, Response, NextFunction } from 'express';
import { PrismaClient } from '@prisma/client';
import { auth } from '../../middlewares/auth';
import HttpException from '../../errors';
import { CreateArticleInput, UpdateArticleInput } from '../../models/article';
import slugify from 'slugify';

const router = Router();
const prisma = new PrismaClient();

router.get('/articles', auth.optional, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { tag, author, favorited, limit = 20, offset = 0 } = req.query;

    const where: {
      tags?: { some: { name: string } };
      author?: { username: string };
      favoritedBy?: { some: { username: string } };
    } = {};

    if (tag) {
      where.tags = { some: { name: tag as string } };
    }

    if (author) {
      where.author = { username: author as string };
    }

    if (favorited) {
      where.favoritedBy = { some: { username: favorited as string } };
    }

    const articlesCount = await prisma.article.count({ where });

    const articles = await prisma.article.findMany({
      where,
      orderBy: {
        createdAt: 'desc',
      },
      skip: Number(offset),
      take: Number(limit),
      include: {
        author: true,
        tags: true,
        favoritedBy: true,
        _count: {
          select: {
            favoritedBy: true,
          },
        },
      },
    });

    return res.json({
      articles: articles.map((article) => {
        return {
          slug: article.slug,
          title: article.title,
          description: article.description,
          body: article.body,
          tagList: article.tags.map((tag) => tag.name),
          createdAt: article.createdAt,
          updatedAt: article.updatedAt,
          favorited: article.favoritedBy.some((u) => u.id === req.user?.id),
          favoritesCount: article._count.favoritedBy,
          author: {
            username: article.author.username,
            bio: article.author.bio,
            image: article.author.image,
            following: false,
          },
        };
      }),
      articlesCount,
    });
  } catch (e) {
    next(e);
  }
});

router.get('/articles/feed', auth.required, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { limit = 20, offset = 0 } = req.query;

    const currentUser = await prisma.user.findFirst({
      where: {
        id: req.user?.id,
      },
      include: {
        following: true,
      },
    });

    if (!currentUser) {
      throw new HttpException(401, {});
    }

    const followingIds = currentUser.following.map((u) => u.id);

    const articlesCount = await prisma.article.count({
      where: {
        authorId: {
          in: followingIds,
        },
      },
    });

    const articles = await prisma.article.findMany({
      where: {
        authorId: {
          in: followingIds,
        },
      },
      orderBy: {
        createdAt: 'desc',
      },
      skip: Number(offset),
      take: Number(limit),
      include: {
        author: true,
        tags: true,
        favoritedBy: true,
        _count: {
          select: {
            favoritedBy: true,
          },
        },
      },
    });

    return res.json({
      articles: articles.map((article) => {
        return {
          slug: article.slug,
          title: article.title,
          description: article.description,
          body: article.body,
          tagList: article.tags.map((tag) => tag.name),
          createdAt: article.createdAt,
          updatedAt: article.updatedAt,
          favorited: article.favoritedBy.some((u) => u.id === req.user?.id),
          favoritesCount: article._count.favoritedBy,
          author: {
            username: article.author.username,
            bio: article.author.bio,
            image: article.author.image,
            following: false,
          },
        };
      }),
      articlesCount,
    });
  } catch (e) {
    next(e);
  }
});

router.get('/articles/:slug', auth.optional, async (req: Request, res: Response, next: NextFunction) => {
  try {
    const article = await prisma.article.findFirst({
      where: {
        slug: req.params.slug,
      },
      include: {
        author: true,
        tags: true,
        favoritedBy: true,
        _count: {
          select: {
            favoritedBy: true,
          },
        },
      },
    });

    if (!article) {
      throw new HttpException(404, {});
    }

    return res.json({
      article: {
        slug: article.slug,
        title: article.title,
        description: article.description,
        body: article.body,
        tagList: article.tags.map((tag) => tag.name),
        createdAt: article.createdAt,
        updatedAt: article.updatedAt,
        favorited: article.favoritedBy.some((u) => u.id === req.user?.id),
        favoritesCount: article._count.favoritedBy,
        author: {
          username: article.author.username,
          bio: article.author.bio,
          image: article.author.image,
          following: false,
        },
      },
    

---
**To approve:** apply the `ai-implement` label.
**To reject:** apply `ai-off` with a reason sub-label.