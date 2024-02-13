# Suggest

The **Suggest Rust component** provides address bar search suggestions from Mozilla. This includes suggestions from sponsors, as well as non-sponsored suggestions for other web destinations. These suggestions are part of the [Firefox Suggest](https://support.mozilla.org/en-US/kb/firefox-suggest-faq) feature.

This component is integrated into Firefox Desktop, Android, and iOS.

## Architecture

Search suggestions from Mozilla are stored in a [Remote Settings](https://remote-settings.readthedocs.io/en/latest/) collection. The Suggest component downloads these suggestions from Remote Settings, stores them in a local SQLite database, and makes them available to the Firefox address bar. Because these suggestions are stored and matched locally, Mozilla never sees the user's search queries.

This component follows the architecture of the other [Application Services Rust components](https://mozilla.github.io/application-services/book/index.html): a cross-platform Rust core, and platform-specific bindings for Firefox Desktop, Android, and iOS. These bindings are generated automatically using the [UniFFI](https://mozilla.github.io/uniffi-rs/) tool.

### For consumers

This section is for application developers. It describes how Firefox Desktop, Android, and iOS consume the Suggest Rust component.

The cornerstone of the component is the `SuggestStore` interface, which is the **store**. The store "ingests" (downloads and persists) suggestions from Remote Settings, and returns matching suggestions as the user types. This is the main interface that applications use to interact with the component.

While the store provides most of the functionality, the application has a few responsibilities:

**1. Create and manage a `SuggestStore` as a singleton.** Under the hood, the store holds multiple connections to the database: a read-write connection for ingestion, and a read-only connection for queries. The store uses the right connection for each operation, so applications shouldn't create multiple stores. The store also separates _cached data_ from _user data_. The application is responsible for specifying the correct platform-specific directories to persist this data. Cached data, like suggestions, are stored in the discardable caches or "local data" directory. User data, like which suggestions have been dismissed, are persisted in the storage or "profile" directory. The application specifies these directories, and creates the store, using the `SuggestStoreBuilder` interface.

**2. Periodically call the store's `ingest()` method to ingest new suggestions.** While the store takes care of efficiently downloading and persisting suggestions from Remote Settings, the application is still responsible for scheduling this work. This is because the Suggest component doesn't have access to the various platform-specific background work scheduling mechanisms, like `nsIUpdateTimerManager` on Desktop, `WorkManager` on Android, or `BGTaskScheduler` on iOS. These are three different APIs with different constraints, written in three different languages. Instead of trying to bind to these different mechanisms, the Suggest component leaves it up to the application to use the right one on each platform. Ingestion is network- and disk I/O-bound, and is done in the background.

**3. Use the `query()` and `interrupt()` methods to query for fresh suggestions as the user types.** The application passes the user's input, and additional options like which suggestion types to return, to `query()`. Querying the database is disk I/O-bound, so it's very important for each application to use its platform's facilities for asynchronous work to avoid blocking its main thread. For example, Firefox Android uses Kotlin coroutines, while iOS uses Swift actors. (The UniFFI bindings for Firefox Desktop take care of this automatically). Running `query()` off-main-thread also lets applications `interrupt()` those queries when the user changes their input. This avoids waiting for a query to return suggestions that are now stale.

### For contributors

This section is a primer for engineers contributing code to the Suggest Rust component.

`suggest.udl` describes the component's interface for foreign language consumers. UniFFI uses this file to generate the language bindings for each platform. If you're adding a new suggestion type, you'll want to update the declarations of `Suggestion` and `SuggestionProvider` in this file, as well as the definitions of those types in `suggestion.rs` and `provider.rs`, respectively.

`store.rs` contains the implementation of `SuggestStore` and most of the component's tests.

`schema.rs` manages the database schema. **Remember to bump the schema version and add a migration whenever you change the schema.**

`db.rs` interacts with the Suggest database. The `SuggestDao` type in this module is a ["data access object"](https://en.wikipedia.org/wiki/Data_access_object) (DAO) that contains all the SQL statements for querying and updating the SQLite database. By convention, `SuggestDao` methods that can write to the database take `&mut self`, and methods that only read take `&self`. The `SuggestDb::read()` and `SuggestDb::write()` methods take a closure that receives either `&SuggestDao` or `&mut SuggestDao`; this is how we enforce that all writes are done in a transaction. If you're curious to learn about how we use SQLite, diagnosing a slow query, or adding a new suggestion type, you'll almost certainly want to look at this module.

`rs.rs` defines all the Remote Settings [record](https://docs.kinto-storage.org/en/stable/concepts.html#buckets-collections-and-records) and [attachment](https://docs.kinto-storage.org/en/stable/faq.html#can-i-store-files-inside-kinto) types. The records in the `quicksuggest` Remote Settings collection don't store the suggestions themselves. Instead, each record has a type, and a pointer to a JSON attachment that contains multiple suggestions of that type. This module defines [Serde](https://serde.rs/)-compatible types for these records and attachments.

`errors.rs` contains all the errors that this component returns. We use the crate-internal `Error` type for all fallible operations within the Rust component, and the public `SuggestApiError` type for errors that applications should handle.

There are other suggestion provider-specific Rust modules, like `yelp.rs`, `pocket.rs`, and `keyword.rs`, that aren't covered in this primer. If you're new to the component, we recommend starting with the highest-level interface in `store.rs` first, and jumping to the other types and modules as you encounter them in the code.

## Tests

We use a technique called ["snapshot testing"](https://notlaura.com/what-is-a-snapshot-test/) with the [`expect-test`](https://docs.rs/expect-test/latest/expect_test/) crate for the Suggest component's tests. This technique makes it easier to compare and update all the expected outputs when adding, removing, and changing suggestion types.

The snapshot tests in `store.rs` look like this:

```rs
expect![["{expected-output}"]].assert_debug_eq(&actual_output);
```

The `expect-test` crate generates the `{expected-output}`, and can update it automatically. If you add, remove, or change a suggestion type, or update the schema, and run `cargo test`, you'll likely see a few failures. `expect-test` will print a readable diff in the `cargo test` output, which you can audit for accuracy.

If the diff looks good, you can update the expectations in-place using:

```shell
> env UPDATE_EXPECT=1 cargo test
```

Most of the tests in `store.rs` are integration-style tests that use a fake Remote Settings interface.
