# Create a multi-tenancy application in Nest.js Part 1 (microservices setup)

## Overview
In Saas applications; multitenancy is a mode of operation where multiple independent instances share the same environment. In plain English, it is when multiple tenants and businesses use the same Saas application.

## Multitenancy Archituecture

I will not cover how to design a multitenancy application in this article, however, you can read about it in detail here: [What is Multi-Tenant Architecture?](https://www.datamation.com/cloud/what-is-multi-tenant-architecture/) By
Andy Patrizio

From here on, we are going to work on an example about the multi-tenants using multiple Databases and microservices architecture.

## What are we going to build?
We will focus on ROLE-BASED ACCESS CONTROL with multi-database and will use two databases -MySQL and MongoDB- for each tenant:
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/keng4t9rg9w8ksccdepc.png)
We will use sample web services to test the flow, for example creating users, sending and receiving notifications... etc.

## Nest.js
From [nest js](https://docs.nestjs.com/): nest (NestJS) is a framework for building efficient, scalable Node.js server-side applications. It uses progressive JavaScript, is built with and fully supports TypeScript (yet still enables developers to code in pure JavaScript), and combines elements of OOP (Object Oriented Programming), FP (Functional Programming), and FRP (Functional Reactive Programming).

## Installation
You will need to install [Node.js](https://nodejs.org/en/), and it must be version 10.13 or higher. I will install v12 LTS and npm. I recommend that you use [nvm](https://github.com/nvm-sh/nvm) to install node.js 
### Setup
You need to install nest cli using npm:
```sh
npm install -g @nestjs/cli
```
then create a new nest.js project:
```sh
nest new multi_tenant 
```

If you encountered an error during the installation like:

<span style="color:red;font-size:14px;">npm ERR! invalid json response body at https://registry.npmjs.org/ajv reason: Unexpected end of JSON input</span>.

You can use this to fix it:
```sh
npm cache clean --force
```

### Microservices setup
Although we can create a monolithic application, it is not usually the cause for mulit-tenancy applications, since they can be quite big, the better -and harder- approach is to use microservices.
Let's start the microservices architecture setup, first install dependency:
```sh
npm i --save @nestjs/microservices
```
We will use Redis as a transport layer so install Redis client using npm:
```sh
npm i --save Redis
```
We also need to install Redis server, I will use docker to install the Redis server:

```sh
docker run --name my-redis-container -p 6379:6379 -d redis
```
Now we need to test the application setup before we create another microservice.

Edit the following:
#### main.ts
 In src/main.ts replace the bootstrap method by:
 ```js
  const app = await NestFactory.create(AppModule);
  app.connectMicroservice<MicroserviceOptions>({
    transport:Transport.REDIS,
    options:{
      url:'redis://127.0.0.1:6379'
    },
  });

  await app.startAllMicroservices();
  await app.listen(3000);
```
### Creating microservices in an application
We will start with two applications: notifications and user services. Using generate app command in nest cli:
```sh
nest g app user-service
nest g app notification
```
Now the application directory will look like this:
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/yue1ln0g5497p3gjhzls.png)

The new service will be created like a new project but will share the same project.json file. We need to change all services' main.ts to work with Redis and change their ports to be unique.

Got to /apps/user-service/src/main.ts and /apps/notification/src/main.ts and add Redis connection and microservice starter>

```js
const app = await NestFactory.create(NotificationModule);
 
 // new connection
 app.connectMicroservice<MicroserviceOptions>({
    transport:Transport.REDIS,
    options:{
      url:'redis://127.0.0.1:6379'
    },
  });
 await app.startAllMicroservices();
 /// the rest
   await app.listen(<port>); // any service must have diffrent port or no port if not use http service and client 
```

Before we start editing we need to start the services in dev mode using the following command:
```sh
npm run start:dev 
npm run start:dev notification 
```
Currently, there is no need to start the user-service.

#### Edit configuration and controller
To send data between services; first we will start with the needed configuration and controller. To make it simple, we will send two integers to the notification service and return the user name and the summation of the two integers.

In the main service app.module, we need to add a client to send data to the notification.

But what does  `app.module.ts` do? The @Module() decorator provides metadata that Nest makes use of to organize the application structure. For more details, you can visit [Nest.js @Module()](https://docs.nestjs.com/modules)


Edit the module file to add microservices ClientsModule and configure it:

> @Module will look like this:
```js
@Module({
  imports: [ClientsModule.register([
    {
      name: 'NOTIFY_SERVICE',
      transport: Transport.REDIS,
      options: {
        url: 'redis://localhost:6379',
      },
    },
  ])],
  controllers: [AppController],
  providers: [AppService],
})
```
ClientsModule is a type of module called a dynamic module. This feature enables you to easily create customizable modules that can register and configure providers dynamically and you can read about it [here](https://docs.nestjs.com/fundamentals/dynamic-modules)

Now, in the app.service we will add a constructer to inject the transport client and edit the getHello method to send the data:

>app.service.ts
```js
  constructor(
      @Inject('NOTIFY_SERVICE') private readonly client: ClientProxy){}
 async getHello(): Promise<string> { // need to use async because we need to wait recieved data

    let recieve= await this.client.send<number>("notify",{user:"Ali",data:{a:1,b:2}}).toPromise();// notify if mapped key will used to in other hand 
     // without toPromise function will return Observable and will not see execute before subscribe so when convert to Promise will recieve data in variable 
  
    return "\t add 1+2="+recieve;
  }
```

The transporters support two methods: `send()` (for request-response messaging) and `emit()` (for event-driven messaging) 

Then in the notification service we will just use it to receive a request and send a response.

> In notification.controller.ts.
> 
> Add a new controller (let's call it notify) and use the annotition @MessagePattern with a mapped key `notify` the same key used in `send()` function.
```js
  @MessagePattern('notify')
  async notify(data:NotifiyData){
    console.log('send')
    Logger.log("notificatoin data"+data.user);
    let a:number=data.data['a'];
    let b:number=data.data['b'];
    console.log(a,b)
    return a+b;
  }
```

We will add an interface in the same class file to map the received data to an object type:
```js
interface NotifiyData{
  user: string;
  data: object;
}
```

#### run
Now run the main and notification services using:

```sh
npm run start:dev 
npm run start:dev notification 
```
Go to the browser and open the main service URL http://localhost:3000/. The output will be `add 1+2=3`

Source code available in  [git](https://github.com/ismaeil-shajar/multitenancy) branch microservices-setup

Go to part 2