# Create Custom Database Module in NestJS
기사원문: https://dev.to/10xtarun/create-custom-database-module-in-nestjs-4kim

## 소개
NestJS는 이미 데이터베이스 연동을 위하여 TypeORMModule for MySQL과 MongooseModule for MongoDB등을 제공 하고 있습니다. 거기에 더불어 여러분은 여러분이 직접 구현했거나 또는 커스톰한 데이터베이스를 연동해야 할지도 모릅니다. 본 문서는 이러한 경우에 대하여 MongoDB의 Native Driver를 예시 삼아 어떻게 데이터베이스를 연동하고 활용할 수 있는지를 설명 하고 있습니다.

### 사전 요구 사항
본 문서는 TypeScript에 대한 이해를 바탕으로 NestJS를 시작하고자 하는 분들을 대상으로 하고 있습니다. 추가적으로 TypeORM 또는 Mongoose 모듈을 이용해 보았다면 도움이 될 수 있습니다.

## 커스톰 프로바이더 in NestJS
여러분들이 NestJS를 이용해봤다면 (물론 그렇기 때문에 여러분은 본 문서를 읽고 있겠지만), 여러분은 아마 의존성 역전을 통하여 생성한 프로바이더들을 이용해 봤을 것입니다. 여러분은 여러 요구 사항들을 수용하기 위한 여러분의 커스톰한 프로바이더를 작성할 수 있습니다. 본 문서와 함께 여러분이 작성할 커스톰 프로바이더와 이에 따른 모듈은 [다음](https://docs.nestjs.com/fundamentals/custom-providers)에서 확인 하실 수 있습니다.

## 모듈 in NestJS
어플리케이션의 구성 요소를 특정 역할 또는 기능을 기준으로 분류해 놓은 것을 모듈이라 부를 수 있습니다. NestJS에서는 Root 레벨에 최소한 하나의 모듈이 존재해야 합니다. 여러분은 본 예제를 통하여 MongoDB를 위한 커스톰 데이터베이스 프로바이더를 만들어 볼 것입니다.

## 커스톰 데이터베이스 모듈 작성

### 사전 작업
1. 새로운 Nest 프로젝트를 작성합니다. 

    ```
    $ nest new custom-db-project 
    ```
2. 이제 커스톰 데이터베이스 모듈을 생성합니다.

    ```
    $ nest generate module database
    ```
3. 데이터베이스를 위한 프로바이더를 생성합니다.

    ```
    $ touch src/database/database.provider.ts
    ```

4. 추가로 MongoDB를 위한 드라이버도 설치합니다.

    ```
    $ npm install --save mongodb
    ```
### 모듈 작성
자 이제 코딩을 시작하겠습니다. 먼저 Mongodb 드라이버를 위한 프로바이더를 작성 합니다.
```typescript
import * as mongodb from 'mongodb';

export const databaseProviders = [
    {
        provide: 'DATABASE_CONNECTION',
        useFactory: async (): Promise<mongodb.Db> => {
            try {
                const client = await mongodb.MongoClient.connect(
                    'mongodb://localhost',
                    { useUnifiedTopology: true }
                );
                const db = client.db('test');
                return db;
            } catch (error) {
                throw error;
            }
        }
    }
]
```
여러분은 ```provider``` 이름의 ```'DATABASE_CONNECTION'```를 이용하여 프로바이더를 정의하였습니다. 이 문자열은 향후 타 모듈에서 본 데이터베이스 프로바이더를 주입하기 위하여 사용될 것입니다.

```useFactory```를 통해 여러분은 mongoDB 드라이버를 초기화 하게 되며, 결과적으로 ```db``` 라고 하는 인스턴스를 반환하게 됩니다. 해당 인스턴스는 바로 타 모듈에서 실제로 사용되는 실체 입니다.

다음으로 ```connect``` 함수를 통해 연결이 establish 되며, ```test``` 테스트데이터베이스로 연결이 완료된 후 최종적으로 ```db```가 리턴되게 됩니다.

이제 여러분은 이전에 생성한 database 모듈를 다시 확인할 시간입니다.
```typescript
import { Module } from '@nestjs/common';

@Module({})
export class DatabaseModule {}
```
다음과 같이 해당 모듈 구성을 변경해 보도록 하겠습니다.
```typescript
import { Module } from '@nestjs/common';
import { databaseProviders } from './database.provider';

@Module({
    providers: [...databaseProviders],
    exports: [...databaseProviders]
})
export class DatabaseModule {}
```
여러분은 이미 본 모듈이 ```databaseProviders``` 배열의 인스턴스들 (객채들)을 프로바이더로 고려 하도록 기술해 놓았습니다. 이를 통해 데이터베이스 프로바이더는 프로젝트의 구성으로 인지 되게 됩니다. 또한 여러분은 본 프로바이더들을 export하여 타 모듈에서 사용할 수 있도록 하였습니다.

## 사용자 모듈 작성
이제 본 프로바이더를 사용하여 또 다른 모듈을 정의하여 Database Module를 사용하는 방법을 확인토록 하겠습니다.

### 사전 작업
1. 먼저 todo 모듈을 작성 합니다.
    ```
    $ nest generate module todo
    ```
2. 데이터베이스 모듈과 실제 인터렉션하는 todo service를 작성합니다. 이 과정은 또한 ```todo.service.spec.ts``` 파일도 생성하게 되는데, 이는 테스트를 위하여 사용되는 파일로써 본 데모에서는 무시하셔도 됩니다.
    ```
    $ nest generate service todo
    ```
이제 TodoModule에 데이터베이스 모듈을 추가할 차례입니다.

### 모듈 작성
아래와 같이 데이터베이스 모듈을 추가합니다.
```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from 'src/database/database.module';
import { TodoService } from './todo.service';

@Module({
    imports: [DatabaseModule],
    providers: [TodoService]
})
export class TodoModule {}
```
```imports``` 를 통해 여러분은 데이터베이스 모듈을 todo 모듈에서 사용할 수 있도록 정의하였습니다. 이는 동시에 데이터베이스 모듈의 프로바이더들도 사용할 수 있도록 합니다.

이제 todo 서비스 프로바이더에 데이터베이스 프로바이더를 주입합니다. 이를 통해 해당 서비스 (todo 서비스 프로바이더)는 데이터베이스의 기능들를 엑세스 할 수 있습니다.
```typescript
import { Inject, Injectable } from '@nestjs/common';
import * as mongodb from 'mongodb';

@Injectable()
export class TodoService {
    constructor(
        @Inject('DATABASE_CONNECTION') private db: mongodb.Db
    ){}

    async getAllTodos(): Promise<any[]> {
        return await this.db.collection('todos').find({}).toArray();
    }
}
```
여러분은 프로바이더의 선언시 사용된 프바이더의 이름 ```DATABASE_CONNECTION``` 를 기억하고 계실 것입니다. 여러분은 해당 이름(역주: 일반적으로는 인젝션 토큰으로 부름)을 이용하여 결과적으로 ```db``` 인스턴스로가 의존성 주입을 통해 정의 되고있음을 볼 수 있습니다.

그리고 ```getAllTodos``` 함수를 통해 여러분은 어떻게 ```db``` 을 통해 주어진 컨렉션에서 데이터를 찾아 내는지를 볼 수 있습니다.

## 마치며
드디어 여러분은 NestJS에서 MongoDB의 커스톰 데이터베이스 모듈을 구현하였습니다. 또한 본 모듈은 mongoose와는 달리 여러 프로젝트들에서 요구사항이 될 지도 모르는 스키마 사용을 강제하지 않습니다.

