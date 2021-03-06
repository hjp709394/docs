## Selectively syncing a subset of your data

By default, DataStore downloads the entire contents of your cloud data source to your local device.
The max number of records that will be stored is configurable [here](~/lib/datastore/conflict.md).

You can utilize selective sync to only persist a subset of your data instead.

Selective sync works by applying predicates to the base and delta sync queries, as well as to incoming subscriptions.

<amplify-block-switcher>
<amplify-block name="Java">

```java
Amplify.addPlugin(new AWSDataStorePlugin(DataStoreConfiguration.builder()
    .syncExpression(Post.class, () -> Post.RATING.gt(5))
    .syncExpression(Comment.class, () -> Comment.STATUS.eq("active"))
    .build()));
```

</amplify-block>
<amplify-block name="Kotlin">

```kotlin
Amplify.addPlugin(AWSDataStorePlugin(DataStoreConfiguration.builder()
    .syncExpression(Post::class.java) { Post.RATING.gt(5) }
    .syncExpression(Comment::class.java) { Comment.STATUS.eq("active") }
    .build()))
```

</amplify-block>
<amplify-block name="RxJava">

```java
RxAmplify.addPlugin(new AWSDataStorePlugin(DataStoreConfiguration.builder()
    .syncExpression(Post.class, () -> Post.RATING.gt(5))
    .syncExpression(Comment.class, () -> Comment.STATUS.eq("active"))
    .build()));
```

</amplify-block>
</amplify-block-switcher>


When DataStore starts syncing, only Posts with `rating > 5` and Comments with `status` equal to `active` will be synced down to the user's local store.

<amplify-callout>

Developers should only specify a single `syncExpression` per model. Any subsequent expressions for the same model will be ignored.

</amplify-callout>

### Reevaluate expressions at runtime
Sync expressions get evaluated whenever DataStore starts.
In order to have your expressions reevaluated, you can execute `Amplify.DataStore.clear()` or `Amplify.DataStore.stop()` followed by `Amplify.DataStore.start()`.

If you have the following expression and you want to change the filter that gets applied at runtime, you can do the following:

<amplify-block-switcher>
<amplify-block name="Java">

```java
public Integer rating = 5;

public void initialize() {
    Amplify.addPlugin(new AWSDataStorePlugin(DataStoreConfiguration.builder()
        .syncExpression(Post.class, () -> Post.RATING.gt(rating))
        .build()));
}

public void changeSync() {
    rating = 1;
    Amplify.DataStore.stop(
        () -> Amplify.DataStore.start(
            () -> Log.i("MyAmplifyApp", "DataStore started"),
            error -> Log.e("MyAmplifyApp", "Error starting DataStore: ", error)
        ),
        error -> Log.e("MyAmplifyApp", "Error stopping DataStore: ", error)
    );
}
```

</amplify-block>
<amplify-block name="Kotlin">

```kotlin
var rating = 5

fun initialize() {
    Amplify.addPlugin(AWSDataStorePlugin(DataStoreConfiguration.builder()
        .syncExpression(Post::class.java) { Post.RATING.gt(5) }
        .build()))
}

fun changeSync() {
    rating = 1
    Amplify.DataStore.stop({
        Amplify.DataStore.start(
            { Log.i("MyAmplifyApp", "DataStore started") },
            { error: DataStoreException? -> Log.e("MyAmplifyApp", "Error starting DataStore", error) }
        )},
        { error: DataStoreException? -> Log.e("MyAmplifyApp", "Error stopping DataStore", error) })
}
```

</amplify-block>
<amplify-block name="RxJava">

```java
public Integer rating = 5;

public void initialize() {
    RxAmplify.addPlugin(new AWSDataStorePlugin(DataStoreConfiguration.builder()
        .syncExpression(Post.class, () -> Post.RATING.gt(rating))
        .build()));
}

public void changeSync() {
    rating = 1;
    RxAmplify.DataStore.stop()
        .andThen(RxAmplify.DataStore.start())
        .subscribe(
            () -> Log.i("MyAmplifyApp", "DataStore restarted"),
            error -> Log.e("MyAmplifyApp", "Error restarting DataStore: ", error)
        );
}
```

</amplify-block>
</amplify-block-switcher>

Each time DataStore starts (via `start` or any other operation: `query`, `save`, `delete`, or `observe`), DataStore will reevaluate the `syncExpressions`.

In the above case, the predicate will contain the value `1`, so all Posts with `rating > 1` will get synced down.

Keep in mind: `Amplify.DataStore.stop()` will retain the local store's existing content. Run `Amplify.DataStore.clear()` to clear the locally-stored contents.

<amplify-callout>

When applying a more restrictive filter, clear the local records first by running `DataStore.clear()` instead:

</amplify-callout>

<amplify-block-switcher>
<amplify-block name="Java">

```java
public void changeSync() {
    rating = 8;
    Amplify.DataStore.clear(
        () -> Amplify.DataStore.start(
            () -> Log.i("MyAmplifyApp", "DataStore started"),
            error -> Log.e("MyAmplifyApp", "Error starting DataStore: ", error)
        ),
        error -> Log.e("MyAmplifyApp", "Error clearing DataStore: ", error)
    );
}
```

</amplify-block>
<amplify-block name="Kotlin">

```kotlin
fun changeSync() {
    rating = 8
    Amplify.DataStore.clear({
        Amplify.DataStore.start(
            { Log.i("MyAmplifyApp", "DataStore started") },
            { error: DataStoreException? -> Log.e("MyAmplifyApp", "Error starting DataStore", error) }
        )},
        { error: DataStoreException? -> Log.e("MyAmplifyApp", "Error clearing DataStore", error) })
}
```

</amplify-block>
<amplify-block name="RxJava">

```java
public void changeSync() {
    rating = 8;
    RxAmplify.DataStore.clear()
        .andThen(RxAmplify.DataStore.start()
        .subscribe(
            () -> Log.i("MyAmplifyApp", "DataStore cleared and restarted"),
            error -> Log.e("MyAmplifyApp", "Error clearing or restarting DataStore: ", error)
        );
}
```

</amplify-block>
</amplify-block-switcher>

This will clear the contents of your local store, reevaluate your sync expressions and re-sync the data from the cloud, applying all of the specified predicates to the sync queries.

You can also have your sync expression return `QueryPredicates.all()` in order to remove any filtering for that model. This will have the same effect as the default sync behavior.

<amplify-block-switcher>
<amplify-block name="Java">

```java
public Integer rating = null;

public void initialize() {
    Amplify.addPlugin(new AWSDataStorePlugin(DataStoreConfiguration.builder()
        .syncExpression(Post.class, () -> {
            if(rating != null) {
                return () -> Post.RATING.gt(rating);
            }
            return QueryPredicates.all();
        })
        .build()));
}
```

</amplify-block>
<amplify-block name="Kotlin">

```kotlin
var rating = null

fun initialize() {
    Amplify.addPlugin(AWSDataStorePlugin(DataStoreConfiguration.builder()
        .syncExpression(Post::class.java) {
            if (rating != null) {
                Post.RATING.gt(rating)
            }
            QueryPredicates.all()
        }
        .build()))
}
```

</amplify-block>
<amplify-block name="RxJava">

```java
public Integer rating = null;

public void initialize() {
    RxAmplify.addPlugin(new AWSDataStorePlugin(DataStoreConfiguration.builder()
        .syncExpression(Post.class, () -> {
            if(rating != null) {
                return () -> Post.RATING.gt(rating);
            }
            return QueryPredicates.all();
        })
        .build()));
}
```

</amplify-block>
</amplify-block-switcher>

<amplify-callout warning>

`DataStore.configure()` should only by called once.

</amplify-callout>

### Advanced use case - Query instead of Scan
You can configure selective sync to retrieve items from DynamoDB with a [query operation](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.html) against a GSI. By default, the base sync will perform a [scan](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Scan.html). Query operations enable a highly efficient and cost-effective data retrieval for customers running DynamoDB at scale. Learn about creating GSIs with the `@key` directive [here](https://docs.amplify.aws/cli/graphql-transformer/key).

In order to do that, your `syncExpression` should return a predicate that maps to a query expression.

For example, for the following schema:
```graphql
type User @model
  @key(name: "byLastName", fields: ["lastName", "createdAt"]) {
  id: ID!
  firstName: String!
  lastName: String!
  createdAt: AWSDateTime!
}
```

To construct a query expression, return a predicate with the primary key of the GSI. You can only use the `eq` operator with this predicate.

For the schema defined above `User.LAST_NAME.eq("Doe")` is a valid query expression.

Optionally, you can also chain the sort key to this expression, using any of the following operators: `eq | ne | le | lt | ge | gt | beginsWith | between`. 

E.g., `User.LAST_NAME.eq("Doe").and(User.CREATED_AT.gt("2020-10-10")`.

Both of these sync expressions will result in AWS AppSync retrieving records from Amazon DynamoDB via a query operation:

<amplify-block-switcher>
<amplify-block name="Java">

```java
Amplify.addPlugin(new AWSDataStorePlugin(DataStoreConfiguration.builder()
    .syncExpression(User.class, () -> User.LAST_NAME.eq("Doe"))
    .build()));

// OR

Amplify.addPlugin(new AWSDataStorePlugin(DataStoreConfiguration.builder()
    .syncExpression(User.class, () -> User.LAST_NAME.eq("Doe").and(User.CREATED_AT.gt("2020-10-10")))
    .build()));
```

</amplify-block>
<amplify-block name="Kotlin">

```kotlin
Amplify.addPlugin(AWSDataStorePlugin(DataStoreConfiguration.builder()
    .syncExpression(User::class.java) { User.LAST_NAME.eq("Doe") }
    .build()))

// OR

Amplify.addPlugin(AWSDataStorePlugin(DataStoreConfiguration.builder()
    .syncExpression(User::class.java) { User.LAST_NAME.eq("Doe").and(User.CREATED_AT.gt("2020-10-10")) }
    .build()))
```

</amplify-block>
<amplify-block name="RxJava">

```java
RxAmplify.addPlugin(new AWSDataStorePlugin(DataStoreConfiguration.builder()
    .syncExpression(User.class, () -> User.LAST_NAME.eq("Doe"))
    .build()));

// OR

RxAmplify.addPlugin(new AWSDataStorePlugin(DataStoreConfiguration.builder()
    .syncExpression(User.class, () -> User.LAST_NAME.eq("Doe").and(User.CREATED_AT.gt("2020-10-10")))
    .build()));
```

</amplify-block>
</amplify-block-switcher>
