# Middleware chain in Next.js
> Заметка для памяти, будет мало текста, результаты получаемые вами могут варьироваться

> Предполагется, некий уровень понимания предмета разговора и первоначального чтения и осмысления раздела документации https://nextjs.org/docs/app/building-your-application/routing/middleware и примеров использования ( например, https://medium.com/@zachshallbetter/middleware-in-next-js-a-comprehensive-guide-7dd0a928541a) .

### Предмет заметки
Middleware в Next.js мощный инструмент, обеспечиващий механизм перехвата входящих http- запросов, запуска некой логики обрабтки данных и возврата ответа с требуемыми или измененными данными.
Позволяет выносить специфичную логику с клиентской части приложения, перенося зону ответвенности на уровень ниже (на уровень логики веб-сервера).
В целом, нужная и полезная вещь, но требующая некоторого внимания, так как отличается от логики middleware в express.js, например.

### Проблема.

В Next.js 14 была изменена логика расположения и очередности запуска. На данный момент, существет только один обработчик middleware.ts, расположеный в корне проекта, запускаемый для всех запросов.
Один файл, с множеством ответственостей, некрасиво.

### Задача

Хотим, что бы каждый обработчик ( иначе, каждая ответвенность) распологался в своем файле, выыполнялись обработчики в определенной последовательности, и цепочка обработки позволяла выйти из нее в любой момент ( с выполнением действия ( редиректа, например) или без). 
А еще, хотим асиинхронность.

### Решение
Рассмотрим _возможное_ решение поставленной задачи. 

Идея в том, что бы собрать все нужные обрабочики ( для определенного пути, ессно) в одно место, запускать их поочередно, расшарив объект `NextResponse.next()` для изменений ( по аналогии с middleware в express.js)
и, если, текущий обработчик не вернул `NextResponse | Response` продолжаем обработку, и в конце цепочки, возвращаем вызвавшей нас функции `NextResponse`

Прежде всго, при написании своих обработчиков, обращаем внимание на базовые типы

```typescript
export type NextMiddlewareResult = NextResponse | Response | null | undefined | void;
export type NextMiddleware = (request: NextRequest, event: NextFetchEvent) => NextMiddlewareResult | Promise<NextMiddlewareResult>;
```

Определим тип нашего обработчика и фабрики

```typescript
export type CustomMiddleware = (
    request: NextRequest,
    event: NextFetchEvent,
    response: NextResponse,
) => NextMiddlewareResult | Promise<NextMiddlewareResult>;

export type MiddlewareFactory = () => CustomMiddleware;
```

Функцию-экзекутор определим как
```typescript
type execMiddleware = (
    request: NextRequest,
    event: NextFetchEvent,
    response: NextResponse,
    middlewares: MiddlewareFactory[],
) => Promise<NextMiddlewareResult>;
```

Создаем папку в проекте `middlewares`, создаем требуемые файлы. 

`middlewares/responseHeaderMiddleware.ts`
```typescript
import { NextResponse, NextRequest } from "next/server";
import { NextFetchEvent } from "next/dist/server/web/spec-extension/fetch-event";

export function responseHeaderMiddleware() {
  return async (
    request: NextRequest,
    event: NextFetchEvent,
    response: NextResponse,
  ) => {
    // Perform whatever logic the  middleware needs to do
    const pathname = request.nextUrl.pathname;
    console.log("middleware2 =>", { pathname });

    response.headers.set("Content-Security-Policy", "default-src 'self'");
    response.headers.set("Access-Control-Allow-Origin", "*");
  };
}
```

`middlewares/redirectMiddleware.ts`
```typescript
import { NextResponse, NextRequest } from "next/server";
import { NextFetchEvent } from "next/dist/server/web/spec-extension/fetch-event";

export function redirectMiddleware() {
    
  return async (
    request: NextRequest,
    event: NextFetchEvent,
    response: NextResponse,
  ) => {
    return NextResponse.redirect(new URL("/dashboard", request.url));
  };
}
```

По умолчанию, в конуе цепочки будем всегда возвращать `NextRespone`, для этого будем выполнять `middlewares/responseNextMiddleware.ts`
```typescript
import { NextResponse, NextRequest } from "next/server";
import { NextFetchEvent } from "next/dist/server/web/spec-extension/fetch-event";

export function responseNext() {
    
  return async (
    request: NextRequest,
    event: NextFetchEvent,
    response: NextResponse,
  ) => {
    return response;
  };
}
```

 Требуемю логику опрделяем в `middlewares/index.ts`
```typescript
import { responseNext } from "@/lib/middlewares/respnseNext";


export async function execMiddleware(
    request: NextRequest,
    event: NextFetchEvent,
    response: NextResponse,
    middlewares: MiddlewareFactory[],
): Promise<NextMiddlewareResult> {
    
    async function exec(
        middlewares: MiddlewareFactory[],
        index = 0,
    ): Promise<NextMiddlewareResult | Promise<NextMiddlewareResult>> {
        const current = middlewares[index];
        return (
            (await current()(request, event, response)) ||
            exec(middlewares, index + 1)
        );
    }

    return await exec([...middlewares, responseNext]);
}
```

Возвращаемся в `middleware.ts`
```typescript
export async function middleware(request: NextRequest, event: NextFetchEvent) {
    // shared response for middlewares
    let sharedResponse: NextResponse = NextResponse.next();

    const middlewares: MiddlewareFactory[] = [
        responseHeaderMiddleware,
        //redirectMiddleware,
    ];

    return await execMiddleware(request, event, sharedResponse, middlewares);
}
```

Данный пример будет устанавливать хедеры, и отдавать запрос. Раскоментируйте `// redirectMiddleware`, будет редиректить

В целом, это все. Спасибо.

Sponsored by [Hablsoft](https://www.hablsoft.com)
