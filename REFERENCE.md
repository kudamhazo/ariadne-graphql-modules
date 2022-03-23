

## `ObjectType`

New `ObjectType` is base class for Python classes representing GraphQL types (either `type` or `extend type`).


### `__schema__`

`ObjectType` key attribute is `__schema__` string that can define only one GraphQL type:

```python
class QueryType(ObjectType):
    __schema__ = """
    type Query {
        year: Int!
    }
    """
```

`ObjectType` implements validation logic for `__schema__`. It verifies that its valid SDL string defining exactly one GraphQL type. 


### Resolvers

Resolvers are class methods or static methods named after schema's fields:

```python
class QueryType(ObjectType):
    __schema__ = """
    type Query {
        year: Int!
    }
    """

    @staticmethod
    def resolve_year(_, info: GraphQLResolveInfo) -> int:
        return 2022
```

> `ObjectType` could look up return type of `Int` scalar's `serialize` method and compare it with resolver's return type as extra safety net.

If resolver function is not present for field, default resolver implemented by `graphql-core` will be used in its place.

In situations when field's name should be resolved to different value, custom mappings can be defined via `__aliases__` attribute:

```python
class UserType(ObjectType):
    __schema__ = """
    type User {
        id: ID!
        dateJoined: String!
    }
    """
    __aliases__ = {
        "dateJoined": "date_joined"
    }
```

Above code will result in Ariadne generating resolver resolving `dateJoined` field to `date_joined` attribute on resolved object.

If `date_joined` exists as `resolve_date_joined` callable on `ObjectType`, it will be used as resolver for `dateJoined`:

```python
class UserType(ObjectType):
    __schema__ = """
    type User {
        id: ID!
        dateJoined: String
    }
    """
    __aliases__ = {
        "dateJoined": "date_joined"
    }

    @staticmethod
    def resolve_date_joined(user, info) -> Optional[str]:
        if can_see_activity(info.context):
            return user.date_joined

        return None
```

> `ObjectType` could raise error if resolver can't be matched to any field on type.


### `__requires__`

When GraphQL type requires on other GraphQL type (or scalar/directive etc. ect.) `ObjectType` will raise an error about missing dependency. This dependency can be provided through `__requires__` attribute:

```python
class UserType(ObjectType):
    __schema__ = """
    type User {
        id: ID!
        dateJoined: String!
    }
    """


class UsersGroupType(ObjectType):
    __schema__ = """
    type UsersGroup {
        id: ID!
        users: [User!]!
    }
    """
    __requires__ = [UserType]
```

`ObjectType` verifies that types specified in `__requires__` actually define required types. If `__schema__` in `UserType` is not defining `User`, error will be raised about missing dependency.

In case of circular dependencies, special `DeferredType` can be used:

```python
class UserType(ObjectType):
    __schema__ = """
    type User {
        id: ID!
        dateJoined: String!
        group: UsersGroup
    }
    """
    __requires__ = [DeferredType("UsersGroup")]


class UsersGroupType(ObjectType):
    __schema__ = """
    type UsersGroup {
        id: ID!
        users: [User!]!
    }
    """
    __requires__ = [UserType]
```

`DeferredType` makes `UserType` happy about `UsersGroup` dependency, deferring dependency check to `make_executable_schema`. If "real" `UsersGroup` is not provided at that time, error will be raised about missing types required to create schema.


## `SubscriptionType`

Specialized subclass of `ObjectType` that defines GraphQL subscription:

```python
class ChatSubscriptions(SubscriptionType):
    __schema__ = """
    type Subscription {
        chat: Chat
    }
    """
    __requires__ = [ChatType]

    async def resolve_chat(chat_id, *_):
        return await get_chat_from_db(chat_id)

    async def subscribe_chat(*_):
        async for event in subscribe("chats"):
            yield event["chat_id"]
```


## `InputType`

Defines GraphQL input:

```python
class UserCreateInput(InputType):
    __schema__ = """
    input UserInput {
        name: String!
        email: String!
        fullName: String!
    }
    """
    __args__ = {
        "fullName": "full_name",
    }
```

### `__args__`

Optional attribue `__args__` is a `Dict[str, str]` used to override key names for `dict` representing input's data.

Following JSON:

```json
{
    "name": "Alice",
    "email:" "alice@example.com",
    "fullName": "Alice Chains"
}
```

Will be represented as following dict:

```python
{
    "name": "Alice",
    "email": "alice@example.com",
    "full_name": "Alice Chains",
}
```


## `ScalarType`

Allows you to define custom scalar in your GraphQL schema.

```python
class DateScalar(ScalarType):
    __schema__ = "scalar Datetime"

    @staticmethod
    def serialize(value) -> str:
        # Called by GraphQL to serialize Python value to
        # JSON-serializable format
        return value.strftime("%Y-%m-%d")

    @staticmethod
    def parse_value(value) -> str:
        # Called by GraphQL to parse JSON-serialized value to
        # Python type
        parsed_datetime = datetime.strptime(formatted_date, "%Y-%m-%d")
        return parsed_datetime.date()
```

Note that those methods are only required if Python type is not JSON serializable, or you want to customize its serialization process.

Additionally you may define third method called `parse_literal` that customizes value's deserialization from GraphQL query's AST, but this is only useful for complex types that represent objects:

```python
from graphql import StringValueNode


class DateScalar(Scalar):
    __schema__ = "scalar Datetime"

    @staticmethod
    def def parse_literal(ast, variable_values: Optional[Dict[str, Any]] = None):
        if not isinstance(ast, StringValueNode):
            raise ValueError()

        parsed_datetime = datetime.strptime(ast.value, "%Y-%m-%d")
        return parsed_datetime.date()
```

If you won't define `parse_literal`, GraphQL will use custom logic that will unpack value from AST and then call `parse_value` on it.


## `InterfaceType`

Defines intefrace in GraphQL schema:

```python
class SearchResultInterface(InterfaceType):
    __schema__ = """
    interface SearchResult {
        summary: String!
        score: Int!
    }
    """

    @staticmethod
    def resolve_type(obj, info):
        # Returns string with name of GraphQL type representing Python type
        # from your business logic
        if isinstance(obj, UserModel):
            return UserType.graphql_name

        if isinstance(obj, CommentModel):
            return CommentType.graphql_name

        return None

    @staticmethod
    def resolve_summary(obj, info):
        # Optional default resolver for summary field, used by types implementing
        # this interface when they don't implement their own
```


## `UnionType`

Defines GraphQL union:

```python
class SearchResultUnion(UnionType):
    __schema__ = "union SearchResult = User | Post | Thread"
    __requires__ = [UserType, PostType, ThreadType]

    @staticmethod
    def resolve_type(obj, info):
        # Returns string with name of GraphQL type representing Python type
        # from your business logic
        if isinstance(obj, UserModel):
            return UserType.graphql_name

        if isinstance(obj, PostModel):
            return PostType.graphql_name

        if isinstance(obj, ThreadModel):
            return ThreadType.graphql_name

        return None
```


## `DirectiveType`

Defines new GraphQL directive in your schema and specifies `SchemaDirectiveVisitor` for it:


```python
from ariadne import SchemaDirectiveVisitor
from graphql import default_field_resolver


class PrefixStringSchemaVisitor(SchemaDirectiveVisitor):
    def visit_field_definition(self, field, object_type):
        original_resolver = field.resolve or default_field_resolver

        def resolve_prefixed_value(obj, info, **kwargs):
            result = original_resolver(obj, info, **kwargs)
            if result:
                return f"PREFIX: {result}"
            return result

        field.resolve = resolve_prefixed_value
        return field


class PrefixStringDirective(DirectiveType):
    __schema__ = "directive @example on FIELD_DEFINITION"
    __visitor__ = PrefixStringSchemaVisitor
```


## `make_executable_schema`

New `make_executable_schema` takes list of Ariadne's types and constructs executable schema from them, performing last-stage validation for types consistency:

```python
class UserType(ObjectType):
    __schema__ = """
    type User {
        id: ID!
        username: String!
    }
    """


class QueryType(ObjectType):
    __schema__ = """
    type Query {
        user: User
    }
    """
    __requires__ = [UserType]

    @staticmethod
    def user(*_):
        return {
            "id": 1,
            "username": "Alice",
        }


schema = make_executable_schema(QueryType)
```


### Automatic merging of roots

Passing multiple `Query`, `Mutation` or `Subscription` definitions to `make_executable_schema` by default results in schema defining single types containing sum of all fields defined on those types, ordered alphabetically by field name.

```python
class UserQueriesType(ObjectType):
    __schema__ = """
    type Query {
        user(id: ID!): User
    }
    """
    ...


class ProductsQueriesType(ObjectType):
    __schema__ = """
    type Query {
        product(id: ID!): Product
    }
    """
    ...

schema = make_executable_schema(UserQueriesType, ProductsQueriesType)
```

Above schema will have single `Query` type looking like this:

```graphql
type Query {
    product(id: ID!): Product
    user(id: ID!): User
}
```

To opt out of this behavior use `merge_roots=False` option:

```python
schema = make_executable_schema(
    UserQueriesType,
    ProductsQueriesType,
    merge_roots=False,
)
```
