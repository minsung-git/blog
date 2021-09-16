# Advanced NestJS: Dynamic Providers
기사원문: https://dev.to/nestjs/advanced-nestjs-dynamic-providers-1ee

_Livio는 Nest JS 코어팀의 멤버이자 @nestjs/terminus integration의 작성자이다._

## Intro
**의존성 주입**(줄여서 DI)는 용의한 테스트를 위한 Loosely coupled archiecture를 구현하는 위한 강력한 기술입니다. NestJS에서는 이와 같은 요소들을 DI 측면에서 _prvider_ 로 부르고 있습니다. Provider는 2개의 주요 구성 요소로 이루어지는데, 하나는 value 다른 하나는 unique한 토큰입니다. NestJS에서 여러분은 이 토큰을 활용하여 value를 요청할 수 있다. 아래의 snippet은 이에 대한 대표적인 예를 보여주고 있습니다.

```typescript
import { NestFactory } from '@nestjs/core';
import { Module } from '@nestjs/common';

@Module({
  providers: [
    {
      provide: 'PORT',
      useValue: 3000,
    },
  ],
})
export class AppModule {}

async function bootstrap() {
  const app = await NestFactory.createApplicationContext(AppModule);

  const port = app.get('PORT');
  console.log(port); // 출력: 3000
}
bootstrap();
```
```AppMoulde```은 토큰 ```PORT```를 갖는 하나의 프로바이더로 이루어져 있습니다.

- 우리는 ```NestFactory.createApplicationContext```를 호출하여 우리의 어플리케이션을 bootstrap 합니다. (본 함수는 ```NestFactory.create```와 유사한 역할을 합니다, 다만 추가 기능으로 HTTP 인스턴스를 초기화까지를 제공합니다.)
- 그 다음으로 우리는 ```app.get('PORT)```를 통해 프로바이더의 가지고 있는 값을 요청합니다. 이는 프로바이더에 정의한것과 같이  ```3000```을 리턴하게 됩니다. 

이 정도면 충분합니다. 하지만 만약 여러분이 사용자에게 무엇을 제공해야할지 모호하거나 또는 어플리케이션의 Runtime중에 프로바이더를 사용해야 한다면 어떻게 할까요?

본 아티클은 종종 다양한 NestJS 인테이그레이션에서 사용되는 테크닉을 다루고 있습니다. 본 테크닉을 통해 여러분은 매우 Dynamic한 NestJS 어플리케이션을 작성하고 더불어 DI의 장점까지도 활용할 수 있게 될 것 입니다.

## 무엇을 구현할 것인가?
우리는 간단하지만 유용한 동적 프로바이더의 사용사례를 통해 이를 이해하고자 합니다. 여러분은 ```logger```라고 불리는 파라메터 데코레이터를 만들자고 합니다. 이 프로바이더는 ```string``` 타입의 ```prefix```를 갖게 되며, 이 데코레이터는 모든 로그 메세지에 주어진 ```prefix```를 추가하는 ```LoggerService```을 주입하게 됩니다.

즉 최종 구현채의 모습은 아래가 될 것입니다.
```typescript
@Injectable()
export class AppService {
  constructor(@Logger('AppService')) private logger: LoggerService) {}

  getHello() {
    this.logger.log('Hellow World'); // 출력: '[AppService] Hello World'
    return 'Hello World';
  }
}
```
## NestJS 어플리케이션 설정
우리는 빠른 시작을 위해 NestJS CLI를 이용할 것 입니다. 만약 NestJS CLI가 설치 되어 있지 않다면 다음의 커멘드를 활용하여 설치합니다.
```bash
npm i -g @nestjs/cli
```
이제 Nest 어플리케이션의 bootstrap을 위하 터미널에서 다음의 커멘드를 실행 합니다.
```bash
nest new logger-app && cd logger-app
```
## Logger service
이제 우리의 ```LoggerService```를 작성하겠습니다. 이 서비스는 이후에 ```@Logger()```데코레이터를 통해 주입되게 됩니다. 본 서비스에 대한 기본적인 요구 사항은 아래와 같습니다.
- 메세지를 표준 출력을 통해 기록
- 각 인스턴스별로 프리픽스를 설정 할 수 있음
다시 한번 NestJS CLI를 활용하여 모듈과 서비스를 bootstrap합니다.
```bash
nest generate module Logger
nest generate service Logger
```
위의 요구 사항을 충족하기 위하여 작은 ```LoggerService```를 작성 합니다.
```typescript
// src/logger/logger.service.ts

import { Injectable, Scope } from '@nestjs/common';

@Injectable({
  scope: Scope.TRANSIENT,
})
export class LoggerService {
  private prefix?: string;

  log(message: string) {
    let formattedMessage = message;

    if (this.prefix) {
      formattedMessage = `[${this.prefix}] ${message}`;
    }

    console.log(formattedMessage);
  }

  setPrefix(prefix: string) {
    this.prefix = prefix;
  }
}
```
무엇 보다도 여러분은 ```Injectable()``` 데코레이터가 ```Scope.TRANSIENT``` 범위 옵션 사용하고 있다는 점을 확인 할 수 있었을 것입니다. 이는 ```LoggerService```가 매번 이를 필요로 하는 어플리케이션에 주입된다는 점을 의미 합니다. 다시 말해 매번 주입 될 때 마다 ```LoggerService```의 인스턴스가 만들어 지게 됩니다. 이는 ```prefix``` 값이 각 어플리케이션 마다 다르게 설정할 수 있어야 한다는 점 때문입니다. 이를 통해 우리는 싱글 인스턴스를 사용함으로써 매번 ```LoggerService```의 ```prefix``` 값이 계속 오버라이드되는 것을 방지할 수 있습니다. (역주: 즉 주입 받는 어플리케이션 마다 다르게 ```prefix```를 설정할 수 있기 때문에 싱글톤을 사용할 수 없습니다.)

그외에는 ```LoggerService```는 직관적인 코드로 쓰여져야 합니다.

이제 우리는 만들어진 서비스를 ```LoggerMoudle```을 통해 export 하는 일만 남았습니다. 이를 통해 우리는 ```AppMoudle```에서 이를 사용할 수 있겠죠.
```typescript
// src/logger/logger.module.ts

import { Module } from '@nestjs/common';
import { LoggerService } from './logger.service';

@Module({
  providers: [LoggerService],
  exports: [LoggerService],
})
export class LoggerModule {}
```
함께 ```AppService```에서 어떻게 사용되고 있는지를 보시죠.
```typescript
// src/app.service.ts

import { Injectable } from '@nestjs/common';
import { LoggerService } from './logger/logger.service';

@Injectable()
export class AppService {
  constructor(private readonly logger: LoggerService) {
    this.logger.setPrefix('AppService');
  }
  getHello(): string {
    this.logger.log('Hello World');
    return 'Hello World!';
  }
}
```
일단은 만족합니다. 이제 ```npm run start```를 실행하여 어플리케이션을 실행하고, ```curl http://localhost:3000/``` 또는 웹 브라우저의 주소창에 ```http://localhost:3000```을 입력하여 해당 코드를 호출 합니다.

만약 여러분이 문제없이 모든 과정을 진행 했다면 아마 아래와 같은 결과를 확인 하실 수 있을 겁니다.
```
[AppService] Hellow World
```
종습니다. 하지만 조금 아쉽지 않나요? 저는 명시적으로 ```this.logger.setPrefix('AppService')```을 생성자에 입력하는 건 조금 아쉽습니다. 차라리 ```@Logger('AppService')``` 를 이 어플리케이션의 ```logger``` 파라매터의 앞단에 명시하면 어떻까요? 이는 less verbose 함과 동시에 우리의 Logger를 쓰기 위하여 매번 생성자에 이를 기술 하는 일 또한 줄여줄 것입니다.

## Logger Decorator
본 예제 어플리케이션에서, 여러분은 굳이 TypeScript에서 어떻게 데코레이터가 동작하는지 알 필요는 없습니다. 여러분은 단지 함수가 데코레이터로 활용될 수 있다는 점만을 알고 계시면 됩니다.

자 우리의 데코레이터를 직접 만들어 보겠습니다.
```bash
touch src/logger/logger.decorator.ts
```
우리는 ```@nestjs/common```이 제공하는 ```@Inject)()``` 데코레이터를 재활용해보겠습니다.
```typescript
// src/logger/logger.decorator.ts

import { Inject } from '@nestjs/common';

export const prefixesForLoggers: string[] = new Array<string>();

export function Logger(prefix: string = '') {
  if (!prefixesForLoggers.includes(prefix)) {
    prefixesForLoggers.push(prefix);
  }
  return Inject(`LoggerService${prefix}`);
}
```
여러분은 ```@Logger('AppService')```의 의미가 ```@Inject('LoggerServiceAppService')```의 별칭이라는 점을 알아 채셨을 것입니다. 유일하게 특별한 것이 있다면 우리가 ```prefixsForLoggers```배열에 ```prefix```를 추가했다는 점이죠. 조금후에 사용하게 될 이 배열은 우리의 모든 prefix들을 저장하게 됩니다.

잠시만 한가지만 더 이야기하자면, 우리의 Nest 어플리케이션은 ```LoggerServiceAppService```토큰에 대해서 전혀 모릅니다. 고로 동적 프로바이더 및 위의 ```prefixesForLoggers``` 배열을 이용하여 토큰을 만들어 보도록 하겠습니다.

## 동적 프로바이더
본 장에서는 여러분은 동적으로 생성되는 프로바이더를 보게 될 것입니다. 

우리는 1) 각각의 prefix마다 프로바이더를 생성할 것입니다. 2) 각각의 프로바이더는 ```'LoggerService' + prefix``` 형태의 토큰을 갖게 되며, 3) 각 프로바이더는 인스턴스 생성시 ```LoggerService.setPrefix(prefix)```를 통해 호출 되게 됩니다.

이와 같은 요구사항들을 충족하기 위하여 새로운 파일을 만듭니다.
```bash
touch src/logger/logger.providers.ts
```
다음의 코드를 복사하여 여러분의 에디터에서 불어넣습니다.
```typescript
// src/logger/logger.provider.ts

import { prefixesForLoggers } from './logger.decorator';
import { Provider } from '@nestjs/common';
import { LoggerService } from './logger.service';

function loggerFactory(logger: LoggerService, prefix: string) {
  if (prefix) {
    logger.setPrefix(prefix);
  }
  return logger;
}

function createLoggerProvider(prefix: string): Provider<LoggerService> {
  return {
    provide: `LoggerService${prefix}`,
    useFactory: logger => loggerFactory(logger, prefix),
    inject: [LoggerService],
  };
}

export function createLoggerProviders(): Array<Provider<LoggerService>> {
  return prefixesForLoggers.map(prefix => createLoggerProvider(prefix));
}
```
```createLoggerProviders``` 함수는 각각의 ```@Logger()``` 데코레이터를 통해 만들어진 Prefix를 위한 각각의 프로바이더들을 위한 배열을 생성 합니다. 고맙게도 본 프로바이더들 생성되기 전에 우리는 NestJS의 ```useFactory```기능을 통해 ```LoggerService.setPrefix()```함수를 실행할 수 있습니다. (역주: 각 프로바이더별로 Prefix를 할당할 수 있다는 점을 의미 합니다.)

이제 ```LoggerModule```에 이 logger 프로바이더들을 추가하는 일만 남았습니다.
```typescript
// src/logger/logger.module.ts

import { Module } from '@nestjs/common';
import { LoggerService } from './logger.service';
import { createLoggerProviders } from './logger.providers';

const loggerProviders = createLoggerProviders();

@Module({
  providers: [LoggerService, ...loggerProviders],
  exports: [LoggerService, ...loggerProviders],
})
export class LoggerModule {}
```
정말 쉽군요. 잠시만, 이거 JavaScript인데, 제대로 동작할까요? 잠시 이를 설명 드리도록 하겠습니다. 먼저 파일이 로드되자 마자 ```createLoggerProviders``` 가장 먼저 호출되게 됩니다. 이 시점에서 ```logger.decorator.tx```의 ```prefixesForLoggers``` 배열은 비어 있는 상태입니다. 당연하게도 이 시점에서는 ```@Logger()``` 데코레이터가 아직 호출되지 않았기 때문입니다.

그렇다면 어떻게 우리는 이 문제를 해결 할 수 있을까요? 자 성스로운 우리의 [동적 모듈](https://docs.nestjs.com/modules#dynamic-modules)을 이용하면 됩니다. 동적 모듈의 함수를 이용하여 우리는 모듈에 대한 설정들(```@Module``` 데코레이터의 파라메터를 사용합니다.)을 생성할 수 있습니다. 이 함수는 ```@Logger``` 데코레이터가 호출된 직후 호출되게 되며, 심지어는 ```prefixForLoggers``` 배열은 필요한 모든 값들을 갖추고 있을 것입니다.

만약 이와 관련하여 보다 알고 싶은게 있다면 [자바 스크립트 이벤트 루프에 대한 비디오](https://www.youtube.com/watch?v=8aGhZQkoFbQ)을 추천 드립니다.

더불어 우리는 ```LoggerModule```을 _동적 모듈_ 로 다시 작성해야 합니다.

```typescript
// src/logger/logger.module.ts

import { DynamicModule } from '@nestjs/common';
import { LoggerService } from './logger.service';
import { createLoggerProviders } from './logger.providers';

export class LoggerModule {
  static forRoot(): DynamicModule {
    const prefixedLoggerProviders = createLoggerProviders();
    return {
      module: LoggerModule,
      providers: [LoggerService, ...prefixedLoggerProviders],
      exports: [LoggerService, ...prefixedLoggerProviders],
    };
  }
}
```
물론 ```app.module.ts```의 import 구문 역시 수정해야 합니다.
```typescript
// src/logger/app.module.ts

@Module({
  controllers: [AppController],
  providers: [AppService],
  imports: [LoggerModule.forRoot()],
})
export class AppModule {}
```
자 이제 준비됐습니다. 이제 ```app.service.ts```를 수정해주고 잘 동작하는지 보도록 합니다.
```typescript
// src/app.service.ts

@Injectable()
export class AppService {
  constructor(@Logger('AppService') private logger: LoggerService) {}

  getHello() {
    this.logger.log('Hello World'); // Prints: '[AppService] Hello World'
    return 'Hello World';
  }
}
```
```http://localhost:3000```을 호출하면 다음과 같은 로그를 보실 수 것입니다.
```bash
[AppService] Hello World
```
내 우리는 해냈습니다.

## 결론
여러분은 NestJS의 여러 고급기능들을 다뤄 봤습니다. 우리는 어떻게 간단한 데코레이터를 만들 수 있는지 부터, 동적 모듈 및 동적 프라바이더를 볼 수 있었습니다. 여러분은 이 인상적인 기술들을 이용하여 Clean하고 테스트 가능한 코드들을 만들 수 있습니다.

이미 언급했던것과 같이 우리는 내부적으로 ```@nestjs/typeorm```과 ```@nestjs/mongoose```에서 사용된 패턴과 정확하게 동일한 패턴을 구연 하였습니다. 예를 들어 Mongoose에서 우리는 각각의 모듈들을 위한 주입 가능한 프로바이더들을 생성하기 위하여 동일한 방법을 사용했습니다.

본 어플리케이션 코드는 [깃헙 저장소](https://github.com/BrunnerLivio/logger-app)에서 보실 수 있습니다. 저는 벌써 몇몇 작은 기능들이 refactor 했으며, 유닛 테스트 또한 추가해 놓았기 때문에 여러분의 Production환경에서도 본 코드를 활용할 수 있습니다. **Happy hacking :)**
