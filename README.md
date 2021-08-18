# Assignment Overview

Create a phoenix server with a graphQL API that allows for interactions with a user schema that has an association called preferences. The server [should connect to a postgres DB](https://github.com/breachintelligence/elixir_interview_assignment). When completed we should be able to start the server with these commands:

* mix deps.get
* mix ecto.create
* mix ecto.migrate
* iex -S mix phx.server

We should also be able to:
* test the queries and mutations you implemented with graphiql @ `http://localhost:4000/graphiql`.
* run tests with `mix test`


## Schemas
### Users
  * name :: text
  * email :: text

Users have one preference.

### Preferences
  * likes_emails :: boolean, default should default to false and be non nullable.
  * likes_phone_calls :: boolean, default should default to false and be non nullable.

Preferences should have a relationship with the users table.

## Queries.
### All Users Query:

```graphql
query allUsers(
    $after: Int,
    $before: Int,
    $first: Int,
    $likesEmails: Boolean,
    $likesPhoneCalls: Boolean,
    $name: String
  ) {
  users(
    after: $after,
    before: $before,
    first: $first,
    likesEmails: $likesEmails,
    likesPhoneCalls: $likesPhoneCalls,
    name: $name
  ) {
    name
    email
    id
    preferences {
      likesEmails
      likesPhoneCalls
    }
  }
}
```

### Find User By Id
```graphql
query findById($id: ID){
  user(id: $id){
    name
    email
    id
    preferences{
      likesEmails
      likesPhoneCalls
    }
  }
}
```

### Resolver hits
Your server should track each time a graphQL resolver is invoked and maintain a count that can be queried by resolver name. For example, your
`find/2` resolver for users might look like this:
```elixir
@spec find(map(), any) :: User.t_res()
def find(%{id: id}, _) do
  ActivityMonitor.update_resolver_activity("find")
  Accounts.find_user(%{id: id})
end
```

Then fetching resolver activity might look like this:
```elixir
@spec find(%{key: String.t}, any) :: {:ok, integer}
def find(%{key: key}, _) do
  with :error <- ActivityMonitor.fetch_resolver_activity(key) do
    {:error, "Requested key: #{key} is invalid"}
  end
end
```

A test module for this resolver should look similar to this:

```elixir
defmodule ServerDemoWeb.Schema.Queries.ActivityMonitorTest do
  use ServerDemoWeb.Support.DataCase, async: true

  alias ServerDemoWeb.Schema

  @resolver_hits"""
    query getResolverHits($key: String){
      resolverHits(key: $key)
    }
  """

  describe "@resolverHits" do
    test "returns the number of times a resolver has been used" do
      assert {:ok, %{data: %{"resolverHits" => count}}} = Absinthe.run(
        @resolver_hits,
        Schema,
        variables: %{"key" => "find"}
      )
      assert count === 0
    end

    test "returns an error when an invalid resolver is requested" do
      assert {:ok, %{data: %{"resolverHits" => count}, errors: errors}} = Absinthe.run(
        @resolver_hits,
        Schema,
        variables: %{"key" => "this_does_not_exist"}
      )
      assert is_nil(count) === true
      assert List.first(errors).message === "Requested key: this_does_not_exist is invalid"
    end
  end
end
```

```graphql
query getResolverHits($key: String){
  resolverHits(key: $key)
}
```


## Mutations

### create a user

```graphql
mutation createUser(
  $name: String,
  $email: String,
  $likesEmails: Boolean,
  $likesPhoneCalls: Boolean
) {
  createUser(
    name: $name,
    email: $email,
    preferences: {
      likesEmails: $likesEmails,
      likesPhoneCalls: $likesPhoneCalls
      }
  ) {
    id
    name
    email
    preferences {
      likesEmails
      likesPhoneCalls
    }
  }
}
```

### update a user

```graphql
mutation updateUser(
  $id: ID,
  $name: String
){
  updateUser(
    id: $id,
    name: $name,
  ){
    id
    name
  }
}
```

### update user preferences
```graphql
mutation updateUserPreferences(
  $userId: ID,
  $likesEmails: Boolean,
  $likesPhoneCalls: Boolean
){
  updateUserPreferences(
    userId: $userId,
    likesEmails: $likesEmails,
    likesPhoneCalls: $likesPhoneCalls
  ) {
    id
    likesEmails
    likesPhoneCalls
  }
}
```


## Subscriptions -- challenge round --


### emits when a new user is created.
```graphql
subscription createdUser {
  createdUser {
    id
    name
    email
  }
}
```

## emits when user preferences are updated.
```graphql
 subscription updatedUserPreferences($userId: ID){
  updatedUserPreferences(userId: $userId) {
    id
    likesEmails
  }
}
```

## Resources:

* [Postgres](https://hub.docker.com/_/postgres)
* [ecto](https://hex.pm/packages/ecto)
* [phoenix](https://hex.pm/packages/phoenix)
  * [mix phx.new](https://hexdocs.pm/phoenix/Mix.Tasks.Phx.New.html#content)
  * [mix phx.gen.schema](https://hexdocs.pm/phoenix/Mix.Tasks.Phx.Gen.Schema.html)
* [absinthe](https://hex.pm/packages/absinthe)
  * [graphiql](https://hexdocs.pm/absinthe/introspection.html#using-graphiql)
* [ecto_shorts](https://hex.pm/packages/ecto_shorts)

