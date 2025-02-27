# Models

Models represent data stored in tables or collections in your database. Models have one or more fields that store codable values. All models have a unique identifier. Property wrappers are used to denote identifiers, fields, and relations. 

Below is an example of a simple model with one field. Note that models do not describe the entire database schema, such as constraints, indexes, and foreign keys. Schemas are defined in [migrations](migration.md). Models are focused on representing the data stored in your database schemas.  

```swift
final class Planet: Model {
    // Name of the table or collection.
    static let schema = "planets"

    // Unique identifier for this Planet.
    @ID(key: .id)
    var id: UUID?

    // The Planet's name.
    @Field(key: "name")
    var name: String

    // Creates a new, empty Planet.
    init() { }

    // Creates a new Planet with all properties set.
    init(id: UUID? = nil, name: String) {
        self.id = id
        self.name = name
    }
}
```

## Schema

All models require a static, get-only `schema` property. This string references the name of the table or collection this model represents. 

```swift
final class Planet: Model {
    // Name of the table or collection.
    static let schema = "planets"
}
```

When querying this model, data will be fetched from and stored to the schema named `"planets"`.

!!! tip
    The schema name is typically the class name pluralized and lowercased. 

## Identifier

All models must have an `id` property defined using the `@ID` property wrapper. This field uniquely identifies instances of your model.

```swift
final class Planet: Model {
    // Unique identifier for this Planet.
    @ID(key: .id)
    var id: UUID?
}
```

By default, the `@ID` property should use the special `.id` key which resolves to an appropriate key for the underlying database driver. For SQL this is `"id"` and for NoSQL it is `"_id"`. 

The `@ID` should also be of type `UUID`. This is the only identifier value currently supported by all database drivers. Fluent will automatically generate new UUID identifiers when models are created. 

`@ID` has an optional value since unsaved models may not have an identifier yet. To get the identifier or throw an error, use `requireID`.

```swift
let id = try planet.requireID()
```

### Exists

`@ID` has an `exists` property that represents whether the model exists in the database or not. When you initialize a model, the value is `false`. After you save a model or when you fetch a model from the database, the value is `true`. This property is mutable.

```swift
if planet.$id.exists {
    // This model exists in database.
}
```

### Custom Identifier

Fluent supports custom identifier keys and types using the `@ID(custom:)` overload. 

```swift
final class Planet: Model {
    // Unique identifier for this Planet.
    @ID(custom: "foo")
    var id: Int?
}
```

The above example uses an `@ID` with custom key `"foo"` and identifier type `Int`. This is compatible with SQL databases using auto-incrementing primary keys, but is not compatible with NoSQL. 

Custom `@ID`s allow the user to specify how the identifier should be generated using the `generatedBy` parameter.

```swift
@ID(custom: "foo", generatedBy: .user)
```

The `generatedBy` parameter supports these cases:

|Generated By|Description|
|-|-|
|`.user`|`@ID` property is expected to be set before saving a new model.|
|`.random`|`@ID` value type must conform to `RandomGeneratable`.|
|`.database`|Database is expected to generate a value upon save.|

If the `generatedBy` parameter is omitted, Fluent will attempt to infer an appropriate case based on the `@ID` value type. For example, `Int` will default to `.database` generation unless otherwise specified.

## Initializer

Models must have an empty initializer method.

```swift
final class Planet: Model {
    // Creates a new, empty Planet.
    init() { }
}
```

Fluent requires this method internally to initialize models returned by queries. It is also used for reflection. 

You may want to add a convenience initializer to your model that accepts all properties. 

```swift
final class Planet: Model {
    // Creates a new Planet with all properties set.
    init(id: UUID? = nil, name: String) {
        self.id = id
        self.name = name
    }
}
```

Using convenience initializers makes it easier to add new properties to the model in the future. 

## Field

Models can have zero or more `@Field` properties for storing data. 

```swift
final class Planet: Model {
    // The Planet's name.
    @Field(key: "name")
    var name: String
}
```

Fields require the database key to be explicitly defined. This is not required to be the same as the property name. 

!!! tip
    Fluent recommends using `snake_case` for database keys and `camelCase` for property names. 

Field values can be any type that conforms to `Codable`. Storing nested structures and arrays in `@Field` is supported, but filtering operations are limited. See [`@Group`](#group) for an alternative.

For fields that contain an optional value, use `@OptionalField`. 

```swift
@OptionalField(key: "tag")
var tag: String?
```

!!! warning
    A non-optional field that has a `willSet` property observer that references its current value or a `didSet` property observer that references its `oldValue` will result in a fatal error.

## Relations

Models can have zero or more relation properties referencing other models like `@Parent`, `@Children`, and `@Siblings`. Learn more about relations in the [relations](relations.md) section.

## Timestamp

`@Timestamp` is a special type of `@Field` that stores a `Foundation.Date`. Timestamps are set automatically by Fluent according to the chosen trigger.

```swift
final class Planet: Model {
    // When this Planet was created.
    @Timestamp(key: "created_at", on: .create)
    var createdAt: Date?

    // When this Planet was last updated.
    @Timestamp(key: "updated_at", on: .update)
    var updatedAt: Date?
}
```

`@Timestamp` supports the following triggers.

|Trigger|Description|
|-|-|
|`.create`|Set when a new model instance is saved to the database.|
|`.update`|Set when an existing model instance is saved to the database.|
|`.delete`|Set when a model is deleted from the database. See [soft delete](#soft-delete).|

`@Timestamp`'s date value is optional and should be set to `nil` when initializing a new model. 

### Timestamp Format

By default, `@Timestamp` will use an efficient `datetime` encoding based on your database driver. You can customize how the timestamp is stored in the database using the `format` parameter.

```swift
// Stores an ISO 8601 formatted timestamp representing
// when this model was last updated.
@Timestamp(key: "updated_at", on: .update, format: .iso8601)
var updatedAt: Date?
```

Note that the associated migration for this `.iso8601` example would require storage in `.string` format.

```swift
.field("updated_at", .string)
```

Available timestamp formats are listed below.

|Format|Description|Type|
|-|-|-|
|`.default`|Uses efficient `datetime` encoding for specific database.|Date|
|`.iso8601`|[ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) string. Supports `withMilliseconds` parameter.|String|
|`.unix`|Seconds since Unix epoch including fraction.|Double|

You can access the raw timestamp value directly using the `timestamp` property.

```swift
// Manually set the timestamp value on this ISO 8601
// formatted @Timestamp.
model.$updatedAt.timestamp = "2020-06-03T16:20:14+00:00"
```

### Soft Delete

Adding a `@Timestamp` that uses the `.delete` trigger to your model will enable soft-deletion.

```swift
final class Planet: Model {
    // When this Planet was deleted.
    @Timestamp(key: "deleted_at", on: .delete)
    var deletedAt: Date?
}
```

Soft-deleted models still exist in the database after deletion, but will not be returned in queries. 

!!! tip
    You can manually set an on delete timestamp to a date in the future. This can be used as an expiration date.

To force a soft-deletable model to be removed from the database, use the `force` parameter in `delete`. 

```swift
// Deletes from the database even if the model 
// is soft deletable. 
model.delete(force: true, on: database)
```

To restore a soft-deleted model, use the `restore` method.

```swift
// Clears the on delete timestamp allowing this 
// model to be returned in queries. 
model.restore(on: database)
```

To include soft-deleted models in a query, use `withDeleted`. 

```swift
// Fetches all planets including soft deleted.
Planet.query(on: database).withDeleted().all()
```

## Enum

`@Enum` is a special type of `@Field` for storing string representable types as native database enums. Native database enums provide an added layer of type safety to your database and may be more performant than raw enums. 

```swift
// String representable, Codable enum for animal types.
enum Animal: String, Codable {
    case dog, cat
}

final class Pet: Model {
    // Stores type of animal as a native database enum.
    @Enum(key: "type")
    var type: Animal
}
```

Only types conforming to `RawRepresentable` where `RawValue` is `String` are compatible with `@Enum`. `String` backed enums meet this requirement by default.

To store an optional enum, use `@OptionalEnum`. 

The database must be prepared to handle enums via a migration. See [enum](schema.md#enum) for more information.

### Raw Enums

Any enum backed by a `Codable` type, like `String` or `Int`, can be stored in `@Field`. It will be stored in the database as the raw value.

## Group

`@Group` allows you to store a nested group of fields as a single property on your model. Unlike Codable structs stored in a `@Field`, the fields in a `@Group` are queryable. Fluent achieves this by storing `@Group` as a flat structure in the database.

To use a `@Group`, first define the nested structure you would like to store using the `Fields` protocol. This is very similar to `Model` except no identifier or schema name is required. You can store many properties here that `Model` supports like `@Field`, `@Enum`, or even another `@Group`. 

```swift
// A pet with name and animal type.
final class Pet: Fields {
    // The pet's name.
    @Field(key: "name")
    var name: String

    // The type of pet. 
    @Field(key: "type")
    var type: String

    // Creates a new, empty Pet.
    init() { }
}
```

After you've created the fields definition, you can use it as the value of a `@Group` property.

```swift
final class User: Model {
    // The user's nested pet.
    @Group(key: "pet")
    var pet: Pet
}
```

A `@Group`'s fields are accessible via dot-syntax.

```swift
let user: User = ...
print(user.pet.name) // String
```

You can query nested fields like normal using dot-syntax on the property wrappers.

```swift
User.query(on: database).filter(\.$pet.$name == "Zizek").all()
```

In the database, `@Group` is stored as a flat structure with keys joined by `_`. Below is an example of how `User` would look in the database.

|id|name|pet_name|pet_type|
|-|-|-|-|
|1|Tanner|Zizek|Cat|
|2|Logan|Runa|Dog|

## Codable

Models conform to `Codable` by default. This means you can use your models with Vapor's [content API](../content.md) by adding conformance to the `Content` protocol.

```swift
extension Planet: Content { }

app.get("planets") { req async throws in 
    // Return an array of all planets.
    try await Planet.query(on: req.db).all()
}
```

When serializing to / from `Codable`, model properties will use their variable names instead of keys. Relations will serialize as nested structures and any eager loaded data will be included. 

### Data Transfer Object

Model's default `Codable` conformance can make simple usage and prototyping easier. However, it is not suitable for every use case. For certain situations you will need to use a data transfer object (DTO). 

!!! tip
    A DTO is a separate `Codable` type representing the data structure you would like to encode or decode. 

Assume the following `User` model in the upcoming examples.

```swift
// Abridged user model for reference.
final class User: Model {
    @ID(key: .id)
    var id: UUID?

    @Field(key: "first_name")
    var firstName: String

    @Field(key: "last_name")
    var lastName: String
}
```

One common use case for DTOs is in implementing `PATCH` requests. These requests only include values for fields that should be updated. Attempting to decode a `Model` directly from such a request would fail if any of the required fields were missing. In the example below, you can see a DTO being used to decode request data and update a model.

```swift
// Structure of PATCH /users/:id request.
struct PatchUser: Decodable {
    var firstName: String?
    var lastName: String?
}

app.patch("users", ":id") { req async throws -> User in 
    // Decode the request data.
    let patch = try req.content.decode(PatchUser.self)
    // Fetch the desired user from the database.
    guard let user = try await User.find(req.parameters.get("id"), on: req.db) else {
        throw Abort(.notFound)
    }
    // If first name was supplied, update it.
    if let firstName = patch.firstName {
        user.firstName = firstName
    }
    // If new last name was supplied, update it.
    if let lastName = patch.lastName {
        user.lastName = lastName
    }
    // Save the user and return it.
    try await user.save(on: req.db)
    return user
}
```

Another common use case for DTOs is customizing the format of your API responses. The example below shows how a DTO can be used to add a computed field to a response.

```swift
// Structure of GET /users response.
struct GetUser: Content {
    var id: UUID
    var name: String
}

app.get("users") { req async throws -> [GetUser] in 
    // Fetch all users from the database.
    let users = try await User.query(on: req.db).all()
    return try users.map { user in
        // Convert each user to GET return type.
        try GetUser(
            id: user.requireID(),
            name: "\(user.firstName) \(user.lastName)"
        )
    }
}
```

Even if the DTO's structure is identical to model's `Codable` conformance, having it as a separate type can help keep large projects tidy. If you ever need to make a change to your models properties, you don't have to worry about breaking your app's public API. You may also consider putting your DTOs in a separate package that can be shared with consumers of your API. 

For these reasons, we highly recommend using DTOs wherever possible, especially for large projects.

## Alias

The `ModelAlias` protocol lets you uniquely identify a model being joined multiple times in a query. For more information, see [joins](query.md#join). 

## Save

To save a model to the database, use the `save(on:)` method.

```swift
planet.save(on: database)
```

This method will call `create` or `update` internally depending on whether the model already exists in the database.

### Create

You can call the `create` method to save a new model to the database.

```swift
let planet = Planet(name: "Earth")
planet.create(on: database)
```

`create` is also available on an array of models. This saves all of the models to the database in a single batch / query. 

```swift
// Example of batch create.
[earth, mars].create(on: database)
```

!!! warning
    Models using [`@ID(custom:)`](#custom-identifier) with the `.database` generator (usually autoincrementing `Int`s) will not have their newly created identifiers accessible after batch create. For situations where you need to access the identifiers, call `create` on each model.

To create an array of models separately, use `map` + `flatten`.

```swift
[earth, mars].map { $0.create(on: database) }
    .flatten(on: database.eventLoop)
```

If using `async`/`await` you can use:

```swift
await withTaskGroup(of: Void.self) { taskGroup in
    [earth, mars].forEach { model in
        taskGroup.addTask { try await model.create(on: database) }
    }
}
```

### Update

You can call the `update` method to save a model that was fetched from the database.

```swift
guard let planet = try await Planet.find(..., on: database) else {
    throw Abort(.notFound)
}
planet.name = "Earth"
try await planet.update(on: database)
```

To update an array of models, use `map` + `flatten`.

```swift
[earth, mars].map { $0.update(on: database) }
    .flatten(on: database.eventLoop)

// TOOD
```

## Query

Models expose a static method `query(on:)` that returns a query builder. 

```swift
Planet.query(on: database).all()
```

Learn more about querying in the [query](./query.md) section.

## Find

Models have a static `find(_:on:)` method for looking up a model instance by identifier. 

```swift
Planet.find(req.parameters.get("id"), on: database)
```

This method returns `nil` if no model with that identifier was found.

## Lifecycle

Model middleware allow you to hook into your model's lifecycle events. The following lifecycle events are supported.

|Method|Description|
|-|-|
|`create`|Runs before a model is created.|
|`update`|Runs before a model is updated.|
|`delete(force:)`|Runs before a model is deleted.|
|`softDelete`|Runs before a model is soft deleted.|
|`restore`|Runs before a model is restored (opposite of soft delete).|

Model middleware are declared using the `ModelMiddleware` or `AsyncModelMiddleware` protocol. All lifecycle methods have a default implementation, so you only need to implement the methods you require. Each method accepts the model in question, a reference to the database, and the next action in the chain. The middleware can choose to return early, return a failed future, or call the next action to continue normally.

Using these methods you can perform actions both before and after the specific event completes. Performing actions after the event completes can be done by mapping the future returned from the next responder.

```swift
// Example middleware that capitalizes names.
struct PlanetMiddleware: ModelMiddleware {
    func create(model: Planet, on db: Database, next: AnyModelResponder) -> EventLoopFuture<Void> {
        // The model can be altered here before it is created.
        model.name = model.name.capitalized()
        return next.create(model, on: db).map {
            // Once the planet has been created, the code 
            // here will be executed.
            print ("Planet \(model.name) was created")
        }
    }
}
```

or if using `async`/`await`:

```swift
struct PlanetMiddleware: AsyncModelMiddleware {
    func create(model: Planet, on db: Database, next: AnyModelResponder) async throws {
        // The model can be altered here before it is created.
        model.name = model.name.capitalized()
        try await next.create(model, on: db)
        // Once the planet has been created, the code 
        // here will be executed.
        print ("Planet \(model.name) was created")
    }
}
```

Once you have created your middleware, you can enable it using `app.databases.middleware`.

```swift
// Example of configuring model middleware.
app.databases.middleware.use(PlanetMiddleware(), on: .psql)
```
