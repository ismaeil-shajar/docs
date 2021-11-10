# Create multi-tenancy application in Nest.js Part 3 (multi-database setup)

in part 1 we set up the nestjs framework also config and test microservices architecture application using nest.js, and in part 2 we start using Sequelize and mongoose to access the database and test both MySQL database and MongoDB.

## Async Connection
in this part we will see how to let the application connect to multiple databases depending on the request this is because we need to use SaaS application using multiple databases so any tenant has his database contains has data and all tenants access the same application so an application will connect to different databases.
will change the pass repository option method and use `forRootAsync()` instead of `forRoot()` and need to use a custom class for configuration.

so for both sequelize and mongoose  will add 
```js
MongooseModule.forRootAsync({
    useClass:MongooseConfigService
  }),
SequelizeModule.forRootAsync({
      useClass:SequelizeConfigService
})
```
and about a class will create a config file and create two class MongooseConfigService and SequelizeConfigService 
>>the plan here is to add injection for each incoming request and use a domain to switch between connection
> so we need to use `@Injectable({scope:Scope.REQUEST})` and on class constructor  `@Inject(REQUEST) private read-only request` and we can get host information from request data.
>  for example let say the domain is example.com so for tenant call company1 domain will be company1.example.com and for company2 domain will be company2.example.com and so on.


> config/MongooseConfigService.ts

```ts
import { Inject, Injectable, Scope } from "@nestjs/common";
import { REQUEST } from "@nestjs/core";
import { MongooseModuleOptions, MongooseOptionsFactory } from "@nestjs/mongoose";

@Injectable({scope:Scope.REQUEST})
export class MongooseConfigService implements MongooseOptionsFactory {
    constructor(@Inject(REQUEST) private readonly request,){}

  createMongooseOptions(): MongooseModuleOptions {
    let domain:string[]
    let database='database_development'
    if(this.request.data ){
      domain=this.request.data['host'].split('.')
      console.log(this.request)
    }
    else{
      domain=this.request['headers']['host'].split('.')
    }

    console.log(domain)
    if(domain[0]!='127' && domain[0]!='www' && domain.length >2){
      database='tenant_'+domain[0]
      console.log('current DB',database)
    }
    return {
      uri: 'mongodb://localhost:27017/'+database,
    };
  }
}
```
> config/SequelizeConfigService.ts

```ts
import { Inject, Injectable, Scope } from "@nestjs/common";
import { REQUEST } from "@nestjs/core";
import { CONTEXT, RedisContext, RequestContext } from "@nestjs/microservices";
import { SequelizeModuleOptions, SequelizeOptionsFactory} from "@nestjs/sequelize";

@Injectable({scope:Scope.REQUEST})
export class SequelizeConfigService implements SequelizeOptionsFactory {
    constructor(@Inject(REQUEST) private readonly request:RequestContext){}
    
    createSequelizeOptions(): SequelizeModuleOptions {

      let domain:string[]
      let database='database_development'
      if(this.request.data ){
        domain=this.request.data['host'].split('.')
        console.log(this.request)
      }
      else{
        domain=this.request['headers']['host'].split('.')
      }

      console.log(domain)
      if(domain[0]!='127' && domain[0]!='www' && domain.length >2){
        database='tenant_'+domain[0]
        console.log('current DB',database)
      }

    return {
      dialect: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'ismaeil',
      password: 'root',
      database: database,
      autoLoadModels: true,
      synchronize: true,
    };
  }
}
```

## Testing
after finishing the configuration now we need to do some work to test because we need to map our localhost and IP to a domain.
I will try to use two ways to test the application locally but for the production, it will be configuration in your domain provider.
### 1- edit hosts file in your local machine and edit this file every time you add a tenant
in Linux edit `/etc/hosts` file and in windows `c:\windows\system32\drivers\etc\hosts`  and add
``` sh
## lines
127.0.0.1	example.com
127.0.0.1	company1.example.com
127.0.0.1	company2.example.com
```
### 2- use local dns
in linux you can install dnsmasq and follow this step
> 1- install dnsmasq in NetworkManager
> 2- add configuration file `sudo nano /etc/NetworkManager/dnsmasq.d/dnsmasq-localhost.conf`
> add this line in file
```
address=/.example.com/127.0.0.1
```
> 3- restart service `sudo systemctl reload NetworkManager`


Source code available in  [git](https://github.com/ismaeil-shajar/multitenancy) branch multi-database

Next in part 4 will add security level and user roles