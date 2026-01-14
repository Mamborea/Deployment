# How to Contribute

Guide for extending AREA with new services, webhooks, and reactions.

## Architecture Overview

AREA uses a webhook-based architecture:
- **Services** provide webhooks that trigger when events happen (GitHub push, Discord message, etc.)
- **Reactions** are actions executed when webhooks fire (send email, create issue, etc.)
- Each service is a separate NestJS module with controller, service, and DTOs

## Adding a New Service

### 1. Create Module Structure

```bash
mkdir back/src/myservice
cd back/src/myservice
```

Create these files:
- `myservice.module.ts` - Module definition
- `myservice.controller.ts` - HTTP endpoints
- `myservice.service.ts` - Business logic
- `dto/` - Data transfer objects

### 2. Create Service

**File:** `back/src/myservice/myservice.service.ts`

```typescript
import { Injectable } from '@nestjs/common';
import axios from 'axios';

@Injectable()
export class MyServiceService {
  private readonly baseUrl = 'https://api.myservice.com';

  async listUserResources(userAccessToken: string) {
    const response = await axios.get(`${this.baseUrl}/resources`, {
      headers: this.getHeaders(userAccessToken),
    });
    return this.handleResponse(response);
  }

  async createWebhook(userAccessToken: string, webhookUrl: string, events: string[]) {
    const response = await axios.post(
      `${this.baseUrl}/webhooks`,
      { url: webhookUrl, events },
      { headers: this.getHeaders(userAccessToken) }
    );
    return this.handleResponse(response);
  }

  private getHeaders(accessToken: string) {
    return {
      Authorization: `Bearer ${accessToken}`,
      'Content-Type': 'application/json',
    };
  }

  private async handleResponse(response: any) {
    if (response.status === 204) return { success: true };
    return response.data;
  }
}
```

### 3. Create Controller with Webhook Handler

**File:** `back/src/myservice/myservice.controller.ts`

```typescript
import { Body, Controller, Post, Headers, UseGuards, Req } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { ApiTags, ApiOperation } from '@nestjs/swagger';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Hook } from 'src/shared/entities/hook.entity';
import { ReactionsService } from 'src/reactions/reactions.service';
import { MyServiceService } from './myservice.service';

@ApiTags('myservice')
@Controller('myservice')
export class MyServiceController {
  constructor(
    private readonly myServiceService: MyServiceService,
    private readonly reactionsService: ReactionsService,
    @InjectRepository(Hook)
    private hooksRepository: Repository<Hook>
  ) {}

  @Post('webhook')
  @ApiOperation({ summary: 'Handle MyService webhook events' })
  async webhook(@Body() body: any, @Headers('x-webhook-id') webhookId: string) {
    console.log('MyService webhook received:', webhookId, body);

    const hooks = await this.hooksRepository.find({
      where: { webhookId, service: 'myservice' },
    });

    for (const hook of hooks) {
      const reactions = await this.reactionsService.findByHookId(hook.id);

      for (const reaction of reactions) {
        try {
          await this.reactionsService.executeReaction(reaction, body, hook.userId);
        } catch (error) {
          console.error(`Failed to execute reaction ${reaction.id}:`, error);
        }
      }
    }

    return { success: true };
  }

  @Post('create-webhook')
  @UseGuards(AuthGuard('jwt'))
  async createWebhook(@Req() req, @Body() dto: any) {
    const provider = await this.authService.getMyServiceProvider(req.user.id);
    const result = await this.myServiceService.createWebhook(
      provider.accessToken,
      process.env.MYSERVICE_WEBHOOK_URL,
      dto.events
    );

    const hook = this.hooksRepository.create({
      userId: req.user.id,
      webhookId: result.id,
      service: 'myservice',
    });

    return this.hooksRepository.save(hook);
  }
}
```

### 4. Create Module

**File:** `back/src/myservice/myservice.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Hook } from 'src/shared/entities/hook.entity';
import { AuthModule } from '../auth/auth.module';
import { ReactionsModule } from 'src/reactions/reactions.module';
import { MyServiceController } from './myservice.controller';
import { MyServiceService } from './myservice.service';

@Module({
  imports: [
    AuthModule,
    TypeOrmModule.forFeature([Hook]),
    ReactionsModule,
  ],
  controllers: [MyServiceController],
  providers: [MyServiceService],
  exports: [MyServiceService],
})
export class MyServiceModule {}
```

### 5. Register in AppModule

**File:** `back/src/app.module.ts`

```typescript
import { MyServiceModule } from './myservice/myservice.module';

@Module({
  imports: [
    // ... other modules
    MyServiceModule,
  ],
})
export class AppModule {}
```

### 6. Add OAuth Flow

**File:** `back/src/auth/auth.controller.ts`

Add authentication endpoints:

```typescript
@Get('myservice/url')
async myServiceAuthUrl(@Query('mobile') mobile: string) {
  const clientId = this.configService.getOrThrow('MYSERVICE_CLIENT_ID');
  const redirectUri = this.configService.getOrThrow('MYSERVICE_CALLBACK_URL');

  const stateData = {
    platform: mobile === 'true' ? 'mobile' : 'web',
    nonce: Math.random().toString(36).substring(7),
  };

  const state = Buffer.from(JSON.stringify(stateData)).toString('base64');

  return `https://myservice.com/oauth/authorize?client_id=${clientId}&redirect_uri=${redirectUri}&scope=read write&state=${state}`;
}

@Post('myservice/validate')
@UseGuards(AuthGuard('jwt'))
async myServiceAuthCallback(@Body() body: { code: string }, @Req() req) {
  const accessToken = await this.authService.getMyServiceToken(body.code);
  await this.authService.linkMyServiceAccount(req.user.id, accessToken);
  return { success: true };
}
```

**File:** `back/src/auth/auth.service.ts`

Add token exchange and account linking:

```typescript
async getMyServiceToken(code: string): Promise<string> {
  const response = await axios.post('https://myservice.com/oauth/token', {
    client_id: this.configService.get('MYSERVICE_CLIENT_ID'),
    client_secret: this.configService.get('MYSERVICE_CLIENT_SECRET'),
    code,
    grant_type: 'authorization_code',
  });
  return response.data.access_token;
}

async linkMyServiceAccount(userId: number, accessToken: string) {
  const provider = await this.providersRepository.findOne({
    where: { userId, type: ProviderType.MYSERVICE },
  });

  if (provider) {
    provider.accessToken = accessToken;
    return this.providersRepository.save(provider);
  }

  return this.providersRepository.save({
    userId,
    type: ProviderType.MYSERVICE,
    accessToken,
  });
}

async getMyServiceProvider(userId: number) {
  return this.providersRepository.findOne({
    where: { userId, type: ProviderType.MYSERVICE },
  });
}
```

### 7. Update ProviderType Enum

**File:** `back/src/shared/enums/provider.enum.ts`

```typescript
export enum ProviderType {
  // ... existing providers
  MYSERVICE = 'myservice',
}
```

### 8. Environment Configuration

**File:** `.env`

```env
MYSERVICE_CLIENT_ID=your_client_id
MYSERVICE_CLIENT_SECRET=your_client_secret
MYSERVICE_CALLBACK_URL=http://localhost:8081/myservice/callback
MYSERVICE_WEBHOOK_URL=http://localhost:8080/myservice/webhook
```

### 9. Create OAuth Documentation

Create `docs/oauth/myservice.md` following the pattern of existing guides (see [github.md](oauth/github.md) for example).

## Adding a Reaction

Reactions are executed when webhooks fire. They're defined in `ReactionsService`.

### 1. Add ReactionType

**File:** `back/src/shared/entities/reaction.entity.ts`

```typescript
export enum ReactionType {
  // ... existing types
  MYSERVICE_DO_SOMETHING = 9,
}
```

### 2. Implement Reaction Logic

**File:** `back/src/reactions/reactions.service.ts`

Add case in `executeReaction()`:

```typescript
async executeReaction(reaction: Reaction, webhookPayload: any, userId: number) {
  switch (reaction.reactionType) {
    // ... existing cases

    case ReactionType.MYSERVICE_DO_SOMETHING:
      return this.doMyServiceAction(reaction.config, webhookPayload, userId);
  }
}

private async doMyServiceAction(config: any, webhookPayload: any, userId: number) {
  const provider = await this.authService.getMyServiceProvider(userId);
  if (!provider) throw new Error('MyService account not linked');

  // Process config with webhook data
  const processedConfig = this.replaceVariables(config, webhookPayload);

  // Call MyService API
  await axios.post(
    'https://api.myservice.com/action',
    { data: processedConfig.data },
    {
      headers: {
        Authorization: `Bearer ${provider.accessToken}`,
        'Content-Type': 'application/json',
      },
    }
  );

  return { success: true, message: 'Action executed successfully' };
}
```

### 3. Inject Service if Needed

If your reaction needs complex logic, inject the service:

```typescript
export class ReactionsService {
  constructor(
    // ... existing injections
    private myServiceService: MyServiceService,
  ) {}
}
```

Update `ReactionsModule`:

```typescript
@Module({
  imports: [
    // ... existing imports
    MyServiceModule,
  ],
})
export class ReactionsModule {}
```

## Variable Replacement

The `replaceVariables()` method substitutes `{{variable}}` placeholders with webhook data:

```typescript
// Webhook payload
const webhookPayload = {
  repository: { name: 'my-repo' },
  issue: { number: 42, title: 'Bug fix' }
};

// Reaction config
const config = {
  message: 'New issue #{{issue.number}} in {{repository.name}}: {{issue.title}}'
};

// After processing
// "New issue #42 in my-repo: Bug fix"
```

### Unit Tests

**File:** `back/src/myservice/myservice.service.spec.ts`

```typescript
import { Test } from '@nestjs/testing';
import { MyServiceService } from './myservice.service';

describe('MyServiceService', () => {
  let service: MyServiceService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [MyServiceService],
    }).compile();

    service = module.get(MyServiceService);
  });

  it('should create webhook', async () => {
    const result = await service.createWebhook('token', 'url', ['event']);
    expect(result).toBeDefined();
  });
});
```

## Code Standards

AREA uses **Biome** for linting and formatting.

### Naming Conventions

- **Variables/Functions:** `camelCase`
- **Classes/Interfaces:** `PascalCase`
- **Constants:** `CONSTANT_CASE`
- **Files:** `kebab-case.ts`

### Run Checks

```bash
# Backend
cd back
npm run lint:check      # Check for issues
npm run lint:fix        # Fix issues
npm run format:check    # Check formatting
npm run format          # Fix formatting

# Frontend
cd front
npm run lint:check
npm run lint:fix
npm run format:check
npm run format:fix
```

### Key Rules

- Indent: 2 spaces
- Semicolons required
- Use imports, not require
- Handle errors properly

## Commit Convention

Follow conventional commits:

```
feat(myservice): add webhook integration
fix(reactions): correct variable replacement in emails
docs(oauth): add MyService setup guide
refactor(github): extract common webhook logic
test(discord): add channel creation tests
chore(deps): update @nestjs/common to 11.0.2
```

Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `perf`, `ci`, `build`

## Pull Request Checklist

- [ ] Code follows Biome standards (`npm run lint:check`)
- [ ] All tests pass (`npm test`)
- [ ] New features have tests
- [ ] Documentation updated (API.md, oauth guides)
- [ ] `.env.example` updated if new env vars added
- [ ] Commit messages follow convention
- [ ] PR description explains what and why

## Project Structure Reference

```
back/src/
├── auth/              # Authentication (JWT, OAuth)
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   └── strategies/
├── github/            # GitHub service
│   ├── github.controller.ts
│   ├── github.service.ts
│   ├── github.module.ts
│   └── dto/
├── discord/           # Discord service
├── gmail/             # Gmail service
├── reactions/         # Reaction execution engine
│   ├── reactions.controller.ts
│   ├── reactions.service.ts
│   └── dto/
├── shared/
│   ├── entities/      # TypeORM entities
│   │   ├── user.entity.ts
│   │   ├── provider.entity.ts
│   │   ├── hook.entity.ts
│   │   └── reaction.entity.ts
│   └── enums/
└── users/
```

## Need Help?

- **Existing services:** Check `back/src/github/` or `back/src/discord/` for examples
- **Swagger docs:** http://localhost:8080/api-docs (when server running)
- **Database schema:** See [ARCHITECTURE.md](ARCHITECTURE.md)
- **OAuth setup:** See guides in [docs/oauth/](oauth/)
- **API reference:** See [API.md](API.md)
