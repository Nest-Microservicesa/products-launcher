# Configuración de Microservicios con NATS en NestJS

Esta guía detalla cómo se configuró la comunicación entre microservicios usando **NATS** en este proyecto. Puedes usar este documento como referencia y repaso para replicar la arquitectura en futuros proyectos.

## 1. Infraestructura: Docker Compose

Para que los microservicios se comuniquen entre sí, necesitan un bus de mensajes (Message Broker). En este caso, usamos el servidor oficial de NATS.

En tu archivo `docker-compose.yml` (y `docker-compose.prod.yml`), la configuración mínima de NATS es simplemente la imagen sin dependencias adicionales:

```yaml
  nats-server:
    image: nats:latest
```

> [!NOTE]
> Los demás microservicios se conectan a este servidor inyectando una variable de entorno `NATS_SERVERS` que apunta al servicio interno de Docker. Por defecto, NATS corre en el puerto 4222.

Ejemplo en el `docker-compose`:
```yaml
  auth-ms:
    # ...
    environment:
      - NATS_SERVERS=nats://nats-server:4222
```

## 2. Convirtiendo un proyecto en un Microservicio NATS

Para que una aplicación NestJS funcione como un microservicio que escucha por eventos o pide/responde a través de NATS, debe inicializarse correctamente en su archivo `main.ts`.

Un ejemplo claro lo vemos en tu servicio de autenticación (`auth-ms/src/main.ts`):

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { envs } from './config';

async function bootstrap() {
  // Crear la aplicación como Microservicio indicando NATS
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.NATS, // Define a NATS como medio
      options: {
        // Pasa los servidores NATS (esto viene de tu config, que lee process.env.NATS_SERVERS)
        servers: envs.natsServers
      }
    }
  );

  // Escuchar a los mensajes entrantes (ya no expone puertos HTTP)
  await app.listen();
}
bootstrap();
```

> [!IMPORTANT]
> Un microservicio configurado de esta manera no procesa peticiones HTTP comunes (como REST), sino que consume eventos provenientes del bus de NATS usando decoradores como `@MessagePattern` o `@EventPattern`.

## 3. Modulo de Transporte: Enviando mensajes a otros microservicios

Para que un microservicio **o un Gateway** (como tu `client-gateway`) pueda enviar mensajes o llamar a otros microservicios, requiere registrar un "Cliente" de NATS. 

Has logrado esto creando un módulo global reusable llamado `NatsModule` (`src/transports/nats.module.ts`):

```typescript
import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { envs, NATS_SERVICE } from 'src/config';

@Module({
    imports: [
        ClientsModule.register([
            {
                name: NATS_SERVICE, // ej: const NATS_SERVICE = 'NATS_SERVICE'
                transport: Transport.NATS,
                options: {
                    servers: envs.natsServers,
                }
            }
        ])
    ],
    exports: [
        // Exportando el mismo registro permite que otros módulos 
        // inyecten el cliente NATS automáticamente sin registrarlo de nuevo.
        ClientsModule.register([
            {
                name: NATS_SERVICE,
                transport: Transport.NATS,
                options: {
                    servers: envs.natsServers,
                }
            }
        ])
    ]
})
export class NatsModule { }
```

### ¿Cómo se usa esto en el código (Servicios/Controladores)?

Al importar este `NatsModule` en tus módulos de negocio (por ejemplo, en `AuthModule` o en un controlador del gateway), automáticamente tienes disponible un `ClientProxy` inyectable.

```typescript
import { Controller, Inject } from '@nestjs/common';
import { ClientProxy } from '@nestjs/microservices';
import { NATS_SERVICE } from 'src/config';

@Controller('auth')
export class AuthController {
  constructor(
    @Inject(NATS_SERVICE) private readonly client: ClientProxy
  ) {}

  loginUser(loginDto: LoginDto) {
    // Al usar client.send(), se manda un mensaje por NATS hacia el microservicio
    // adecuado, y se espera una respuesta.
    return this.client.send('auth.login.user', loginDto);
  }
}
```

---

## 🔥 Procesos Clave para Repasar

1. **La inyección pura por variable**: Tienes un archivo de configuración (`src/config/envs.ts` usualmente auxiliado por Joi o Zod) encargado de validar y exportar la variable `envs.natsServers`. Asegúrate de que dicho archivo se encarga de partir en un *Array* cualquier listado separado por comas si utilizas varios nodos de NATS en tu clúster.
2. **El patrón Request-Response (`client.send`) vs. Publicar-Suscribir (`client.emit`)**: 
   - `client.send('patron', payload)`: Úsalo cuando necesitas que el microservicio **responda** con un resultado de vuelta. Se corresponde con el decorador `@MessagePattern` del otro lado. (Ojo: ¡si el servicio falla internamente o nadie escucha, el client.send fallará con timeout o NATS throweará!).
   - `client.emit('evento', payload)`: Utilízalo para cuando simplemente avisas que algo ocurrió pero "lanzas la piedra y escondes la mano". Nadie necesita responder. Se corresponde con `@EventPattern`.
3. **Manejo de Errores (Exceptions / RpcExceptions)**: Cuando un microservicio retorna un error en un flujo normal HTTP lanzas `HttpException`. En microservicios (NATS) lanzas un `RpcException`. Es importante que programes *Exception Filters* (Filtros globales) en tu gateway para tomar los errores de tipo Rpc provenientes de NATS y convertirlos a formato HTTP entendible para el FrontEnd.

> [!WARNING]
> No expongas tus microservicios internos al mundo. Nota que el único servicio con un bloque `ports` hacia internet es el `client-gateway`, además de algún puerto esporádico (como stripe). Tu backend se comunica invisiblemente mediante la red interna de NATS.
