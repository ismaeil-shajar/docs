# Create a multi-tenancy application in Nest.js Part 2 (database setup)

In [part 1](https://dev.to/ismaeil_shajar/create-a-multi-tenancy-application-in-nestjs-part-1-microservices-setup-59cm), we setup nestjs framework, configured and tested the microservice application using nest.js.

## Database
Nest gives us all the tools to work with any SQL and NoSQL database. You have a lot of options, you can also use almost all ORM and libraries in nodejs and typescript, like Sequelize, TypeORM, Prisma, and of course mongoose.

In this application, we will work with MySQL and MongoDB. We will also use the most popular js libraries; Sequelize as ORM for MySQL, and mongoose for MongoDB.

## Database integration

### Sequelize
To start using sequelize; we first need to install the required dependencies which include @nestjs/sequelize, mysql2 because we will connect to MySQL database and other needed dependencies.

```sh
$ npm install --save @nestjs/sequelize sequelize sequelize-typescript mysql2
$ npm install --save-dev @types/sequelize
```
In the services, we will import SequelizeModule in the main modules to set connection configuration:

> ex: app.module.ts

```js
@Module({
  imports: [
    SequelizeModule.forRoot({
      dialect: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      models: [],
    }),
  ],
})
```
The `forRoot()` method will include all the configuration properties. You can read more details [here](https://docs.nestjs.com/techniques/database#sequelize-integration).

After configuring the connection, we need to create a table entity. 
For example, we can set a user model in the user service (will add the connection in the service too) by creating user.model.ts which will look like:

> user.model.ts

```js
/// imports
@Table({tableName:'Users'})
export class Users extends Model<Users> {
    @Column( {allowNull: false })
    firstName: string;

    @Column( {allowNull: false })
    lastName: string;

    @Column( { allowNull: false,unique: true })
    email: string;

    @Column( {allowNull: false})
    password: string;    
      
    @Column( { allowNull: false})
    type: string;
}
```
We should also add the dto:
> create-user-dto.ts

```js
export class CreateUserDto{
    readonly firstName:string
    readonly lastName:string
   readonly   email:string
   readonly password:string
   readonly type:string
}
```
And don't forget to add Users in models array in `forRoot()`
 
Now let's complete the setup and configuration. 
If you don't have a database you need to create an empty table and change Sequelize configuration by adding: `   autoLoadModels: true,
    synchronize: true` .
Then in the module, you will add the repository by adding 
`SequelizeModule.forFeature([Users])` in the imports array.
In our case, we use the main module so it will be:
> user-service.module.ts
```js
@Module({
  imports: [SequelizeModule.forRoot({
    dialect: 'mysql',
    host: 'localhost',
    port: 3306,
    username: 'ismaeil',
    password: 'root',
    database: 'test',
    autoLoadModels: true,
    synchronize: true,
    models: [Users],
  }),SequelizeModule.forFeature([Users])],
  controllers: [UserServiceController],
  providers: [UserServiceService],
})
```

And we will edit main service to add findall and create method:
>user-service.service.ts
```js
@Injectable()
export class UserServiceService {
  constructor(
    @InjectModel(Users)
  private readonly userModel: typeof Users){}
  async findAll(): Promise<Users[]> {
    return this.userModel.findAll() ;
  }
  
  async create( createUserDto:CreateUserDto):Promise<Users> {
    return this.userModel.create(<Users>createUserDto)
  }
}
```

Lastly, edit the controller to enable the use of REST requests to access and edit the database:
>user-service.controller.ts
```js
@Controller('users')
export class UserServiceController {
  constructor(private readonly userServiceService: UserServiceService) {}

  @Get()
  async findAll(){
      return this.userServiceService.findAll();
  }

  @Post()
  async createUser(@Body() createUserDto:CreateUserDto){
    return  this.userServiceService.create(createUserDto)
  }

}
```
Now run the browser and test http://127.0.0.1:3003/users. This should access the database and create a table for the first time and return an empty array.

We can add data using a POST request:
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/hc5lxfele85s1hgma79s.png)

#### Tips
> If we have an existing database and need to import tables with types without a lot of work we can use sequelize-typescript-generator.
> 
> You can search about it to see how it works. But here are some simple steps:
> 
> 1- Download and install npx
> 
> 2- Create a folder to save output typescript models `mkdir models`
> 
> 3- Install sequelize-typescript-generator in your machine: `npm install -g sequelize-typescript-generator`
> 
> 4- Install mysql driver: `npm install -g mysql2`
> 
> 5- Run the command: `npx stg -D mysql -h localhost -p 3306 -d <databaseName> -u <username> -x <password> --indices  --case camel --out-dir models --clean`

Source code available in  [git](https://github.com/ismaeil-shajar/multitenancy) branch database-connection


### mongoose
Just like the previous one, we need to install dependencies to use MongoDB in Nest:
```sh
$ npm install --save @nestjs/mongoose mongoose
```
Import MongooseModule into the root Module

```ts
@Module({
  imports: [MongooseModule.forRoot('mongodb://localhost:27017/test')],
})
```
`forRoot()`  accepts the same configuration as mongoose.connect() from the Mongoose package.



We will use the MongoDB database in the notification service. First, we will add `forRoot()`  in the root module and will create a child module called a message to serve notification messages.

The root module will look like this:
> notification.module.ts
```ts

@Module({
  imports: [MongooseModule.forRoot('mongodb://localhost:27017/test'),
  MessageModule],
  controllers: [NotificationController],
  providers: [NotificationService],
})
```
The message module files will be as follows:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/s1wffsh1047ag1n3pb7l.png)

Because we are using mongoose, we need to create a schema and after that import the repository in a module.

In src/message/schemes we will create message.schema.ts file which will look like this:
```js
export type MessageSchemaDocument = MessageSchema & Document;

@Schema()
export class MessageSchema{
    @Prop()
    name: string    
    
    @Prop()
    createdAt: Date

    @Prop({type:mongoose.Schema.Types.Mixed})
    data: Record<string, any>
}

export const MessageSchemaSchema = SchemaFactory.createForClass(MessageSchema);
```
Put the following code in message.module:
>message.module.ts
```ts
@Module({
  imports: [MongooseModule.forFeature([{name:MessageSchema.name,schema:MessageSchemaSchema}])],
  controllers: [MessageController],
  providers: [MessageService],
})
```

And put the following methods in the message service:
>message.service.ts

```ts
@Injectable()
export class MessageService {
    constructor(@InjectModel(MessageSchema.name) private readonly messageModel: Model<MessageSchemaDocument>) {}
    async findAll () {
        return await this.messageModel.find().exec()
    }    
    async create (messageDto:MessageDto) {
        return await this.messageModel.create(messageDto)
    }
}
```

Create MessageDto:
```ts
export class MessageDto {
    readonly name: string    
    readonly createdAt:Date = new Date();
    readonly data?: any
}
```
For Request mapping:
> message.controller.ts

```ts
@Controller('message')
export class MessageController {
  constructor(private readonly messagenService: MessageService) {}

  @Get()
  async findAll(){
    return this.messagenService.findAll();
  }

  @Post()
  @UsePipes(new ValidationPipe({ transform: true }))
  async create(@Body() messageDto:MessageDto){
    return this.messagenService.create(messageDto);
  }
}
```
*Note: Pipes are used in transforming and validating input data, in our case we can use `@UsePipes(new ValidationPipe({ transform: true }))` to set the empty properties in the Dto with default values. For more details refer to [Pipes](https://docs.nestjs.com/pipes) and [validation](https://docs.nestjs.com/techniques/validation).

Now you can test using a Post request to the URL http://127.0.0.1:3002/message with body:
```json
    {
        "name":"validation",
        "data":{"message":"testing validation message if it success","status":"valid"}
    }
```
To retrieve all the records, use the Get request http://127.0.0.1:3002/message

Source code available in  [git](https://github.com/ismaeil-shajar/multitenancy) branch mongodb-connection

In part 3, we will complete the database setup to use multiple databases depending on the request header.